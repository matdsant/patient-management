# Patient Management System — PMS 📬

---

> ## 📖 Sobre o Projeto
> Este projeto é um sistema de **microsserviços** construído com Spring Boot, que gerencia o cadastro de pacientes e dispara automaticamente um fluxo de eventos entre serviços para tratar **cobrança** e **analytics**.
> A comunicação entre os serviços combina dois modelos:
> - **Assíncrona (event-driven)** via **Apache Kafka** — usada para propagar eventos, como a criação de um paciente, para os serviços interessados (ex.: analytics).
> - **Síncrona** via **gRPC / Protobuf** — usada quando um serviço precisa chamar outro diretamente e aguardar uma resposta (ex.: Patient Service → Billing Service).
>> ### Fluxo típico
>> 1. Um paciente é criado no **Patient Service** (via API Gateway, com JWT emitido pelo Auth Service).
>> 2. O Patient Service publica um **evento no tópico Kafka `patient`** descrevendo o novo cadastro.
>> 3. O **Analytics Service** consome esse evento (consumer group `analytics-service`) para fins de análise.
>> 4. Em paralelo, o Patient Service chama o **Billing Service via gRPC** (síncrono) para gerar a cobrança correspondente.

---

## 🧱 Arquitetura de Serviços

| Serviço | Responsabilidade | Comunicação | Porta HTTP | Outras portas |
|---|---|---|---|---|
| **API Gateway** | Ponto único de entrada, roteamento e validação de JWT | Spring Cloud Gateway (reativo) | 4004 | — |
| **Auth Service** | Autenticação e gerenciamento de usuários (JWT) | REST + banco próprio | 4005 | — |
| **Patient Service** | Cadastro e gestão de pacientes | Kafka (producer) + gRPC (client do Billing) | 4000 | — |
| **Billing Service** | Geração de cobranças | gRPC (server) | 4001 | gRPC: 9001 |
| **Analytics Service** | Consome eventos de pacientes para análise | Kafka (consumer) | 4002 | — |
| **Integration Tests** | Testes de integração ponta a ponta (JUnit 5 + REST Assured) | HTTP, contra os serviços acima | — | — |

> ⚠️ **Status atual**: `auth-service` e `integration-tests` têm apenas o scaffolding (pom.xml, classe principal/estrutura de pacotes) — a lógica de negócio (`AuthController`, `SecurityConfig`, `JwtUtil`, os testes de integração etc.) ainda precisa ser implementada. Isso é resultado de uma perda de conteúdo identificada no histórico do git (commit que apagou arquivos achando que estava "recuperando" algo); veja `git log --oneline -- auth-service` para o contexto.

---

## 🛠️ Tecnologias

- Java / Spring Boot
- Spring Security + JWT (`jjwt`)
- Apache Kafka
- gRPC + Protocol Buffers
- PostgreSQL
- H2 (testes)
- Docker / Docker Compose
- Springdoc OpenAPI (documentação de API)

---

## ⚙️ Configuração por Serviço

> As dependências Maven e variáveis de ambiente abaixo já estão configuradas nos respectivos `pom.xml` e no `docker-compose.yml` — esta seção serve como referência do que cada serviço espera, não como um passo a passo de setup.

### 🩺 Patient Service

**Variáveis de ambiente:**

```bash
BILLING_SERVICE_ADDRESS=billing-service
BILLING_SERVICE_GRPC_PORT=9001
SPRING_DATASOURCE_PASSWORD=password
SPRING_DATASOURCE_URL=jdbc:postgresql://patient-service-db:5432/db
SPRING_DATASOURCE_USERNAME=admin_user
SPRING_JPA_HIBERNATE_DDL_AUTO=update
SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
SPRING_SQL_INIT_MODE=always
```

Publica no tópico Kafka `patient` (`KafkaProducer.java`) e chama o Billing Service via gRPC (`BillingServiceGrpcClient.java`, propriedades `billing.service.address` / `billing.service.grpc.port`, default `localhost:9001`).

---

### 💳 Billing Service

**Variáveis de ambiente:**

```bash
SERVER_PORT=4001
GRPC_SERVER_PORT=9001
```

Servidor gRPC (`billing.BillingService`) consumido pelo Patient Service.

---

### 📊 Analytics Service

**Variáveis de ambiente:**

```bash
SERVER_PORT=4002
SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
```

Consome o tópico `patient` (`KafkaConsumer.java`, `groupId=analytics-service`).

---

### 🔐 Auth Service

**Variáveis de ambiente:**

```bash
SERVER_PORT=4005
SPRING_DATASOURCE_PASSWORD=password
SPRING_DATASOURCE_URL=jdbc:postgresql://auth-service-db:5432/db
SPRING_DATASOURCE_USERNAME=admin_user
SPRING_JPA_HIBERNATE_DDL_AUTO=update
SPRING_SQL_INIT_MODE=always
```

O `pom.xml` já inclui Spring Security, Spring Data JPA, `jjwt` e driver PostgreSQL. **Pendente**: implementar `AuthController`, `AuthService`, `UserService`, `SecurityConfig`, `JwtUtil`, os DTOs e o `User`/`UserRepository` (atualmente vazios) e criar um `data.sql` para popular a tabela `users`, por exemplo:

```sql
CREATE TABLE IF NOT EXISTS "users" (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL
);

INSERT INTO "users" (id, email, password, role)
SELECT '223e4567-e89b-12d3-a456-426614174006', 'testuser@test.com',
       '$2b$12$7hoRZfJrRKD2nIm2vHLs7OBETy.LWenXXMLKf99W8M4PUwO6KB7fu', 'ADMIN'
    WHERE NOT EXISTS (
    SELECT 1 FROM "users"
    WHERE id = '223e4567-e89b-12d3-a456-426614174006'
       OR email = 'testuser@test.com'
);
```

---

### 🌐 API Gateway

Rotas configuradas em `application.yml` (Spring Cloud Gateway):

| Path | Destino | Filtro |
|---|---|---|
| `/auth/**` | `auth-service:4005` | `StripPrefix=1` |
| `/api/patients/**` | `patient-service:4000` | `StripPrefix=1`, `JwtValidation` |
| `/api-docs/patients` | `patient-service:4000/v3/api-docs` | `RewritePath` |
| `/api-docs/auth` | `auth-service:4005/v3/api-docs` | `RewritePath` |

O filtro `JwtValidation` (`JwtValidationGatewayFilterFactory.java`) valida o token chamando `GET /validate` no Auth Service antes de liberar a requisição.

---

### 🗄️ Bancos de dados (Postgres)

Uma instância por serviço com estado (`auth-service-db`, `patient-service-db`), mesmas credenciais:

```bash
POSTGRES_DB=db
POSTGRES_PASSWORD=password
POSTGRES_USER=admin_user
```

---

## 🐳 Kafka

O broker roda em modo **KRaft** (sem Zookeeper), imagem oficial `apache/kafka:3.9.2`:

```bash
KAFKA_NODE_ID=0
KAFKA_PROCESS_ROLES=controller,broker
KAFKA_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093,EXTERNAL://:9094
KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,EXTERNAL://localhost:9094
KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT
KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER
KAFKA_CONTROLLER_QUORUM_VOTERS=0@kafka:9093
```

De fora do Docker (ex.: um cliente Kafka local), use `localhost:9094`. Entre containers, use `kafka:9092`.

> Nota: a imagem `bitnami/kafka` deixou de publicar tags versionadas gratuitamente nesse repositório — por isso o projeto usa a imagem oficial da Apache.

---

## 🐳 Rodando localmente com Docker Compose

O `docker-compose.yml` na raiz sobe toda a stack (2x Postgres, Kafka, e os 5 serviços Spring Boot) em uma rede compartilhada, com limites de CPU/memória por container (heap da JVM ajustado via `-XX:MaxRAMPercentage` para caber no limite).

```bash
# builda as imagens e sobe tudo em background
docker compose up -d --build

# acompanhar logs de um serviço
docker compose logs -f patient-service

# derrubar (mantém os volumes/dados)
docker compose down

# derrubar e apagar os dados dos bancos/kafka também
docker compose down -v
```

**Portas expostas no host:**

| Serviço | Porta |
|---|---|
| api-gateway | `4004` |
| auth-service | `4005` |
| patient-service | `4000` |
| billing-service | `4001` (HTTP) / `9001` (gRPC) |
| analytics-service | `4002` |
| patient-service-db | `5432` |
| auth-service-db | `5433` |
| kafka (listener externo) | `9094` |

Exemplo de smoke test depois do `docker compose up`:

```bash
curl http://localhost:4004/api-docs/patients   # via API Gateway
curl http://localhost:4000/patients            # direto no Patient Service
```

---

## ✅ Resumo do Fluxo (explicação rápida)

> Sistema de microsserviços com comunicação assíncrona via **Kafka** e chamadas síncronas via **gRPC**: quando um paciente é criado, um evento é publicado e o Analytics Service reage a esse evento, enquanto o Billing Service é chamado de forma síncrona para gerar a cobrança.

---

