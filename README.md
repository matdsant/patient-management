# Patient Management System — PMS 🏥

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
| **Infrastructure** | Provisionamento da stack na AWS (via LocalStack) | AWS CDK (Java) | — | — |

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

> 💡 Para inspecionar os dados localmente (ex.: via IntelliJ Data Sources, DBeaver, pgAdmin),
> conecte em `localhost:5433` (auth-service-db) ou `localhost:5432` (patient-service-db)
> usando as credenciais acima.

> ⚠️ `JwtUtil` também exige uma property `jwt.secret` (chave Base64) para assinar/validar o token, mas ela **não** está definida em `application.properties` nem no `docker-compose.yml` — veja o roadmap abaixo.

Endpoints (`AuthController`):

| Método | Path | Descrição |
|---|---|---|
| `POST` | `/login` | Recebe `{ email, password }`, valida contra `UserRepository` (senha com `BCryptPasswordEncoder`) e retorna `{ token }` (JWT, expira em 10h) |
| `GET` | `/validate` | Recebe `Authorization: Bearer <token>` e retorna 200/401 conforme a assinatura/validade do token (`JwtUtil`) |

`SecurityConfig` hoje libera todas as rotas (`permitAll()`) e desabilita CSRF — a própria emissão/validação do JWT é quem controla o acesso. `data.sql` popula a tabela `users` com um usuário de teste (`testuser@test.com` / `password123`, role `ADMIN`) se ela ainda não existir.

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

# login para obter o token JWT
curl -X POST http://localhost:4004/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"testuser@test.com","password":"password123"}'

# usa o token para acessar a rota protegida
curl http://localhost:4004/api/patients -H "Authorization: Bearer <token>"
```

---

## 📄 Documentação Swagger

Apenas `auth-service` e `patient-service` expõem REST diretamente e têm Swagger habilitado (`billing-service` é gRPC e `analytics-service` é consumidor Kafka, sem endpoints REST).

**Swagger UI (interface completa, com "Try it out"):**

| Serviço | URL |
|---|---|
| auth-service | http://localhost:4005/swagger-ui.html |
| patient-service | http://localhost:4000/swagger-ui.html |

**OpenAPI JSON via `api-gateway`** (sem UI, só o contrato cru — o gateway não expõe uma interface Swagger própria):

| Serviço | URL |
|---|---|
| auth-service | http://localhost:4004/api-docs/auth |
| patient-service | http://localhost:4004/api-docs/patients |

> Rotas protegidas por `JwtValidation` (ex.: `GET /api/patients`) só aceitam requisições via `api-gateway` (`4004`) com header `Authorization: Bearer <token>`. Para testar sem o gateway, use a porta direta do serviço — nesse caso o filtro de JWT não se aplica.

---

## ✅ Testes de Integração

Módulo `integration-tests` (JUnit 5 + REST Assured), apontando para o `api-gateway` em `http://localhost:4004`. Requer a stack no ar (`docker compose up -d`) antes de rodar:

```bash
cd integration-tests
mvn test
```

- `AuthIntegrationTest` — login com credenciais válidas retorna 200 + token; credenciais inválidas retornam 401.
- `PatientIntegrationTest` — faz login, usa o token para chamar `GET /api/patients` via gateway e valida o retorno.

---

## ☁️ Infraestrutura (LocalStack / AWS CDK)

Módulo `infrastructure` define, em AWS CDK (Java), o provisionamento da stack completa para simulação local via **LocalStack**: VPC, um `DatabaseInstance` Postgres por serviço com estado (`auth-service`, `patient-service`) com health check, um cluster MSK para o Kafka, um cluster ECS Fargate com um serviço por módulo (`auth-service`, `billing-service`, `analytics-service`, `patient-service`) e um `api-gateway` exposto via Application Load Balancer.

```bash
cd infrastructure
mvn compile exec:java   # gera o template CDK em ./cdk.out

# aplica o stack no LocalStack e recupera o DNS do load balancer
./localstack-deploy.sh
```

> Pressupõe LocalStack rodando localmente e a AWS CLI configurada para apontar para `http://localhost:4566`.

---

## ✅ Resumo do Fluxo (explicação rápida)

> Sistema de microsserviços com comunicação assíncrona via **Kafka** e chamadas síncronas via **gRPC**: quando um paciente é criado, um evento é publicado e o Analytics Service reage a esse evento, enquanto o Billing Service é chamado de forma síncrona para gerar a cobrança.

---

## 🚧 Roadmap Técnico / Melhorias Futuras

> Levantamento de pontos de melhoria identificados em auditoria técnica do código atual. Organizado por prioridade para servir de backlog.

### 🔴 Pontos Críticos

- [x] ~~**Reimplementar `auth-service`**~~ — `AuthController`, `AuthService`, `UserService`, `JwtUtil`, `User`/`UserRepository`, DTOs e `SecurityConfig` já implementados, com `data.sql` populando um usuário de teste.
- [x] ~~**Definir `JWT_SECRET` no `auth-service`**~~ — `JWT_SECRET=Y2hhVEc3aHJnb0hYTzMyZ2ZqVkpiZ1RkZG93YWxrUkM=` adicionada às variáveis de ambiente do `auth-service` no `docker-compose.yml` (mesmo valor de referência já usado em `infrastructure`/CDK).
- [ ] **Endurecer `SecurityConfig` do auth-service** — hoje libera todas as rotas (`anyRequest().permitAll()`) só para viabilizar `/login` e `/validate` publicamente; vale revisar se algo mais deveria exigir autenticação conforme o serviço crescer.
- [ ] **Fechar as portas dos serviços internos no host** — `patient-service` (4000), `billing-service` (4001/9001) e `analytics-service` (4002) são acessíveis diretamente via Docker, sem passar pelo `api-gateway` e sem nenhuma checagem de JWT própria. O filtro `JwtValidation` do gateway é hoje o único ponto de auth do sistema e é trivialmente contornável.
- [x] ~~**Criar testes de verdade**~~ — o módulo `integration-tests` agora cobre o fluxo de login (`AuthIntegrationTest`) e o acesso autenticado a `GET /api/patients` via gateway (`PatientIntegrationTest`). Ainda faltam testes unitários para `PatientController`, `PatientService`, `PatientMapper`, `GlobalExceptionHandler`, `JwtValidationGatewayFilterFactory`, `BillingGrpcService` e `KafkaConsumer`.
- [ ] **Tratar erro da chamada gRPC em `PatientService.createPatient()`** — hoje, se o `billing-service` estiver fora do ar, o paciente já foi salvo no Postgres e a exceção do gRPC sobe sem tratamento, retornando 500 sem indicar se a cobrança/evento Kafka foi criado. Avaliar `@Transactional` + tratamento explícito da falha (compensação ou fila de retry).
- [ ] **Implementar `billing-service` de verdade** — `BillingGrpcService.createBillingAccount()` sempre retorna `accountId = "12345"` fixo, ignora o request e não tem entidade, repositório nem persistência.

### 🟡 Pontos Importantes

- [ ] Adicionar paginação em `GET /patients` (hoje usa `findAll()` sem limite).
- [ ] Criar endpoint `GET /patients/{id}` (hoje só existe listagem completa).
- [ ] Corrigir códigos HTTP: `PatientNotFoundException` deveria retornar 404 (hoje retorna 400) e `EmailAlreadyExistsException` deveria retornar 409 (hoje retorna 400).
- [ ] Adotar Flyway ou Liquibase para migrações — hoje o schema é gerenciado só por `ddl-auto=update` + `data.sql`.
- [ ] Adicionar Spring Boot Actuator (`/health`, `/metrics`) nos 5 serviços e configurar `healthcheck:` no `docker-compose.yml` para cada um (hoje só Postgres e Kafka têm healthcheck; `depends_on: service_started` não garante que a aplicação já esteja pronta).
- [ ] Adicionar rate limiting no `api-gateway` (nenhum filtro de limite de requisições configurado hoje).
- [ ] Mover credenciais dos bancos (`POSTGRES_PASSWORD=password`, repetida em dois bancos) para `.env`/secrets em vez de texto plano no `docker-compose.yml`.
- [ ] Configurar dead-letter queue / retry no consumidor Kafka do `analytics-service` (hoje uma falha de desserialização só loga e descarta a mensagem).
- [ ] Adicionar timeout/tratamento de erro em `JwtValidationGatewayFilterFactory` para quando `auth-service` estiver fora do ar ou lento.

### 🟢 Pontos de Melhorias

- [ ] Unificar os arquivos `.proto` duplicados (`billing_service.proto` existe idêntico em `billing-service` e `patient-service`; `patient_event.proto` existe idêntico em `patient-service` e `analytics-service`) em um módulo de contrato compartilhado.
- [ ] Padronizar o formato de resposta de erro entre serviços — hoje só `patient-service` tem `@ControllerAdvice`, e mesmo lá o formato varia entre erros de validação e erros de negócio.
- [ ] Adicionar versionamento de API (ex.: prefixo `/v1/...`).
- [ ] Reduzir duplicação entre `application.yml` e `application-prod.yml` no `api-gateway` (hoje duplicam a lista inteira de rotas só para trocar o host).
- [ ] Trocar `String` por `LocalDate` (com validação de formato) nos campos de data do `PatientRequestDTO`, para falhar de forma limpa em vez de estourar exceção no parse.

---