# Deployman — DevOps prático para ambientes reais

Site estático publicado em **https://deployman.com.br** via GitHub Pages.

## Novo fluxo editorial

O Deployman deixou de depender de posts gerados automaticamente por Issues. O fluxo agora é:

1. Issues antigas servem como base de pesquisa e backlog editorial.
2. Um tema é escolhido e reescrito como artigo técnico completo.
3. O artigo é versionado em Markdown dentro de `posts/`.
4. `posts/manifest.json` controla o que aparece na home.
5. A home carrega os posts locais e abre o artigo no próprio site.

A prioridade é qualidade e curadoria, não volume automático.

## Estrutura

```text
index.html                  # Home e leitor de artigos
posts/manifest.json         # Catálogo dos posts publicados
posts/*.md                  # Artigos curados em Markdown
.github/workflows/*.disabled # Fluxos antigos desativados
```

## Como publicar um novo post

1. Crie um arquivo Markdown em `posts/slug-do-post.md`.
2. Adicione o post em `posts/manifest.json`.
3. Faça commit e abra PR.
4. Após merge na `main`, o GitHub Pages publica automaticamente.

## Convenção de post

Use uma estrutura prática:

```markdown
# Título claro

## O problema
## Diagnóstico
## Como resolver
## Armadilhas comuns
## Checklist
## Conclusão
```

Evite tom genérico de IA. Prefira contexto real, comandos úteis, trade-offs e checklist operacional.
