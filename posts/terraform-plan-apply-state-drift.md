---
title: "Terraform na prática: plan, apply, state e drift sem magia"
category: "Terraform / IaC"
source_issue: 221
---

# Terraform na prática: plan, apply, state e drift sem magia

Terraform não é só escrever meia dúzia de recursos e rodar `apply`. O que separa um uso experimental de um uso seguro em produção é entender três pontos: plano, estado e drift.

Quando esses conceitos são ignorados, o Terraform deixa de ser ferramenta de controle e vira uma forma mais rápida de quebrar infraestrutura.

## O papel do state

O `state` é a memória do Terraform. Ele guarda o vínculo entre o código e os recursos reais.

Em laboratório, state local resolve:

```bash
terraform init
terraform plan
terraform apply
```

Em ambiente compartilhado, state local é problema. Você precisa de backend remoto e lock.

Exemplo com S3 e DynamoDB:

```hcl
terraform {
  backend "s3" {
    bucket         = "empresa-terraform-state"
    key            = "prod/app/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

Sem lock, duas pessoas podem aplicar mudanças ao mesmo tempo. Em produção, isso é convite para incidente.

## Plan não é formalidade

O `plan` precisa ser lido. Não basta ver que o comando saiu com sucesso.

```bash
terraform fmt -check
terraform validate
terraform plan -out=tfplan
terraform show tfplan
```

Preste atenção em ações destrutivas:

```text
-/+ destroy and then create replacement
- destroy
```

Mudanças em nome, região, subnet ou tipo de recurso podem forçar recriação.

## Drift: quando a realidade saiu do código

Drift acontece quando alguém altera recurso fora do Terraform: console, CLI, automação paralela ou ferramenta de terceiro.

Para detectar:

```bash
terraform plan -detailed-exitcode
```

Códigos úteis:

```text
0 = sem mudanças
1 = erro
2 = há mudanças planejadas
```

Isso é ótimo para pipelines de verificação.

## Cuidados antes do apply

Um fluxo mínimo mais seguro:

```bash
terraform fmt -check
terraform validate
terraform plan -out=tfplan
# revisão humana aqui
terraform apply tfplan
```

Evite `terraform apply -auto-approve` em produção sem gates, logs e revisão.

## Checklist de produção

- [ ] State remoto configurado?
- [ ] Lock habilitado?
- [ ] `plan` revisado antes do `apply`?
- [ ] Variáveis sensíveis fora do repositório?
- [ ] Pipeline separa `plan` e `apply`?
- [ ] Há rotina para detectar drift?
- [ ] Módulos têm versionamento?

## Conclusão

Terraform é poderoso porque transforma infraestrutura em mudança revisável. Mas o ganho real vem da disciplina: state remoto, plano revisado, detecção de drift e aplicação controlada. Sem isso, é só automação acelerando erro humano.
