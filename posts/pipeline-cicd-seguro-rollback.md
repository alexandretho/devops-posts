---
title: "Pipeline CI/CD seguro: build, teste, rollback e deploy sem susto"
category: "CI/CD / GitHub Actions"
source_issue: 233
---

# Pipeline CI/CD seguro: build, teste, rollback e deploy sem susto

Pipeline boa não é a que faz deploy mais rápido. É a que entrega mudança rastreável, testada e com caminho claro de rollback quando algo dá errado.

Automatizar deploy sem pensar em segurança operacional só troca um erro manual por um erro automático, mais rápido e geralmente mais difícil de interromper.

## Separe build de deploy

Um fluxo saudável começa gerando um artefato imutável. Para aplicação em container, isso geralmente significa imagem Docker versionada pelo SHA do commit.

Exemplo no GitHub Actions:

```yaml
name: build

on:
  push:
    branches: [main]

jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - name: Login GHCR
        run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u "${{ github.actor }}" --password-stdin

      - name: Build image
        run: docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .

      - name: Push image
        run: docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
```

Com isso, cada deploy aponta para uma versão exata.

## Use aprovação para produção

Ambiente de produção não deveria ser só mais um job automático sem controle.

No GitHub Actions, use `environment` com reviewers:

```yaml
deploy-prod:
  needs: docker
  runs-on: ubuntu-latest
  environment: production
  steps:
    - name: Deploy
      run: ./scripts/deploy.sh ghcr.io/ORG/REPO:${{ github.sha }}
```

Aprovação manual não é burocracia quando protege ambiente crítico.

## Rollback precisa ser simples

Se a imagem é versionada por SHA, rollback vira escolher o SHA anterior conhecido como estável.

Exemplo conceitual:

```bash
kubectl set image deployment/minha-app \
  app=ghcr.io/org/repo:SHA_ANTERIOR \
  -n producao

kubectl rollout status deployment/minha-app -n producao
```

E, se precisar desfazer o último rollout:

```bash
kubectl rollout undo deployment/minha-app -n producao
```

## Secrets: menos é mais

Evite secrets long-lived quando puder. Prefira permissões mínimas e tokens específicos por ambiente.

Checklist básico:

- não imprimir secrets em logs;
- separar secrets de staging e produção;
- revisar quem pode aprovar deploy;
- evitar token pessoal em pipeline;
- preferir OIDC quando o provedor suporta.

## Checklist da pipeline

- [ ] Build gera artefato imutável?
- [ ] Testes rodam antes do deploy?
- [ ] Produção exige aprovação?
- [ ] Deploy usa tag por SHA?
- [ ] Existe comando claro de rollback?
- [ ] Secrets têm escopo mínimo?
- [ ] Logs mostram versão implantada?

## Conclusão

CI/CD bom não é só velocidade. É previsibilidade. Quando cada deploy tem artefato rastreável, aprovação adequada e rollback simples, a automação deixa de ser risco e passa a ser controle operacional.
