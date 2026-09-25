# CLAUDE.md

Contexto para o Claude trabalhar neste workspace. Visão geral e instruções de execução estão no `README.md`.

## Estrutura

Repositório guarda-chuva com 3 **git submodules** independentes (cada um com seu remote em
`github.com/gabriel-sartoretto`, branch `master`). O repo raiz só versiona README, CLAUDE.md, `.gitmodules` e os
ponteiros de commit dos submodules.

- **Nunca** mova código de um serviço para outro nem para a raiz.
- Mudança em um serviço = commit **dentro da pasta do serviço**. Só depois atualize o ponteiro na raiz.
- Mudanças que cruzam serviços (contratos de mensagem, endpoints REST) precisam ser feitas nos dois lados.

Stack comum: Java + Quarkus, reativo (Mutiny `Uni`, Hibernate Reactive Panache, `@WithTransaction`/`@WithSession`),
SmallRye Reactive Messaging, Postgres 14. Pacote base `br.com.alura`. Código e comentários em **português**.

## Serviços

### mensageria-banking-validation (porta 8181, Postgres `agencia` em 5432)
Dono da situação cadastral e orquestrador da saga.
- `SituacaoCadastralController` — `POST/GET/PUT /situacao-cadastral`, `GET /situacao-cadastral/{cnpj}` (204 se não achar).
- `SituacaoCadastralService.alterar` — UPDATE condicional (`situacaoCadastral <> ?1`), então repetição não gera
  efeito duplicado. Se alterou: envia `Audit` pelo RabbitMQ; se virou `INATIVO`, grava uma `Saga` (UUID novo, `OPEN`)
  e publica `br.com.alura.Agencia` (Avro) com o `sagaId` no Kafka.
- `SagaController` — `PUT /saga/sucesso|ignorada|erro` (body = id da saga) → `COMPLETED|IGNORED|ERROR`.
- `SagaResyncService` — `@Scheduled(every 10s, delayed 10s)` reenvia sagas `OPEN` com mais de 2 minutos.
- `docker-compose.yml` sobe **toda a infra compartilhada**: RabbitMQ (5672/15672), Zookeeper, Kafka (9092),
  Schema Registry (8081). O container da própria API está comentado de propósito (roda pela IDE; senão conflita na 8181).

### mensageria-banking-service (porta 8080, Postgres `agencia` em 5433)
- `AgenciaController` — `POST /agencia`: consulta o validation via REST Client (`situacao-cadastral-api`),
  só persiste se `ATIVO` e se o CNPJ ainda não existe.
- `RemoverAgenciaService` — `@Incoming("remover-agencia-channel")`, grupo `banking-service-consumer-group`.
  Agência inexistente → fecha saga como ignorada; nome contendo `"ERRO"` → **falha simulada** (fecha como erro);
  senão deleta e fecha como sucesso. Mensagens sem `sagaId` não fecham saga.
- `SagaHttpService` — REST Client para os endpoints `/saga/*` do validation.

### mensageria-banking-audit (porta 8282, Postgres `audit` em 5434)
- `BankingAuditService` — `@Incoming("notificacoes")`, fila `notificacao.audit`, exchange direct, com DLQ
  (routing key `agencia.change_status.dlq`). Recebe `JsonObject` e grava `cnpj` + `situacaoCadastral` na tabela `audit`.

## Contratos entre serviços

| Canal | Producer → Consumer | Formato |
|---|---|---|
| RabbitMQ exchange `notificacoes`, routing key `agencia.change_status` | validation → audit | JSON (`Audit`: id, cnpj, situacaoCadastral) |
| Kafka tópico `remover-agencia-avro` | validation → service | Avro `br.com.alura.Agencia` |
| REST `GET /situacao-cadastral/{cnpj}` | service → validation | JSON |
| REST `PUT /saga/{sucesso,ignorada,erro}` | service → validation | body = sagaId |

**O schema Avro está duplicado** em `src/main/avro/Agencia.avsc` do validation e do service — hoje idênticos.
Qualquer alteração deve ser feita nos dois arquivos e ser compatível (novos campos com `default`, como `sagaId`).

## Comandos

```bash
./mvnw quarkus:dev      # dentro da pasta do serviço
./mvnw package          # build
./mvnw test
git submodule status    # na raiz: commit referenciado de cada serviço
```

Devservices estão desligados (`quarkus.devservices.enabled=false`): a infra precisa estar no ar via docker compose.
