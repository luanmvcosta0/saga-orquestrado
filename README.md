# 🔄 Saga Orquestrado — Microsserviços com Kafka

Arquitetura de **microsserviços** implementando o padrão **Saga Orquestrado** para garantir consistência de dados em transações distribuídas. Um pedido criado no `order-service` dispara uma saga coordenada pelo `orchestrator-service`, que conduz a validação do produto, o pagamento e a baixa de estoque via eventos no **Apache Kafka** — com fluxo de compensação (rollback) em caso de falha em qualquer etapa.

> 🚧 **Status: em desenvolvimento.** A infraestrutura está completa (serviços, tópicos Kafka, bancos e orquestração de containers); a implementação da lógica da saga está em andamento (veja o [roadmap](#-roadmap)).

## 🏗️ Arquitetura

```
                        ┌──────────────────────┐
                        │    order-service      │  :3000  (MongoDB)
                        │  cria o pedido e      │
                        │  inicia a saga        │
                        └──────────┬───────────┘
                                   │ start-saga
                                   ▼
                        ┌──────────────────────┐
              ┌────────▶│ orchestrator-service  │  :8080
              │         │  coordena os passos   │
              │         └──────────┬───────────┘
              │                    │ eventos success/fail por etapa
              │    ┌───────────────┼────────────────┐
              │    ▼               ▼                ▼
        ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
        │   product-   │  │   payment-   │  │  inventory-  │
        │  validation  │  │   service    │  │   service    │
        │    :8090     │  │    :8091     │  │    :8092     │
        │ (PostgreSQL) │  │ (PostgreSQL) │  │ (PostgreSQL) │
        └──────────────┘  └──────────────┘  └──────────────┘

                    Apache Kafka como barramento de eventos
```

### Os serviços

| Serviço | Porta | Banco | Responsabilidade |
|---------|-------|-------|------------------|
| `order-service` | 3000 | MongoDB | Ponto de entrada: cria pedidos e inicia/finaliza a saga |
| `orchestrator-service` | 8080 | — | Orquestrador: decide o próximo passo ou a compensação com base nos eventos |
| `product-validation-service` | 8090 | PostgreSQL | Valida a existência dos produtos do pedido |
| `payment-service` | 8091 | PostgreSQL | Processa o pagamento do pedido |
| `inventory-service` | 8092 | PostgreSQL | Realiza a baixa no estoque |

### Tópicos Kafka

A comunicação entre os serviços acontece por 11 tópicos, criados automaticamente via `TopicBuilder` na subida do orquestrador:

- `start-saga` / `orchestrator` / `notify-ending`
- `finish-success` / `finish-fail`
- `product-validation-success` / `product-validation-fail`
- `payment-success` / `payment-fail`
- `inventory-success` / `inventory-fail`

Cada etapa publica seu resultado (`success`/`fail`) e o orquestrador reage: avança para o próximo passo da saga ou dispara os eventos de **compensação** para desfazer as etapas anteriores.

## 🛠️ Tecnologias

- **Java 17**
- **Spring Boot 3** (Web, Kafka)
- **Apache Kafka** — mensageria entre os serviços
- **MongoDB** — banco do order-service
- **PostgreSQL** — bancos dos serviços de produto, pagamento e estoque
- **Redpanda Console** — UI para inspecionar tópicos e mensagens do Kafka
- **Docker & Docker Compose** — orquestração de todo o ambiente
- **Gradle** — build dos serviços
- **Python** — script de automação do build (`build.py`)

## 🚀 Como rodar

### Pré-requisitos

- Docker e Docker Compose
- Java 17+ e Gradle (para build local)
- Python 3 (opcional, para o script de automação)

### Subindo tudo com o script de build

O `build.py` compila os 5 serviços em paralelo e sobe todos os containers:

```bash
git clone https://github.com/luanmvcosta0/saga-orquestrado.git
cd saga-orquestrado
python build.py
```

### Ou manualmente

```bash
# Build de cada serviço
cd order-service && gradle build -x test && cd ..
# (repita para os demais serviços)

# Subida dos containers
docker-compose up --build -d
```

### Acessos

| Recurso | URL |
|---------|-----|
| Order Service (API de entrada) | `http://localhost:3000` |
| Redpanda Console (tópicos Kafka) | `http://localhost:8081` |
| Kafka (externo) | `localhost:9092` |

## 🗺️ Roadmap

- [x] Estrutura dos 5 microsserviços com Spring Boot + Gradle
- [x] Configuração de producers/consumers Kafka em todos os serviços
- [x] Definição e criação automática dos 11 tópicos da saga
- [x] `docker-compose` com Kafka, MongoDB, 3 PostgreSQL e Redpanda Console
- [x] Script de build paralelo dos serviços (`build.py`)
- [ ] Endpoint de criação de pedido no `order-service` (MongoDB)
- [ ] Lógica de orquestração da saga (avanço de etapas e compensação)
- [ ] Validação de produtos, processamento de pagamento e baixa de estoque
- [ ] Histórico/rastreabilidade dos eventos da saga por pedido
- [ ] Testes de integração do fluxo completo (sucesso e rollback)

## 🎯 Conceitos aplicados

- **Padrão Saga (orquestrado)** para transações distribuídas — alternativa ao 2PC em microsserviços
- **Database per service** — cada serviço com seu próprio banco (MongoDB e PostgreSQL)
- **Mensageria assíncrona** com Kafka: producers, consumers, consumer groups e tópicos dedicados por evento
- **Compensação (rollback semântico)** em caso de falha em qualquer etapa
- Containerização de um ambiente multi-serviço com **Docker Compose**
- Observabilidade dos eventos com **Redpanda Console**

---

Feito por [Luan Costa](https://github.com/luanmvcosta0) 👋
