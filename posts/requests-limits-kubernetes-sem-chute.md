---
title: "Como dimensionar requests e limits no Kubernetes sem chutar valores"
category: "Kubernetes / OKD"
source_issue: 239
---

# Como dimensionar requests e limits no Kubernetes sem chutar valores

Um dos jeitos mais rápidos de desperdiçar dinheiro em Kubernetes é definir `requests` altos demais “por segurança”. O outro é colocar valores baixos demais e descobrir, em produção, que o pod vive sendo throttled ou morto por falta de memória.

`requests` e `limits` não deveriam ser chute. Eles são uma decisão operacional baseada em consumo real, comportamento da aplicação e tolerância a risco.

## Requests não são limits

Pense assim:

- `requests`: o mínimo que o scheduler reserva para o pod;
- `limits`: o teto que o container não deve ultrapassar.

Exemplo:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"
  limits:
    cpu: "1000m"
    memory: "1Gi"
```

Se você configura `requests` muito altos, o cluster parece cheio mesmo com baixa utilização real. Se configura `limits` de CPU muito apertados, pode provocar throttling e latência.

## Comece olhando consumo real

Antes de editar manifesto, colete dados:

```bash
kubectl top pods -n producao
kubectl top pod minha-app-abc123 -n producao --containers
kubectl describe pod minha-app-abc123 -n producao
```

Se você usa Prometheus, olhe percentis, não só média. Média esconde pico.

Consultas úteis como referência:

```promql
quantile_over_time(0.95, container_memory_working_set_bytes[7d])
rate(container_cpu_usage_seconds_total[5m])
container_cpu_cfs_throttled_seconds_total
```

## Uma regra prática inicial

Para aplicações web comuns:

- use p95 ou p99 como base para memória;
- deixe margem para pico controlado;
- seja mais cuidadoso com `limit` de CPU;
- não copie valor de outro serviço sem medir.

Um ponto de partida conservador:

```yaml
resources:
  requests:
    cpu: "200m"
    memory: "384Mi"
  limits:
    memory: "768Mi"
```

Nem todo serviço precisa de `limit` de CPU. Em muitos ambientes, limitar CPU agressivamente piora latência por throttling.

## Sinais de configuração ruim

Procure por eventos:

```bash
kubectl get events -n producao --sort-by=.lastTimestamp
kubectl describe pod minha-app-abc123 -n producao
```

Sinais clássicos:

- `OOMKilled`: memória insuficiente;
- `CPUThrottlingHigh`: CPU limitada demais;
- pods pendentes: requests maiores que capacidade disponível;
- cluster caro e ocioso: requests superdimensionados.

## Checklist de revisão

- [ ] O request foi baseado em métrica real?
- [ ] Existe diferença clara entre request e limit?
- [ ] O serviço tem histórico de pico?
- [ ] O HPA depende de CPU? Então request de CPU está definido?
- [ ] Há evidência de throttling?
- [ ] Há pods OOMKilled?

## Conclusão

Dimensionar recursos no Kubernetes é um processo contínuo. Comece com dados, ajuste aos poucos e revise depois de incidentes ou mudanças de tráfego. O objetivo não é deixar tudo no mínimo: é reservar o suficiente para estabilidade sem pagar por folga invisível.
