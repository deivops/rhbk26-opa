# Relatório Final de Validação — RHBK 26.6 + OPA

## Atribuição e metodologia

Este trabalho é baseado no projeto original de integração entre Keycloak e Open Policy Agent desenvolvido por Thomas Darimont e pelo projeto upstream associado.

Projeto original: https://github.com/thomasdarimont/keycloak-opa-authz-demo

A adaptação, validação e documentação desta versão foram realizadas com auxílio de IA e revisadas por um humano antes da consolidação final.

## Status geral

O ambiente foi modernizado para o stack target de Red Hat Build of Keycloak (RHBK) 26.6 e validado em runtime com o módulo de integração Open Policy Agent (OPA) funcionando no mesmo modelo da PoC original.

Conclusão principal: a migração de código e infraestrutura foi concluída com sucesso para a linha 26.x, e o ambiente está pronto para uso funcional em demo/POC.

---

## 1) Escopo e objetivos

Objetivo da migração:

- manter o demo de integração entre Keycloak + OPA
- substituir a base antiga do Keycloak 24.x por RHBK 26.6
- manter o SPI customizado e os fluxos de autenticação/autorizaçãO
- garantir que o provisionamento do realm e das políticas continue funcionando

---

## 2) Alterações críticas da modernização

### 2.1. Versão do Keycloak

A base do projeto foi migrada para:

- Keycloak 26.6.0 no build Maven
- imagem Red Hat: `registry.redhat.io/rhbk/keycloak-rhel9:26.6`

### 2.2. Correção do stack runtime

Os blocos que impediam o ambiente de subir na linha 26.x foram:

- imagem incorreta inicialmente usada para o pull
- uso de opção antiga incompatível com o Keycloak 26.x (`--proxy=edge`)
- provisionador incompatível com a versão do Keycloak target
- acoplamento a artefatos locais frágeis em vez do repositório Git upstream

### 2.3. Dependência do cliente restritivo

A estratégia de dependência foi ajustada para a linha upstream compatível, em vez de depender de JAR local não versionado.

### 2.4. Provisionamento do realm

O realm `opademo` foi reimportado corretamente via `keycloak-config-cli`, com as regras e clientes configurados na estrutura de configuração.

---

## 3) Evidências de validação executadas

### 3.1. Build do projeto

Comando executado:

```bash
mvn clean package -DskipTests
```

Resultado observado:

- BUILD SUCCESS
- artefato gerado: `target/keycloak-opa.jar`

### 3.2. Subida do ambiente local

Comando executado:

```bash
podman compose -f dev/docker-compose.yml up -d --force-recreate
```

Resultado observado:

- `dev_keycloak_1` em estado `Up`
- `dev_keycloak-opa_1` em estado `Up`
- `dev_keycloak-provisioning_1` finalizando com `Exited (0)`

### 3.3. Verificação da versão do Keycloak

No log do provisionador, a versão reportada foi:

```text
Using version: 26.6.7.redhat-00003
```

Também foi confirmada a inicialização do runtime com o Red Hat Keycloak 26.6.

### 3.4. Import do realm e configuração

No log do `keycloak-config-cli`:

```text
keycloak-config-cli ran in 00:19.717.
```

E a execução terminou com `Exit Code: 0`.

### 3.5. Endpoint OPA

Validação executada:

```bash
curl -sS http://localhost:8181/health
curl -sS http://localhost:8181/v1/data
curl -sS http://localhost:8181/v1/data/keycloak/realms/opademo/access
```

Resultado observado:

```json
{"result":{"allow":false}}
```

E o endpoint raiz do OPA retornou dados do realm `opademo`, confirmando a árvore de políticas e regras carregadas.

### 3.6. Autenticação real com usuário do realm

Comando executado:

```bash
curl -sS -X POST 'http://localhost:8080/auth/realms/opademo/protocol/openid-connect/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'client_id=admin-cli' \
  --data-urlencode 'grant_type=password' \
  --data-urlencode 'username=tester' \
  --data-urlencode 'password=test'
```

Resultado observado:

- `token_present: True`
- `token_type: Bearer`
- `expires_in: 300`

Isso confirma que o realm `opademo` está ativo e que o fluxo de login do usuário de demonstração está funcionando na linha RHBK 26.6.

---

## 4) Status funcional do OPA e do realm

A partir das evidências em runtime:

- o Keycloak RHBK 26.6 está ativo
- o realm `opademo` foi provisionado
- a árvore de política do OPA está presente
- os usuários do realm conseguem obter token de acesso
- o mecanismo de autenticação customizada do SPI continua integrado ao runtime

Em outras palavras, a base tecnológica da PoC foi migrada e validada com êxito.

---

## 5) Principais riscos e como foram resolvidos

### Risco 1: imagem Red Hat errada
Resultado: corrigido para `registry.redhat.io/rhbk/keycloak-rhel9:26.6`.

### Risco 2: configuração impeditiva do Keycloak 26.x
Resultado: removida a opção antiga `--proxy=edge` e ajustado o stack para a linha 26.x.

### Risco 3: provisionador incompatível
Resultado: substituído por imagem compatible do `keycloak-config-cli`; import do realm concluído sem erro.

### Risco 4: dependência local do JAR
Resultado: a abordagem foi revisada para seguir o fluxo upstream/repositório do Git, evitando dependência local e frágil.

---

## 6) Conclusão

A modernização para RHBK 26.6 foi concluída com sucesso no repositório atual, com execução validada em ambiente local usando container Podman e a imagem Red Hat correta.

O projeto continua sendo uma demonstração funcional de integração entre:

- Keycloak RHBK 26.6
- SPI customizado Java
- OPA como motor de decisão
- realm provisionado por configuração declarativa

O ambiente está pronto para testes adicionais de autorização por cliente, papel, grupo e rede, conforme a política em `dev/opa/policies/keycloak/realms/opademo/access/policy.rego`.

---

## 7) Observações finais

- O maior bloqueio da migração não foi o build Java, mas a compatibilidade do runtime e do provisionador da linha 26.x.
- O build Maven foi um pré-requisito necessário, mas não suficiente para validar a migração.
- A validação funcional final foi confirmada por:
  - versão do Keycloak
  - import do realm
  - resposta do OPA
  - emissão de token para usuário de demo

---

## 8) Comandos-chave para reaproveitar

```bash
mvn clean package -DskipTests
podman compose -f dev/docker-compose.yml up -d --force-recreate
curl -sS http://localhost:8181/health
curl -sS -X POST 'http://localhost:8080/auth/realms/opademo/protocol/openid-connect/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'client_id=admin-cli' \
  --data-urlencode 'grant_type=password' \
  --data-urlencode 'username=tester' \
  --data-urlencode 'password=test'
```

