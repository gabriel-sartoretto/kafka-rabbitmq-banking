# kafka-rabbitmq-banking-alura

Repositório "guarda-chuva" que agrupa os três microsserviços do curso de mensageria (Kafka + RabbitMQ) da Alura.
Cada serviço continua no **seu próprio repositório**; aqui eles entram como [git submodules](https://git-scm.com/book/pt-br/v2/Git-Tools-Submodules).

| Serviço | Repositório | Porta | Papel |
|---|---|---|---|
| banking-validation | [mensageria-banking-validation](https://github.com/gabriel-sartoretto/mensageria-banking-validation) | 8181 | Fonte da situação cadastral das agências. **Producer** RabbitMQ e Kafka, dono da saga |
| banking-service | [mensageria-banking-service](https://github.com/gabriel-sartoretto/mensageria-banking-service) | 8080 | Cadastro de agências. **Consumer** Kafka (remove agência inativada) |
| banking-audit | [mensageria-banking-audit](https://github.com/gabriel-sartoretto/mensageria-banking-audit) | 8282 | Auditoria. **Consumer** RabbitMQ (grava cada mudança de situação) |

## Arquitetura

```
                 PUT /situacao-cadastral
                          │
                          ▼
              ┌───────────────────────┐   RabbitMQ (exchange "notificacoes",   ┌──────────────────┐
              │  banking-validation   │── routing key agencia.change_status) ─▶│  banking-audit   │
              │        :8181          │                                        │      :8282       │
              │  (tabela saga + job   │                                        └──────────────────┘
              │   de resync a cada 10s)│
              └───────────────────────┘
                 │ Kafka (tópico remover-agencia-avro, Avro + Schema Registry)
                 │ só quando a situação vira INATIVO
                 ▼
              ┌───────────────────────┐  GET /situacao-cadastral/{cnpj} (ao cadastrar)
              │    banking-service    │─────────────────────────────────▶ validation
              │        :8080          │  PUT /saga/{sucesso|ignorada|erro} (fecha a saga)
              └───────────────────────┘─────────────────────────────────▶ validation
```

## Clonando

```bash
git clone --recurse-submodules <url-deste-repo>
# ou, se já clonou sem os submodules:
git submodule update --init --recursive
```

## Subindo o ambiente

A infraestrutura compartilhada (RabbitMQ, Kafka, Zookeeper, Schema Registry e o Postgres do validation) está no
`docker-compose.yml` do **banking-validation**. Os outros dois têm cada um o seu Postgres.

```bash
cd mensageria-banking-validation && docker compose up -d && cd ..
cd mensageria-banking-audit      && docker compose up -d postgres-db-alura-audit && cd ..
cd mensageria-banking-service    && docker compose up -d postgres-db-alura-banking-service && cd ..
```

Depois rode cada serviço em modo dev (`./mvnw quarkus:dev`) dentro da sua pasta.

## Trabalhando com os submodules

- Commits e pushes são feitos **dentro de cada pasta de serviço**, no repositório dele, como sempre.
- Este repo guarda só *qual commit* de cada serviço está referenciado. Depois de commitar num serviço,
  atualize a referência aqui: `git add mensageria-banking-<nome> && git commit -m "atualiza <nome>"`.
- Para puxar a última versão de todos: `git submodule update --remote --merge`.
