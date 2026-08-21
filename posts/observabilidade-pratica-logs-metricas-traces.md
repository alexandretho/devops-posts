---
title: "Observabilidade na prática: logs, métricas e traces sem dashboard bonito inútil"
category: "Observabilidade"
source_issue: 142
---

# Observabilidade na prática: logs, métricas e traces sem dashboard bonito inútil

Observabilidade não é ter vinte dashboards abertos no monitor. É conseguir responder rápido: o que quebrou, onde quebrou, quem foi afetado e qual é a próxima ação.

Quando a stack de observabilidade vira coleção de gráficos bonitos, ela ajuda pouco no incidente. O valor aparece quando logs, métricas e traces contam a mesma história.

## Comece pelas perguntas certas

Antes de criar dashboard, defina o que você precisa responder:

- a aplicação está disponível?
- o erro aumentou?
- a latência piorou?
- o problema está na aplicação, banco, rede ou dependência externa?
- quantos usuários foram afetados?

Essas perguntas guiam os sinais.

## Métricas: saúde e tendência

Métricas mostram comportamento ao longo do tempo. Para serviço web, comece com os quatro sinais clássicos:

- latência;
- tráfego;
- erros;
- saturação.

Exemplos PromQL:

```promql
rate(http_requests_total[5m])
rate(http_requests_total{status=~"5.."}[5m])
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
container_memory_working_set_bytes
```

Não alerte para tudo. Alerta bom é aquele que pede ação.

## Logs: contexto do evento

Log bom ajuda a explicar o que aconteceu. Log ruim só aumenta custo.

Prefira logs estruturados:

```json
{
  "level": "error",
  "service": "checkout-api",
  "request_id": "abc-123",
  "user_id": "42",
  "message": "payment provider timeout",
  "duration_ms": 3200
}
```

Com `request_id`, fica muito mais fácil cruzar log com trace e erro de usuário.

## Traces: caminho da requisição

Trace responde onde o tempo foi gasto. Em arquitetura com APIs, filas e banco, isso vale ouro.

Um trace útil mostra:

- serviço de entrada;
- chamadas internas;
- dependências externas;
- query lenta;
- erro e status code;
- duração por etapa.

Sem trace, muita análise vira chute educado.

## Dashboard operacional mínimo

Para uma aplicação crítica, eu começaria com:

- disponibilidade por endpoint;
- taxa de erro 5xx;
- p95/p99 de latência;
- consumo de CPU/memória;
- restarts de pods;
- saturação de banco/fila;
- top erros por mensagem.

## Checklist de observabilidade útil

- [ ] Cada alerta tem ação clara?
- [ ] Logs têm `request_id` ou correlação?
- [ ] Métricas mostram latência, erro, tráfego e saturação?
- [ ] Há trace para fluxo crítico?
- [ ] Dashboard ajuda no incidente ou só enfeita?
- [ ] Existe runbook linkado no alerta?

## Conclusão

Observabilidade boa reduz tempo de diagnóstico. O objetivo não é colecionar ferramenta, é diminuir incerteza durante problema real. Se um painel não ajuda alguém a decidir o próximo passo, ele provavelmente é decoração.
