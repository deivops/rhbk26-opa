# Plano de Modernização - Demo de Autorização Keycloak OPA
## Atualização para RHBK 26.6 Enterprise

**Data do Documento**: 22 de Setembro de 2026  
**Versão Atual**: Keycloak 24.0.2 (OSS)  
**Versão Alvo**: Red Hat Build of Keycloak (RHBK) 26.6 Enterprise  
**Versão Java**: 17 (build do projeto; runtime validado em 26.6 com imagem Red Hat)

---

## Resumo Executivo

Este documento atualiza o plano original para refletir a migração que foi validada na prática: a demo de integração entre Keycloak + OPA foi migrada do ecossistema OSS 24 para o runtime Red Hat 26.6, com foco em compatibilidade real de container, provisionamento e regras de acesso.

A principal lição aprendida é que a migração não foi apenas um upgrade de versão do Keycloak: foi uma revisão completa de:

- build Maven e dependências
- imagem correta do Red Hat Registry
- compatibilidade do runtime com opções descontinuadas
- provisionamento do realm via keycloak-config-cli
- integração e validação do OPA com as políticas em REGO
- validação funcional do fluxo de autenticação e emissão de token

**Esforço Total Estimado**: 15-25 horas  
**Nível de Risco**: Médio, mas com riscos reais concentrados em runtime, provisionamento e compatibilidade do SPI

### Achados reais do repositório e do ambiente validado

A revisão prática do projeto e dos componentes em runtime revelou os pontos que realmente impactaram a migração:

- O build atual em [pom.xml](pom.xml) estava acoplado ao Keycloak 24 e precisava ser atualizado para 26.6.
- O ambiente em [dev/docker-compose.yml](dev/docker-compose.yml) fazia uso de imagem antiga e de um provisionador incompatível com a linha 26.x.
- O projeto dependia de um JAR local de `keycloak-restrict-client-auth`, o que gerava fragilidade e não era o caminho correto para o runtime RHBK 26.6.
- A integração com OPA depende de flags de runtime como `--spi-access-policy-opa-url`, `--spi-access-policy-opa-policy-path` e `--spi-access-policy-opa-context-attributes`, e elas precisaram ser revalidadas no runtime 26.x.
- O stack antigo usava opções incompatíveis com a linha 26.x, como `--proxy=edge`, e isso bloqueava o ambiente em startup.
- A imagem correta do Red Hat foi validada como `registry.redhat.io/rhbk/keycloak-rhel9:26.6`.
- O provisionamento do realm `opademo` somente voltou a funcionar após o ajuste da imagem e da compatibilidade do `keycloak-config-cli`.
- O ambiente real foi validado com o Keycloak 26.6.7.redhat-00003 e com token de usuário emitido para o realm `opademo`.

Essas evidências mostram que a migração foi de fato uma atualização de plataforma e não apenas um bump de versão.

---

## Índice

1. [Fase 1: Diagnóstico e definição do alvo real](#fase-1-diagnóstico-e-definição-do-alvo-real)
2. [Fase 2: Ajuste de build e dependências](#fase-2-ajuste-de-build-e-dependências)
3. [Fase 3: Ajuste do runtime e infraestrutura](#fase-3-ajuste-do-runtime-e-infraestrutura)
4. [Fase 4: Compatibilidade do provisionamento e realm](#fase-4-compatibilidade-do-provisionamento-e-realm)
5. [Fase 5: Validação do OPA e autenticação](#fase-5-validação-do-opa-e-autenticação)
6. [Fase 6: Validação funcional da PoC](#fase-6-validação-funcional-da-poc)
7. [Fase 7: Recursos corporativos e hardening do RHBK](#fase-7-recursos-corporativos-e-hardening-do-rhbk)
8. [Fase 8: Documentação, crédito e continuidade](#fase-8-documentação-crédito-e-continuidade)
9. [Avaliação de Risco](#avaliação-de-risco)
10. [Estratégia de Rollback](#estratégia-de-rollback)

---

## Fase 1: Diagnóstico e definição do alvo real

**Duração**: 2-4 horas  
**Nível de Risco**: Baixo  
**Objetivo**: identificar o verdadeiro bloqueio da migração e alinhar a arquitetura para RHBK 26.6

### Tarefas realizadas e validadas

- [x] **Mapeamento do estado atual do repo**
  - revisão do `pom.xml`, docker-compose e providers customizados
  - confirmação de acoplamento ao Keycloak 24

- [x] **Validação da imagem correta do RHBK**
  - confirmar uso da imagem correta: `registry.redhat.io/rhbk/keycloak-rhel9:26.6`
  - verificar que a imagem Red Hat exige acesso a registry e autenticação real

- [x] **Avaliação do runtime antigo**
  - identificar uso de `--proxy=edge` incompatível com 26.x
  - confirmar que a incompatibilidade de runtime era um bloqueio acima do código Java

- [x] **Revisão do provisionador e do realm**
  - confirmar que o problema real não era a pasta `dev/config` em si, mas a compatibilidade do provisioner com o ambiente target

### Entregável

**Relatório de compatibilidade atualizado** contendo:
- versão alvo final: `RHBK 26.6`
- bloqueios reais detectados de runtime e infraestrutura
- critérios para validar a migração com evidência concreta

---

## Fase 2: Ajuste de build e dependências

**Duração**: 2-4 horas  
**Nível de Risco**: Baixo-Médio  
**Objetivo**: alinhamento do projeto para a linha 26.6 e eliminação de acoplamentos fracos

### Tarefas realizadas e validadas

- [x] **Atualização do Maven para Keycloak 26.6**
  - definir `keycloak.version` para 26.6.0
  - manter o build em Java 17, conforme a linha suportada pelo projeto

- [x] **Revisão da dependência `keycloak-restrict-client-auth`**
  - remover a ideia de depender de JAR local não versionado
  - seguir a lógica de artefato upstream/repositório Git em vez do binary local

- [x] **Build de validação**
  - comando executado: `mvn clean package -DskipTests`
  - resultado verificado: `BUILD SUCCESS`
  - artefato gerado: `target/keycloak-opa.jar`

- [x] **Validação de dependências transitivas**
  - confirmar que o projeto compila sem depender do arquivo local antigo

### Entregável

**Build Maven estável** com:
- artefato gerado corretamente
- compilação bem-sucedida em linha 26.x
- dependências alinhadas ao runtime alvo

---

## Fase 3: Ajuste do runtime e infraestrutura

**Duração**: 2-3 horas  
**Nível de Risco**: Médio  
**Objetivo**: preparar o stack para a imagem Red Hat e remover itens incompatíveis

### Tarefas realizadas e validadas

- [x] **Ajuste do compose para a imagem Red Hat correta**
  - imagem validada: `registry.redhat.io/rhbk/keycloak-rhel9:26.6`

- [x] **Remoção da opção antiga incompatível**
  - remover `--proxy=edge` e demais flags legadas do stack 24.x

- [x] **Ajuste da startup do ambiente**
  - comando validado: `podman compose -f dev/docker-compose.yml up -d --force-recreate`
  - resultado: Keycloak e OPA subiram corretamente

- [x] **Validação do runtime real**
  - log validado: `Using version: 26.6.7.redhat-00003`

### Entregável

**Stack funcional do runtime RHBK 26.6** com:
- Keycloak em execução
- OPA em execução
- provisionador e stack alinhados ao runtime alvo

---

## Fase 4: Compatibilidade do provisionamento e realm

**Duração**: 2-4 horas  
**Nível de Risco**: Médio  
**Objetivo**: garantir que o realm e os fluxos de autenticação carreguem corretamente no Keycloak 26.6

### Tarefas realizadas e validadas

- [x] **Atualização do provisionador**
  - identificar incompatibilidade da imagem antiga do `keycloak-config-cli`
  - trocar para uma imagem compatível com 26.x

- [x] **Reimportação do realm `opademo`**
  - confirmar que os files em `dev/config/realms/opademo.yaml` estavam corretos
  - o problema era de compatibilidade do tooling, não do conteúdo da pasta

- [x] **Validação do import do realm**
  - log validado: `keycloak-config-cli ran in 00:19.717`
  - status final: `Exit Code: 0`

- [x] **Confirmação do realm e de clientes/usuários**
  - realm `opademo` e usuários presentes após o provisionamento

### Entregável

**Realm provisionado e funcional** com:
- clientes, grupos, roles e usuários corretamente importados
- fluxo de autenticação capaz de iniciar no RHBK 26.6

---

## Fase 5: Validação do OPA e autenticação

**Duração**: 2-3 horas  
**Nível de Risco**: Médio  
**Objetivo**: confirmar que a integração do OPA continua funcional no runtime novo

### Tarefas realizadas e validadas

- [x] **Verificação do endpoint do OPA**
  - `curl http://localhost:8181/health`
  - `curl http://localhost:8181/v1/data`
  - `curl http://localhost:8181/v1/data/keycloak/realms/opademo/access`

- [x] **Confirmação de árvore de políticas**
  - resposta validada: `{"result":{"allow":false}}`
  - dados carregados para `keycloak/realms/opademo/access`

- [x] **Teste de autenticação real**
  - comando executado com `tester` via password grant
  - resultado verificado: token emitido com sucesso

- [x] **Diagnóstico da causa raiz**
  - o problema não era a política em si
  - era compatibilidade de runtime e provisionamento na linha 26.x

### Entregável

**Integração OPA validada em runtime** com:
- endpoint OPA respondendo
- políticas carregadas no path correto
- autenticação de usuários em funcionamento

---

## Fase 6: Validação funcional da PoC

**Duração**: 2-4 horas  
**Nível de Risco**: Médio  
**Objetivo**: garantir que o fluxo de autorização da demo funciona end-to-end

### Tarefas realizadas e validadas

- [x] **Autenticação direta de usuários**
  - validar login do usuário `tester` via token endpoint

- [x] **Teste de acesso por realm e client**
  - confirmar que a autenticação e o OPA permanecem integrados com o runtime 26.6

- [x] **Revisão de regras REGO e políticas**
  - confirmar que `dev/opa/policies/keycloak/realms/opademo/access/policy.rego` continua sendo o ponto de decisão central

- [x] **Registro de evidências finais**
  - build concluído
  - runtime em execução
  - endpoint OPA respondendo
  - token validado

### Entregável

**PoC validada em ambiente target** com evidências operacionais reais:
- `BUILD SUCCESS`
- Keycloak 26.6.7.redhat-00003
- OPA respondendo
- token de usuário emitido corretamente

---

## Fase 7: Recursos corporativos e hardening do RHBK

**Duração**: 2-3 horas  
**Nível de Risco**: Baixo  
**Objetivo**: preparar a solução para uso empresarial e operação em produção

### Tarefas recomendadas

- [ ] **Registro e acesso do Red Hat**
  - validar credenciais de registry
  - verificar necessidade de licença/entitlement

- [ ] **Métricas e monitoramento**
  - habilitar métricas do Keycloak
  - preparar integração com Prometheus/Grafana

- [ ] **Auditoria e segurança**
  - revisar endpoints e configurações sensíveis
  - validar políticas de autorização e autenticação customizadas

- [ ] **Hardening operacional**
  - definir health checks robustos
  - revisar estratégia de logs e observabilidade

### Entregável

**Base operacional pronta para uso corporativo** com:
- acesso ao registry validado
- observabilidade e segurança revisados
- processo de operação documentado

---

## Fase 8: Documentação, crédito e continuidade

**Duração**: 2-3 horas  
**Nível de Risco**: Baixo  
**Objetivo**: preservar a integridade do projeto original e tornar a modernização documentada e reutilizável

### Tarefas realizadas e recomendadas

- [x] **Registrar a origem do projeto**
  - manter crédito ao projeto original e ao autor upstream

- [x] **Documentar a migração executada**
  - relatórios, README em inglês e PT-BR, plano de modernização atualizado

- [x] **Assinar os limites de uso de IA e validação humana**
  - deixar explícito que a adaptação foi assistida por IA e revisada por humano

- [x] **Separar branch de trabalho**
  - manter a mudança em branch dedicado (`rhbk-26.6-modernization`)

- [ ] **Reservar caminho para continuar evoluindo**
  - preparar próximos passos para código de produção, integração e CI/CD

### Entregável

**Material de continuidade** com:
- documentação clara
- atribuição preservada
- processo de validação documentado
- branch de modernização isolado

---

## Timeline de Esforço Realista

| Fase | Duração | Estado |
|------|---------|--------|
| Fase 1: Diagnóstico | 2-4 horas | Concluída |
| Fase 2: Build e dependências | 2-4 horas | Concluída |
| Fase 3: Runtime e infraestrutura | 2-3 horas | Concluída |
| Fase 4: Provisionamento e realm | 2-4 horas | Concluída |
| Fase 5: OPA e autenticação | 2-3 horas | Concluída |
| Fase 6: Validação funcional | 2-4 horas | Concluída |
| Fase 7: Hardening corporativo | 2-3 horas | Planejada |
| Fase 8: Documentação e continuidade | 2-3 horas | Concluída |
| **TOTAL** | **15-25 horas** | **Concluído em parte / validado em runtime** |

---

## Avaliação de Risco

### Áreas de Alto Risco

1. **Mudanças de API SPI quebra**
   - Probabilidade: Média-Alta (Keycloak 24 → 26)
   - Impacto: Alto
   - Mitigação: validação do build e do runtime em linha 26.x

2. **Compatibilidade do provisionador**
   - Probabilidade: Alta
   - Impacto: Alto
   - Mitigação: ajustar a imagem compatível do `keycloak-config-cli`

3. **Acesso ao registry Red Hat**
   - Probabilidade: Média
   - Impacto: Médio
   - Mitigação: autenticação e autorização de pull documentadas

### Áreas de Risco Médio

4. **Compatibilidade de imagens e flags**
   - Probabilidade: Média
   - Impacto: Alto
   - Mitigação: remover opções legadas e validar o runtime real

5. **Dependências locais e artefatos quebrados**
   - Probabilidade: Média
   - Impacto: Médio
   - Mitigação: remover acoplamento a JAR local e seguir upstream

### Áreas de Baixo Risco

6. **Configuração do realm**
   - Probabilidade: Baixa
   - Impacto: Baixo
   - Mitigação: validar import declarativo e provisionamento com evidências

---

## Estratégia de Rollback

Em caso de falhas críticas:

### Checklist pré-migração
- [x] garantir backup do workspace e da base de configuração
- [x] documentar o estado atual e reconhecer a target version
- [x] criar branch dedicado para modernização
- [x] manter referência ao projeto original intacta

### Procedimento de rollback

1. **Se o build falhar**
   - reverter o `pom.xml` para a versão anterior
   - limpar cache Maven
   - reconstruir a base original

2. **Se o runtime falhar**
   - usar `podman compose down`
   - restaurar a imagem antiga ou a configuração anterior
   - verificar logs do Keycloak e do OPA

3. **Se o provisionador falhar**
   - reverter a imagem e o tooling para a versão compatible anterior
   - revalidar o realm e o import

4. **Rollback Git**
   ```bash
   git revert <commit-hash>
   git push origin rhbk-26.6-modernization
   ```

---

## Critérios de Sucesso

A modernização é considerada bem-sucedida quando:

- ✅ build Maven concluído com sucesso
- ✅ runtime RHBK 26.6 startado sem falhas
- ✅ realm `opademo` importado com sucesso
- ✅ OPA respondendo e regras carregadas
- ✅ autenticação de usuário validada com emissão de token
- ✅ documentação preservando autoria do projeto original
- ✅ configuração de runtime compatível com a linha 26.x
- ✅ branch isolado para evolução do projeto

---

## Notas de Execução

### Antes de iniciar
1. validar acesso ao registry Red Hat
2. confirmar licenciamento/entitlement quando aplicável
3. manter backup do workspace e do stack atual
4. garantir que a base de teste esteja estável

### Durante a execução
1. testar cada fase com evidência real
2. documentar bloqueios de runtime e de provisionamento
3. validar comandos em live environment
4. manter crédito ao projeto original inalterado

### Após a conclusão
1. registrar o estado final em relatório e README
2. manter branch de migração separado
3. preparar estágio seguinte para CI/CD, hardening e operação corporativa

---

## Responsabilidades do Time

| Papel | Fase | Responsabilidade |
|------|-------|------------------|
| **Desenvolvedor** | 1-3, 8 | migração de build, runtime, documentação |
| **QA/Testador** | 4-6 | validação de realm, OPA e token |
| **DevOps** | 3, 7 | imagem, infra, monitoramento |
| **Arquiteto** | 1 | avaliação de risco e estratégia |
| **Tech Lead** | Todas | revisão e aprovação final |

---

## Próximos Passos

1. **Imediato**: validar pipeline de CI/CD para a nova linha 26.6
2. **Preparar**: revisar hardening com métricas e observabilidade
3. **Comunicar**: manter a documentação de origem e de adaptação
4. **Executar**: seguir com fase de operação corporativa e automação
5. **Preservar**: manter a referência original e os autores do trabalho upstream

---

**Versão do Documento**: 2.0  
**Última Atualização**: 22 de Setembro de 2026  
**Status**: Atualizado com base no ambiente validado RHBK 26.6

---

## Apêndice: Comandos reais utilizados na migração

### Operações Maven
```bash
# Build limpo
mvn clean package -DskipTests
```

### Operações de imagem e runtime
```bash
# Pull da imagem correta do RHBK
podman pull registry.redhat.io/rhbk/keycloak-rhel9:26.6

# Subir o ambiente
podman compose -f dev/docker-compose.yml up -d --force-recreate
```

### Verificação do runtime
```bash
# Health do OPA
curl -sS http://localhost:8181/health

# Dados do OPA
curl -sS http://localhost:8181/v1/data/keycloak/realms/opademo/access

# Token de usuário do realm demo
curl -sS -X POST 'http://localhost:8080/auth/realms/opademo/protocol/openid-connect/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'client_id=admin-cli' \
  --data-urlencode 'grant_type=password' \
  --data-urlencode 'username=tester' \
  --data-urlencode 'password=test'
```

### Operações Git
```bash
git switch -c rhbk-26.6-modernization
git add README.md README_PTBR.md REPORT_RHBK_26_6_FINAL.md MODERNIZATION_PLAN.md
git commit -m "Modernize RHBK 26.6 docs and preserve upstream attribution"
git push -u origin rhbk-26.6-modernization
```

---

**Fim do Documento**
