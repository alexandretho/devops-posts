---
title: "HPA no Kubernetes: por que seu pod não escala mesmo com CPU alta?"
category: "Kubernetes / OKD"
source_issue: 234
---

# HPA no Kubernetes: por que seu pod não escala mesmo com CPU alta?

Você configurou o HorizontalPodAutoscaler, gerou carga na aplicação, viu a CPU subir no dashboard e mesmo assim o número de réplicas ficou parado. Esse é um daqueles problemas em que a primeira reação costuma ser culpar o HPA, mas quase sempre o erro está em alguma dependência ao redor dele.

O HPA não escala por instinto. Ele depende de métricas disponíveis, `requests` configurados, alvo correto e tempo suficiente para tomar decisão. Se uma dessas peças falha, ele simplesmente não tem base para agir.

## O que o HPA precisa para funcionar

Para escalar por CPU ou memória, o HPA normalmente precisa de três coisas:

- Metrics Server saudável;
- pods com `resources.requests` definidos;
- métrica aparecendo corretamente no `describe` do HPA.

Comece pelo básico:

```bash
kubectl get hpa -A
kubectl describe hpa nome-do-hpa -n namespace
kubectl top pods -n namespace
kubectl top nodes
```

Se `kubectl top` não funciona, o problema não está no HPA. Está na coleta de métricas.

## Diagnóstico rápido

Olhe primeiro a saída do `describe`:

```bash
kubectl describe hpa app-hpa -n producao
```

Procure mensagens como:

```text
failed to get cpu utilization: missing request for cpu
unable to get metrics for resource cpu
current CPU utilization: <unknown>
```

Essas mensagens são ouro. Elas apontam diretamente para a causa provável.

## Causa comum: container sem request de CPU

Para calcular percentual de CPU, o Kubernetes compara o uso atual com o `request` definido. Sem `request`, ele não sabe o que significa “70% de CPU”.

Exemplo mínimo correto:

```yaml
resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

Depois de ajustar, aplique o manifesto e acompanhe:

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/minha-app -n producao
kubectl describe hpa app-hpa -n producao
```

## Causa comum: Metrics Server indisponível

Verifique se a API de métricas responde:

```bash
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes | head
```

Se isso falhar, revise o Metrics Server antes de mexer no HPA.

## Causa comum: alvo errado

Parece bobo, mas acontece: HPA apontando para Deployment antigo, nome errado ou namespace errado.

```bash
kubectl get hpa app-hpa -n producao -o yaml
kubectl get deployment -n producao
```

Confira `scaleTargetRef`:

```yaml
scaleTargetRef:
  apiVersion: apps/v1
  kind: Deployment
  name: minha-app
```

## Checklist de troubleshooting

- [ ] `kubectl top pods` funciona?
- [ ] O container tem `resources.requests.cpu`?
- [ ] O HPA aponta para o Deployment certo?
- [ ] O `describe hpa` mostra métrica atual ou `<unknown>`?
- [ ] O limite mínimo/máximo de réplicas permite escalar?
- [ ] A carga dura tempo suficiente para o HPA reagir?

## Conclusão

Quando o HPA não escala, evite sair aumentando réplica manualmente sem entender a causa. Primeiro valide métricas, requests e alvo. Na prática, um `kubectl describe hpa` bem lido economiza muito tempo e evita mexer no lugar errado.
