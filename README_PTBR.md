# Keycloak Open Policy Integration Demo

Exemplo de integração entre Open Policy Agent e Keycloak, originalmente apresentado no Keycloak Dev Day 2024.

Este repositório foi adaptado e validado para Red Hat Build of Keycloak 26.6 em um ambiente de runtime local. O trabalho original e o projeto upstream continuam sendo creditados ao autor e mantenedores originais.

Projeto original: https://github.com/thomasdarimont/keycloak-opa-authz-demo

[Slides](keycloak-devday-2024-flexible-authz-for-keycloak-with-openpolicyagent.pdf)

## Atribuição e validação

Este fork é baseado no trabalho originalmente criado por Thomas Darimont e no projeto demo de Keycloak + OPA.

A migração, validação e atualização da documentação foram apoiadas com assistência de IA e revisadas/validadas por um operador humano antes de serem mantidas neste repositório.

## Status

Este repositório foi validado com Red Hat Build of Keycloak (RHBK) 26.6 e a stack de runtime foi confirmada como funcional neste ambiente.

A stack validada foi:

- Imagem do Keycloak: `registry.redhat.io/rhbk/keycloak-rhel9:26.6`
- OPA: `openpolicyagent/opa:0.62.1`
- keycloak-config-cli: imagem compatível com Keycloak 26.x
- Build alvo: Keycloak 26.6.0

---

## Build

```bash
mvn clean package -DskipTests
```

Isso gerou o artefato esperado:

```text
target/keycloak-opa.jar
```

---

## Execução com HTTP

Este projeto foi validado com Podman no ambiente local, mas o comando equivalente com Docker também é suportado quando o Docker estiver disponível.

### Podman

```bash
podman compose -f dev/docker-compose.yml up -d --force-recreate
```

### Docker

```bash
docker compose -f dev/docker-compose.yml up
```

Assim que a stack estiver no ar, o console administrativo fica disponível em:

```text
http://localhost:8080/auth
```

Credenciais padrão do admin usadas no demo:

```text
admin / admin
```

---

## Execução com HTTPS

Este exemplo usa `https://id.kubecon.test:8443/auth` como URL do servidor de autenticação do Keycloak.

Para usar HTTPS, adicione um mapeamento para `id.kubecon.test` no seu `/etc/hosts` e gere os certificados com o [mkcert](https://github.com/FiloSottile/mkcert).

```bash
(cd dev/config/certs && mkcert -install && mkcert -cert-file kubecon.pem -key-file kubecon-key.pem "*.kubecon.test")
```

Em seguida, inicie o arquivo de compose HTTPS:

```bash
docker compose -f dev/docker-compose-https.yml up
```

---

## Realm do demo

O demo contém um realm chamado `opademo`, provisionado por `dev/config/realms/opademo.yaml` por meio do [keycloak-config-cli](https://github.com/adorsys/keycloak-config-cli).

### Usuários

O demo inclui os seguintes usuários:

- Usuário: `tester` / Senha: `test`
- Usuário: `admin` / Senha: `test`
- Usuário: `guest` / Senha: `test`

### Clientes

O realm `opademo` inclui vários clientes de exemplo para demonstrar decisões de acesso expressas em REGO via OPA.

---

## Política de acesso do OPA

As políticas de acesso dos clientes estão definidas em:

```text
dev/opa/policies/keycloak/realms/opademo/access/policy.rego
```

O OPA observa esse arquivo para atualizações e está configurado para avaliar decisões de acesso por meio do SPI do Keycloak.

Para habilitar a verificação da política na interface do realm, navegue em:

```text
Realm opademo -> Authentication -> Required Actions -> Enable OPA Policy Check
```

---

## Configuração do realm

O realm, os clientes, roles, grupos e usuários estão definidos em:

```text
dev/config/realms/opademo.yaml
```

Para ajustar o comportamento de acesso do usuário `tester`, remova o comentário da atribuição relevante de role ou grupo em `opademo.yaml` e reimporte o realm com o provisionador.

Fluxo típico de reimportação/reinicialização:

```bash
podman compose -f dev/docker-compose.yml up -d --force-recreate
```

---

## Validações executadas em runtime

Os seguintes checks foram executados com sucesso neste ambiente:

```bash
mvn clean package -DskipTests
podman compose -f dev/docker-compose.yml up -d --force-recreate
curl -sS http://localhost:8181/health
curl -sS http://localhost:8181/v1/data/keycloak/realms/opademo/access
curl -sS -X POST 'http://localhost:8080/auth/realms/opademo/protocol/openid-connect/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'client_id=admin-cli' \
  --data-urlencode 'grant_type=password' \
  --data-urlencode 'username=tester' \
  --data-urlencode 'password=test'
```

Resultados observados na validação:

- `BUILD SUCCESS`
- O Keycloak reportou a versão `26.6.7.redhat-00003`
- O OPA respondeu com dados sob `keycloak/realms/opademo`
- a requisição de token retornou um bearer token válido

---

## Observações sobre a migração para RHBK 26.x

Os principais problemas da migração não foram o build Java em si, mas itens de compatibilidade de runtime, como:

- referência incorreta da imagem Red Hat
- flags de runtime descontinuadas incompatíveis com Keycloak 26.x
- incompatibilidade entre o provisionador antigo e o target do Keycloak 26.x
- dependências locais de JAR frágeis

O repositório foi ajustado para funcionar com a imagem enterprise suportada e com um fluxo de provisionamento compatível.

---

## Resumo final

A migração para RHBK 26.6 foi concluída com sucesso no repositório atual. A stack do demo está funcional, com Keycloak, OPA e realm `opademo` operando no ambiente target da Red Hat.

O ponto principal da migração foi corrigir a compatibilidade do runtime e do provisionador da linha 26.x, e isso foi validado em execução real com build, import do realm e obtenção de token do usuário do demo.
