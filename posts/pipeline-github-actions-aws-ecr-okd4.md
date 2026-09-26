---
title: "Pipeline no GitHub Actions com AWS: testes, build, ECR e deploy em OKD4"
category: "CI/CD / GitHub Actions / AWS"
---

# Pipeline no GitHub Actions com AWS: testes, build, ECR e deploy em OKD4

Muito pipeline que vejo em produção faz duas coisas: builda e deploya. O problema é que entre o commit e o deploy, uma série de verificações precisam acontecer — e quando elas não acontecem na pipeline, elas acontecem em produção.

Este post descreve como estruturo pipelines reais usando GitHub Actions integrado com AWS CodeBuild, ECR, testes automatizados e deploy em OKD4.

## A estrutura de uma pipeline que vale alguma coisa

A ordem importa. Cada estágio deve falhar rápido e barato antes que o problema chegue em um estágio mais caro ou mais arriscado.

```
commit → lint/testes unitários → build de imagem → scan de vulnerabilidade
       → testes de integração → push ECR → aprovação (prod) → deploy OKD4
```

A regra prática: o que é mais rápido e mais barato de rodar fica primeiro. Teste unitário falha em segundos. Build de imagem leva minutos. Deploy em produção tem custo de reversão.

## Repositório e controle de branch

O gatilho da pipeline parte do GitHub. A estratégia de branching define o fluxo:

- `main` → deploy em **produção** (com aprovação manual)
- `develop` → deploy em **staging** (automático)
- `feature/*` → apenas testes + build (sem deploy)

```yaml
# .github/workflows/pipeline.yml
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
```

Pull requests para `main` precisam passar por testes antes de serem mergeados. Isso não é opcional — é branch protection configurado no repositório.

```bash
# Configurar proteção via gh CLI
gh api repos/org/repo/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["test","build"]}' \
  --field enforce_admins=false \
  --field required_pull_request_reviews='{"required_approving_review_count":1}' \
  --field restrictions=null
```

## Estágio 1: testes automatizados no GitHub Actions

O primeiro job roda testes — sem build, sem AWS, sem custo extra. Se falhar aqui, parou.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Instalar dependências
        run: npm ci

      - name: Testes unitários
        run: npm run test:unit

      - name: Testes de integração
        run: npm run test:integration
        env:
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}

      - name: Cobertura mínima
        run: npm run test:coverage -- --coverageThreshold='{"global":{"lines":80}}'
```

Cobertura mínima não é métrica de vaidade — é um contrato. Se alguém apaga um teste sem perceber, a pipeline avisa antes do merge.

## Estágio 2: build da imagem com AWS CodeBuild

O build da imagem roda no CodeBuild — não no runner do GitHub Actions. Motivos:

- **Custo**: instâncias grandes no CodeBuild são mais baratas para builds pesados que runners pagos do GitHub
- **Cache de layer**: o CodeBuild mantém cache local entre builds no mesmo projeto
- **Isolamento de rede**: o CodeBuild pode rodar dentro de uma VPC, com acesso a recursos internos sem expor portas públicas
- **Rastreabilidade**: cada build tem ARN próprio, logs no CloudWatch, duração, status

O GitHub Actions dispara o CodeBuild via AWS SDK:

```yaml
  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      id-token: write   # necessário para OIDC
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Configurar credenciais AWS via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/github-actions-deploy
          aws-region: us-east-1

      - name: Login no ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build e push para ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build \
            --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
            --build-arg GIT_SHA=${{ github.sha }} \
            --label "org.opencontainers.image.revision=${{ github.sha }}" \
            -t $ECR_REGISTRY/${{ vars.ECR_REPO }}:$IMAGE_TAG \
            -t $ECR_REGISTRY/${{ vars.ECR_REPO }}:latest \
            .
          docker push $ECR_REGISTRY/${{ vars.ECR_REPO }}:$IMAGE_TAG
          docker push $ECR_REGISTRY/${{ vars.ECR_REPO }}:latest
          echo "image=$ECR_REGISTRY/${{ vars.ECR_REPO }}:$IMAGE_TAG" >> $GITHUB_OUTPUT
```

**Por que OIDC e não access key?**
Access keys expiram, podem vazar, precisam de rotação manual. OIDC gera credencial temporária com escopo por repositório e branch — sem secret de longa duração.

## Estágio 3: scan de vulnerabilidade

Imagem buildada, scan antes de qualquer deploy:

```yaml
  scan:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - name: Scan com Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ needs.build.outputs.image }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'  # falha o job se encontrar CRITICAL ou HIGH

      - name: Upload para GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'
```

O resultado vai para a aba **Security** do repositório — não some em log, fica registrado com o commit.

## Estágio 4: deploy em OKD4

Com imagem no ECR e scan limpo, o deploy acontece. OKD4 usa `oc` — o CLI do OpenShift — que é compatível com `kubectl` mas com camadas extras de controle de acesso via ServiceAccount e SCC (Security Context Constraints).

```yaml
  deploy-staging:
    needs: [build, scan]
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: staging

    steps:
      - name: Instalar oc CLI
        run: |
          curl -Lo oc.tar.gz https://mirror.openshift.com/pub/openshift-v4/clients/oc/latest/linux/oc.tar.gz
          tar -xzf oc.tar.gz
          sudo mv oc /usr/local/bin/

      - name: Login no OKD4
        run: |
          oc login ${{ secrets.OKD_SERVER }} \
            --token=${{ secrets.OKD_TOKEN }} \
            --insecure-skip-tls-verify=false

      - name: Atualizar imagem no Deployment
        run: |
          oc set image deployment/app \
            app=${{ needs.build.outputs.image }} \
            -n ${{ vars.OKD_NAMESPACE }}

      - name: Aguardar rollout
        run: |
          oc rollout status deployment/app \
            -n ${{ vars.OKD_NAMESPACE }} \
            --timeout=5m

      - name: Verificar pods saudáveis
        run: |
          oc get pods -n ${{ vars.OKD_NAMESPACE }} \
            -l app=app \
            --field-selector=status.phase=Running
```

Para produção, o job `deploy-prod` tem `environment: production` — o que ativa a aprovação manual configurada no GitHub Environments antes de executar.

## O que não pode faltar na configuração

**Secrets no GitHub:**
- `OKD_TOKEN` — ServiceAccount token com permissão mínima (`edit` no namespace, não `cluster-admin`)
- `OKD_SERVER` — endpoint do API server do cluster

**No OKD, o ServiceAccount precisa de SCC adequado:**
```bash
oc create serviceaccount github-deploy -n meu-namespace
oc policy add-role-to-user edit \
  system:serviceaccount:meu-namespace:github-deploy \
  -n meu-namespace
oc sa create-token github-deploy -n meu-namespace
```

**ECR lifecycle policy** para não acumular imagens indefinidamente:
```json
{
  "rules": [{
    "rulePriority": 1,
    "description": "Manter últimas 20 imagens por SHA",
    "selection": {
      "tagStatus": "tagged",
      "countType": "imageCountMoreThan",
      "countNumber": 20
    },
    "action": { "type": "expire" }
  }]
}
```

## O que diferencia uma pipeline boa de uma ruim

Pipeline ruim: faz push de `latest` sem tag, não tem testes antes do build, segredos como variável de ambiente no workflow, deploy automático em produção sem aprovação.

Pipeline boa: tag por SHA (rastreável), testes antes de qualquer build, OIDC em vez de access key, scan antes de deploy, aprovação manual para produção, rollout com timeout e verificação de saúde ao final.

A diferença entre as duas não é o ferramental — é a disciplina de pensar cada etapa como uma porta que só abre se o anterior passou.

---

*Os exemplos usam GitHub Actions, AWS ECR e OKD4. Os princípios se aplicam a qualquer combinação de CI/CD + registry + cluster Kubernetes.*
