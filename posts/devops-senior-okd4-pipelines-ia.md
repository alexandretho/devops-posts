---
title: "DevOps Sênior em ambiente OKD4: atualização de cluster, pipelines e automação com IA"
category: "DevOps / SRE / Operação"
---

# DevOps Sênior em ambiente OKD4: atualização de cluster, pipelines e automação com IA

Operar Kubernetes em produção é uma coisa. Operar OKD4 — a distribuição enterprise do Red Hat baseada em OpenShift — em ambiente financeiro regulado com SLA real é outra conversa. Este post descreve o tipo de trabalho que faço como DevOps Sênior: não o manual, mas o raciocínio operacional por trás das decisões.

## O ambiente

OKD4 com múltiplos clusters, workloads críticos, pipelines de CI/CD integradas ao ciclo de entrega de produto, observabilidade com Dynatrace e stack ELK, e integração crescente com AWS — incluindo serviços gerenciados como Bedrock para IA.

O perfil do trabalho não é só "manter rodando". É evoluir a plataforma enquanto o negócio continua operando, garantir que upgrades não virem incidentes e que cada mudança seja rastreável.

## Atualização de cluster OKD4: onde a maioria subestima o risco

Atualizar OKD4 em produção não é `kubectl apply` de manifesto. É uma operação com blast radius real se mal conduzida.

**O que valido antes de qualquer upgrade:**

```bash
# Estado geral dos nodes
oc get nodes
oc get nodes -o custom-columns='NODE:.metadata.name,STATUS:.status.conditions[-1].type,VERSION:.status.nodeInfo.kubeletVersion'

# Verificar se todos os operadores estão Available
oc get clusteroperators

# Verificar upgrade path — OKD/OpenShift só permite saltos de minor version específicos
oc adm upgrade

# MachineConfigPool — precisa estar UPDATED antes de prosseguir
oc get mcp
```

O ponto crítico que quase todo upgrade ignora: **MachineConfigPool deve estar totalmente em `UPDATED=True` antes de iniciar o próximo salto**. Deixar nodes com configuração mista é pedir inconsistência.

**O que monitoro durante o upgrade:**

- `clusteroperators` com `PROGRESSING=True` por tempo acima do esperado é sinal de travamento, não de progresso
- Pods no namespace `openshift-*` que ficam em `Pending` ou `CrashLoopBackOff` indicam recurso insuficiente ou incompatibilidade de versão
- Eventos no namespace `openshift-cluster-version` mostram o histórico real do que o operator de upgrade está fazendo

## Pipelines: build rastreável ou não é CI/CD

A maior fraqueza que vejo em pipelines corporativas é a rastreabilidade quebrada entre o que foi buildado e o que rodou em produção.

**O padrão que sigo:**

```yaml
# Imagem tagueada por SHA do commit, não por "latest"
image: registry.example.com/app:${GIT_SHA}

# Build com context limpo
docker build \
  --no-cache \
  --label "git.sha=${GIT_SHA}" \
  --label "build.date=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  -t registry.example.com/app:${GIT_SHA} .
```

O deploy só acontece quando:
1. Build passou
2. Testes automatizados passaram (unitários + integração)
3. Scan de vulnerabilidade não bloqueou
4. Aprovação manual (para produção) ou automática (para staging)

Rollback é `oc rollout undo`, mas **só funciona se o histórico estiver íntegro** — o que depende de `revisionHistoryLimit` configurado corretamente no Deployment.

## Observabilidade: Dynatrace e ELK na prática

Ter Dynatrace e ELK é uma vantagem enorme. Desperdiçar essa vantagem com dashboards bonitos e sem alertas úteis é um problema que vejo em quase todo ambiente.

**O que configuro como primeira coisa em qualquer ambiente:**

No Dynatrace:
- Alertas de aumento de error rate acima de baseline (não threshold fixo — baseline dinâmico)
- SLO configurado para latência do percentil 95, não média
- Problemas agrupados por serviço, não por métrica individual

No ELK:
- Índices com ILM (Index Lifecycle Management) configurado — sem ILM você acumula dados infinitamente ou perde logs por falta de espaço
- Alertas de log por padrão de erro crítico, não por volume de log

```
ECS (Elastic Common Schema) → Filebeat → Logstash → Elasticsearch → Kibana
                                                                    ↓
                                                              Alertmanager
```

A correlação entre trace (Dynatrace) e log (ELK) é o que transforma observabilidade em diagnóstico real. Quando um trace mostra latência anormal, o link para o log do mesmo request no mesmo timeframe fecha o ciclo de investigação em minutos.

## AWS e automação com Bedrock

A integração com AWS evoluiu de "só S3 para backup" para workloads de IA usando Bedrock.

Bedrock é o serviço gerenciado da AWS para modelos de linguagem — Claude, Titan, Llama, entre outros — com vantagem de não exigir infraestrutura própria de GPU e estar dentro do perímetro de compliance da conta AWS.

**Casos de uso que opero:**

- Classificação de logs de erro com LLM via Bedrock: o pipeline captura logs críticos, passa para o modelo com contexto do sistema, e retorna categoria + prioridade sugerida
- Geração de resumo automático de incidentes: após resolução, o agente consolida timeline, impacto, causa raiz e ações tomadas em formato padronizado
- Análise de custo de recursos AWS: script que lê Cost Explorer e pede ao modelo uma análise de anomalias vs. tendência esperada

```python
import boto3

bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')

response = bedrock.invoke_model(
    modelId='anthropic.claude-3-sonnet-20240229-v1:0',
    body=json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1024,
        "messages": [
            {
                "role": "user",
                "content": f"Analise estes logs de erro e classifique por prioridade:\n\n{logs}"
            }
        ]
    })
)
```

## O que diferencia operação real de tutorial

Qualquer pessoa consegue seguir um tutorial do Kubernetes. O que não está no tutorial:

- **Decisão de quando NÃO fazer upgrade**: janela de negócio, freeze period, risco de dependência com versão específica de operator
- **Gestão de incidente com blast radius parcial**: parte do cluster degradada, decisão de isolar ou aceitar degradação controlada
- **Trade-off entre automação e controle**: automatizar o rollback pode resolver em segundos mas também pode mascarar causa raiz sistêmica
- **Comunicação de status durante incidente**: a mensagem para o time de negócio é tão importante quanto o fix técnico

Operação sênior não é saber mais comandos. É ter o julgamento para decidir quando executar, quando esperar e quando escalar.

---

*Este post descreve decisões e práticas técnicas de operação em ambiente enterprise. Detalhes de negócio e identificação de ambiente foram omitidos intencionalmente.*
