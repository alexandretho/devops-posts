---
title: "Hermes: o agente de IA que opera minha infraestrutura enquanto eu durmo"
category: "IA Aplicada / Automação"
---

# Hermes: o agente de IA que opera minha infraestrutura enquanto eu durmo

Tem uma linha que separa usar ChatGPT para tirar dúvidas e ter um agente de IA que realmente executa coisas no seu ambiente. Eu cruzei essa linha há alguns meses e o que está do outro lado é diferente do que eu esperava.

## O que é o Hermes

Hermes é um agente de IA que roda localmente no meu servidor, conectado ao Claude (Anthropic), com acesso real a ferramentas: terminal, browser, GitHub, Docker, arquivos, APIs externas, banco de dados, agenda de tarefas. Ele não é um chatbot. Ele executa.

Quando eu mando uma mensagem do Telegram — pode ser texto, áudio, imagem, arquivo — ele processa, decide qual conjunto de ferramentas usar, executa e me devolve um resultado real. Não uma sugestão. O resultado.

## O que ele faz de concreto

Alguns exemplos do dia a dia atual:

**Infraestrutura e DevOps:**
- Verifica o status de todos os containers Docker, serviços healthcheck, portas abertas
- Faz restart seletivo quando detecta serviço degradado
- Cria e gerencia cron jobs com lógica de negócio complexa
- Comita e faz push em repositórios GitHub após executar tarefas
- Lê logs, identifica padrão de erro e sugere ou aplica correção

**Desenvolvimento:**
- Lê uma issue, cria branch, escreve código, testa, abre PR
- Analisa PRs com diff real e comenta inline no GitHub
- Executa ciclos TDD (RED → GREEN → REFACTOR) sozinho, reportando resultado de cada teste

**Automação operacional:**
- Monitora preços, portais de imóveis, portais de licitação
- Dispara alertas personalizados quando condições são atendidas
- Mantém dashboards e bancos de dados atualizados em segundo plano
- Processa arquivos enviados por WhatsApp e atualiza sistemas internos

**Contexto persistente:**
- Lembra do que conversamos semanas atrás
- Mantém memória sobre projetos, preferências e contexto de cada sistema
- Busca em histórico de conversas para não pedir a mesma informação duas vezes

## Como isso funciona por baixo

A arquitetura central é simples: Hermes recebe mensagem via Telegram, passa para um loop de raciocínio com Claude como modelo, o modelo decide quais ferramentas chamar, as ferramentas são executadas no servidor real, o resultado volta para o modelo, e o ciclo repete até a tarefa estar concluída.

O que torna isso diferente de um script de automação comum é que o modelo decide a sequência de ações com base no contexto — ele lê o estado atual antes de agir, adapta o plano se algo falha, e encadeia operações que dependem umas das outras sem precisar de um workflow pré-programado.

```
mensagem Telegram
  → Hermes (orquestrador)
    → Claude (raciocínio)
      → ferramentas: terminal / GitHub / Docker / browser / arquivos / banco
        → resultado real
          → resposta no Telegram
```

## O que aprendi operando isso

**Prompts são contratos.** Quando o agente vai executar algo com efeito colateral real — push no GitHub, restart de container, envio de e-mail — a instrução precisa ser tão precisa quanto uma especificação técnica. Ambiguidade vira bug.

**Ferramentas são o diferencial.** O modelo em si é commodity. O que muda é o conjunto de capacidades expostas e a qualidade dos contratos entre o modelo e as ferramentas. Um agente com 20 ferramentas mal desenhadas é pior que um com 5 bem definidas.

**Verificação antes de comemorar.** O agente pode dizer "arquivo criado" e ter criado com encoding errado. "Commit feito" não significa que o push funcionou. Aprendi a instruí-lo a verificar o estado após cada ação antes de declarar sucesso.

**Memória e contexto mudam tudo.** Sem memória persistente, cada conversa começa do zero e o agente vira uma LLM glorificada. Com memória indexada e recuperação semântica, ele começa a parecer alguém que conhece o projeto de verdade.

## O que ainda não funciona bem

Honestidade técnica importa. Algumas coisas ainda são frágeis:

- **Tarefas longas sem supervisão**: em fluxos com muitos passos, pode derivar do objetivo se uma ferramenta retorna resultado inesperado
- **Ambiguidade de escopo**: "atualiza o sistema" sem especificação pode virar algo maior do que o pretendido
- **Custo de tokens**: sessões pesadas consomem significativamente. Ainda estou ajustando quais tarefas valem execução autônoma vs. assistida

## Por que isso importa para DevOps

A maior parte do trabalho operacional é repetição com variação pequena: verificar estado, aplicar mudança, validar resultado, reportar. Essa estrutura é exatamente o que agentes de IA executam bem.

Não estou falando de substituir engenheiro. Estou falando de ter um agente que executa os runbooks enquanto você dorme, te acorda se algo fugiu do esperado, e já te entrega o diagnóstico inicial quando você abre o olho.

O modelo de operação muda. Não é mais "eu executo". É "eu defino o que deve acontecer, valido o resultado, e ajusto quando necessário".

---

*Este post faz parte de uma série sobre automação e IA aplicada à operação real. Nenhum ambiente foi prejudicado na produção deste conteúdo — pelo menos não de forma permanente.*
