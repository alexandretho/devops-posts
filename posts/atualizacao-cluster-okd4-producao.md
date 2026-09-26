---
title: "Atualização de cluster OKD4 em produção: o que ninguém conta antes de você começar"
category: "Kubernetes / OKD"
---

# Atualização de cluster OKD4 em produção: o que ninguém conta antes de você começar

A documentação oficial do OpenShift explica como atualizar um cluster. O que ela não explica é o que fazer quando a atualização para no meio, quando um operador fica preso em `Progressing` há 40 minutos, ou quando um nó não consegue aplicar o MachineConfig novo depois de 3 tentativas.

Este post cobre a atualização de OKD4 como ela realmente acontece em ambiente enterprise — com as verificações antes, o acompanhamento durante e o diagnóstico quando algo não vai como esperado.

## O que faz uma atualização de OKD4 ser diferente de Kubernetes upstream

No Kubernetes vanilla, você atualiza control plane e nodes de forma relativamente direta. No OKD4/OpenShift, a atualização é orquestrada pelo **Cluster Version Operator (CVO)**, que coordena dezenas de outros operadores. O upgrade não é um comando que executa e termina — é uma sequência de reconciliações que pode levar horas.

O CVO atualiza os operadores em ordem de dependência. Se um operador intermediário travar, toda a cadeia para. Entender isso muda como você monitora a operação.

## Antes de começar: checklist que não é opcional

**1. Verificar o caminho de atualização**

OKD4 não permite saltos arbitrários de versão. A rota é definida pelo grafo de atualização e precisa ser validada antes:

```bash
oc adm upgrade

# Saída esperada:
# Cluster version is 4.14.12
# Upstream is unset, so the cluster will use an appropriate default.
# Channel: stable-4.14 (available channels: candidate-4.14, ...)
#
# Recommended updates:
#   VERSION     IMAGE
#   4.14.15     quay.io/openshift-release-dev/ocp-release@sha256:...
```

Se a versão alvo não aparecer em `Recommended updates`, você não pode pular direto para ela. Precisa ir pelas intermediárias.

**2. Estado de saúde dos operadores**

Todos os `clusteroperators` precisam estar `Available=True`, `Progressing=False`, `Degraded=False` antes de iniciar:

```bash
oc get clusteroperators

# Procurar qualquer linha com DEGRADED=True ou AVAILABLE=False
oc get co | awk 'NR==1 || $3=="False" || $4=="True" || $5=="True"'
```

Um operador degradado antes do upgrade vai agravar durante. Resolva primeiro.

**3. MachineConfigPool estável**

```bash
oc get mcp

# NAME     CONFIG                      UPDATED   UPDATING   DEGRADED
# master   rendered-master-abc123      True      False      False
# worker   rendered-worker-xyz789      True      False      False
```

`UPDATING=True` significa que há mudança de MachineConfig pendente nos nós. Não inicie upgrade em cima disso.

**4. Nodes sem pressão de recurso**

```bash
oc get nodes
oc describe nodes | grep -A5 "Conditions:"

# Verificar se algum node está com:
# MemoryPressure, DiskPressure, PIDPressure = True
```

**5. etcd saudável**

O etcd é o estado do cluster. Antes de qualquer upgrade:

```bash
oc get pods -n openshift-etcd | grep etcd
# Todos devem estar Running

# Verificar saúde dos membros
oc rsh -n openshift-etcd etcd-<master-node> -- etcdctl member list --write-out=table
oc rsh -n openshift-etcd etcd-<master-node> -- etcdctl endpoint health --cluster --write-out=table
```

Se um membro do etcd estiver sem quorum, o upgrade vai falhar de forma não determinística. Corrija antes.

**6. Backup de etcd**

Não é opcional:

```bash
# Em cada nó master
oc debug node/<master-node> -- chroot /host /usr/local/bin/cluster-backup.sh /home/core/assets/backup
```

## Iniciando o upgrade

Com tudo verde, a atualização:

```bash
# Para versão específica
oc adm upgrade --to=4.14.15

# Para a versão recomendada mais recente no canal atual
oc adm upgrade --to-latest=true
```

A partir daqui, o CVO assume. O progresso aparece em:

```bash
# Visão resumida
oc get clusterversion

# Visão detalhada com histórico
oc describe clusterversion version
```

## Monitoramento durante o upgrade

Não abra e feche o terminal a cada 5 minutos. Crie um loop de observação:

```bash
# Acompanhar operadores em tempo real
watch -n30 'oc get co | grep -v "True.*False.*False"'
# Mostra apenas operadores fora do estado esperado

# Acompanhar MachineConfigPool
watch -n60 'oc get mcp'

# Eventos recentes no namespace do CVO
oc get events -n openshift-cluster-version --sort-by='.lastTimestamp' | tail -20
```

A ordem de atualização dos operadores segue dependências internas. Operadores críticos como `kube-apiserver`, `kube-controller-manager` e `kube-scheduler` atualizam primeiro e causam breve disrupção no plano de controle — conexões ao API server podem cair por alguns segundos. Isso é normal.

## Quando um operador trava em `Progressing`

O cenário mais comum de problema: um operador fica `Progressing=True` por muito mais tempo do que o esperado (referência: se passou de 30–40 min sem avançar, investigue).

**Diagnóstico:**

```bash
# Qual operador está travado
oc get co | grep "True.*True"

# Eventos do operador específico
oc get events -n openshift-<operador> --sort-by='.lastTimestamp'

# Logs do pod do operador
oc logs -n openshift-<operador> deployment/<operador> --tail=100

# Para kube-apiserver especificamente
oc logs -n openshift-kube-apiserver-operator \
  deployment/kube-apiserver-operator --tail=100 | grep -i error
```

**Causa comum: pod do novo operador não consegue subir**

```bash
oc get pods -n openshift-<operador>
oc describe pod <pod-travado> -n openshift-<operador>
# Procurar: ImagePullBackOff, OOMKilled, FailedScheduling
```

**Causa comum: node não aplica novo MachineConfig**

```bash
oc get mcp
# Se UPDATING=True por tempo longo:

# Ver qual nó está travado
oc get nodes

# Ver o MachineConfigDaemon naquele nó
oc logs -n openshift-machine-config-operator \
  $(oc get pods -n openshift-machine-config-operator \
    -l k8s-app=machine-config-daemon \
    --field-selector spec.nodeName=<nome-do-no> \
    -o name) --tail=100
```

## Atualização dos nós via MachineConfigPool

Depois que o control plane atualiza, os nós worker atualizam via MachineConfigPool. O processo drena o nó, aplica a nova configuração (que inclui nova versão do kubelet e cri-o) e reinicia.

Por padrão, o OKD4 atualiza um nó por vez no pool `worker`. Isso é controlado pelo `maxUnavailable`:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: worker
spec:
  maxUnavailable: 1  # padrão — altere para acelerar se o ambiente suportar
```

Em clusters grandes, `maxUnavailable: 2` ou `maxUnavailable: 20%` pode reduzir significativamente o tempo total — desde que os workloads tolerem perda de mais de um nó simultaneamente.

**Acompanhar o drain de nó:**

```bash
# Ver qual nó está sendo drenado
oc get nodes | grep SchedulingDisabled

# Acompanhar o progresso do drain
oc get events --field-selector reason=Evicting -A | tail -20
```

## Verificação pós-upgrade

Com todos os operadores de volta a `Available=True`, `Progressing=False`, `Degraded=False`:

```bash
# Versão atual do cluster
oc get clusterversion

# Todos os nós na nova versão
oc get nodes -o custom-columns='NODE:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion'

# Operadores estáveis
oc get co

# Validação de workloads críticos
oc get pods -A | grep -v "Running\|Completed" | grep -v "^NAMESPACE"
```

## O que mais impacta o tempo total

Em ambiente com muitos nós worker, o gargalo é o MachineConfigPool — cada nó precisa ser drenado, atualizado e reinicializado. Um cluster com 20 nós worker e `maxUnavailable: 1` pode levar 4–6 horas só nessa fase.

Planejar a janela de manutenção considerando esse tempo é parte do trabalho. Em ambientes críticos, comunicar o tempo esperado para as equipes de negócio antes de iniciar é parte da responsabilidade de quem conduz o upgrade.

---

*Este post descreve operações em OKD4 4.14.x. Detalhes podem variar entre versões minor. Sempre valide o caminho de atualização com `oc adm upgrade` antes de aplicar em produção.*
