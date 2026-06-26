# 🏥 Patient Management System 
## Microservices & Architecture


> Projeto de exemplo construído com **Java + Spring Boot**, seguindo uma arquitetura de microsserviços com comunicação via **REST**, **gRPC** e **Kafka**.

---

## 📑 Sumário

- [Visão Geral](#-visão-geral)
- [Arquitetura](#-arquitetura)
- [Serviços](#-serviços)
    - [Patient Service](#1-patient-service)
    - [Billing Service](#2-billing-service)
    - [Notification Service](#3-notification-service)
    - [Auth Service](#4-auth-service)
- [Infraestrutura](#-infraestrutura)
    - [Kafka](#kafka)
    - [Bancos de Dados](#bancos-de-dados)
- [Pré-requisitos](#-pré-requisitos)
- [Como Executar](#-como-executar)
- [Variáveis de Ambiente — Resumo](#-variáveis-de-ambiente--resumo)
- [Contribuindo](#-contribuindo)
- [Licença](#-licença)
---

## 🧭 Visão Geral

O sistema é composto por quatro microsserviços independentes que se comunicam entre si através de diferentes protocolos, dependendo da necessidade:

| Serviço | Responsabilidade | Comunicação |
|---|---|---|
| **Patient Service** | Cadastro e gerenciamento de pacientes | REST (entrada) · gRPC (saída p/ Billing) · Kafka (eventos) |
| **Billing Service** | Geração e controle de cobranças | gRPC (servidor) |
| **Notification Service** | Envio de notificações a partir de eventos | Kafka (consumidor) |
| **Auth Service** | Autenticação e emissão de tokens JWT | REST · JWT |

Cada serviço possui seu próprio banco de dados (padrão *Database per Service*), reforçando o isolamento e a independência entre os módulos.
 
---

## 🏗️ Arquitetura

```
                ┌──────────────────┐
                │   Auth Service    │
                │   (JWT / REST)    │
                └─────────┬─────────┘
                          │
                          ▼
┌──────────────┐   gRPC   ┌──────────────────┐
│   Cliente    │ ───────▶ │ Patient Service   │
│ (REST API)   │          │                   │
└──────────────┘          └─────────┬─────────┘
                                     │ gRPC
                                     ▼
                           ┌──────────────────┐
                           │  Billing Service  │
                           └──────────────────┘
                                     │
                                     │ Kafka (evento)
                                     ▼
                           ┌──────────────────────┐
                           │ Notification Service │
                           └──────────────────────┘
```

- **Patient Service** recebe requisições REST, persiste dados no Postgres e dispara chamadas **gRPC** para o **Billing Service**, além de publicar eventos no **Kafka**.
- **Billing Service** expõe um servidor **gRPC** para processar cobranças.
- **Notification Service** consome eventos do **Kafka** (mensagens em Protobuf) e dispara notificações.
- **Auth Service** centraliza autenticação, emitindo tokens JWT validados pelos demais serviços.
---

## 🧩 Serviços

### 1. Patient Service

Responsável pelo cadastro e gerenciamento de pacientes. Integra-se ao **Billing Service** via gRPC e publica eventos no **Kafka** para o **Notification Service**.

#### Variáveis de Ambiente

```bash
BILLING_SERVICE_ADDRESS=billing-service
BILLING_SERVICE_GRPC_PORT=9005
JAVA_TOOL_OPTIONS=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
SPRING_DATASOURCE_PASSWORD=password
SPRING_DATASOURCE_URL=jdbc:postgresql://patient-service-db:5432/db
SPRING_DATASOURCE_USERNAME=admin_user
SPRING_JPA_HIBERNATE_DDL_AUTO=update
SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
SPRING_SQL_INIT_MODE=always
```

> 💡 `JAVA_TOOL_OPTIONS` habilita o **remote debug** na porta `5005`. Remova essa variável em ambientes de produção.

#### Dependências gRPC (`pom.xml`)

Adicione ao bloco `<dependencies>`:

```xml
<!-- GRPC -->
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-netty-shaded</artifactId>
    <version>1.69.0</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-protobuf</artifactId>
    <version>1.69.0</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-stub</artifactId>
    <version>1.69.0</version>
</dependency>
<dependency> <!-- necessário para Java 9+ -->
    <groupId>org.apache.tomcat</groupId>
    <artifactId>annotations-api</artifactId>
    <version>6.0.53</version>
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>4.29.1</version>
</dependency>
```

Substitua o bloco `<build>` por:

```xml
<build>
    <extensions>
        <!-- Garante compatibilidade de OS para o protoc -->
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.0</version>
        </extension>
    </extensions>
    <plugins>
        <!-- Spring Boot / Maven -->
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
 
        <!-- PROTO -->
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>com.google.protobuf:protoc:3.25.5:exe:${os.detected.classifier}</protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.68.1:exe:${os.detected.classifier}</pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

#### Kafka Producer (`application.properties`)

```properties
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.ByteArrayDeserializer
```
 
---

### 2. Billing Service

Expõe um servidor **gRPC** responsável por registrar e processar cobranças vinculadas a pacientes.

#### Dependências gRPC (`pom.xml`)

Adicione ao bloco `<dependencies>`:

```xml
<!-- GRPC -->
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-netty-shaded</artifactId>
    <version>1.69.0</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-protobuf</artifactId>
    <version>1.69.0</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-stub</artifactId>
    <version>1.69.0</version>
</dependency>
<dependency> <!-- necessário para Java 9+ -->
    <groupId>org.apache.tomcat</groupId>
    <artifactId>annotations-api</artifactId>
    <version>6.0.53</version>
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>4.29.1</version>
</dependency>
```

Substitua o bloco `<build>` (idêntico ao do Patient Service):

```xml
<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.0</version>
        </extension>
    </extensions>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>com.google.protobuf:protoc:3.25.5:exe:${os.detected.classifier}</protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.68.1:exe:${os.detected.classifier}</pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

> ℹ️ O Billing Service escuta na porta gRPC `9005`, referenciada por `BILLING_SERVICE_GRPC_PORT` no Patient Service.
 
---

### 3. Notification Service

Consome eventos publicados no **Kafka** (serializados em **Protobuf**) e é responsável por disparar notificações relacionadas a eventos de pacientes/cobranças.

#### Variáveis de Ambiente

```bash
SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
```

#### Dependências (adicionar às existentes)

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
    <version>3.3.0</version>
</dependency>
 
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>4.29.1</version>
</dependency>
```

#### Bloco `<build>` do `pom.xml`

```xml
<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.0</version>
        </extension>
    </extensions>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>com.google.protobuf:protoc:3.25.5:exe:${os.detected.classifier}</protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.68.1:exe:${os.detected.classifier}</pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```
 
---

### 4. Auth Service

Responsável por autenticar usuários e emitir tokens **JWT** consumidos pelos demais serviços.

#### Dependências (adicionar às existentes)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
 
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.6.0</version>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
</dependency>
```

#### Variáveis de Ambiente

```bash
SPRING_DATASOURCE_PASSWORD=password
SPRING_DATASOURCE_URL=jdbc:postgresql://auth-service-db:5432/db
SPRING_DATASOURCE_USERNAME=admin_user
SPRING_JPA_HIBERNATE_DDL_AUTO=update
SPRING_SQL_INIT_MODE=always
```

#### `data.sql` — usuário inicial

Script executado automaticamente na inicialização (graças a `SPRING_SQL_INIT_MODE=always`) para garantir a existência da tabela `users` e de um usuário administrador de teste:

```sql
-- Garante que a tabela 'users' existe
CREATE TABLE IF NOT EXISTS "users" (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL
);
 
-- Insere o usuário apenas se não houver conflito de id ou email
INSERT INTO "users" (id, email, password, role)
SELECT '223e4567-e89b-12d3-a456-426614174006', 'testuser@test.com',
       '$2b$12$7hoRZfJrRKD2nIm2vHLs7OBETy.LWenXXMLKf99W8M4PUwO6KB7fu', 'ADMIN'
WHERE NOT EXISTS (
    SELECT 1
    FROM "users"
    WHERE id = '223e4567-e89b-12d3-a456-426614174006'
       OR email = 'testuser@test.com'
);
```

> ⚠️ **Atenção:** essa senha já vem hasheada (bcrypt) e serve apenas para fins de desenvolvimento/teste. **Nunca** reutilize essas credenciais em produção.

#### Auth Service DB — Variáveis de Ambiente

```bash
POSTGRES_DB=db
POSTGRES_PASSWORD=password
POSTGRES_USER=admin_user
```
 
---

## 🛠️ Infraestrutura

### Kafka

Configuração para execução do container Kafka localmente (ex.: via IntelliJ ou Docker), em modo *KRaft* (sem Zookeeper):

```bash
KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,EXTERNAL://localhost:9094
KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka:9093
KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT
KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093,EXTERNAL://:9094
KAFKA_CFG_NODE_ID=0
KAFKA_CFG_PROCESS_ROLES=controller,broker
```

| Listener | Porta | Uso |
|---|---|---|
| `PLAINTEXT` | `9092` | Comunicação interna entre serviços (rede Docker) |
| `CONTROLLER` | `9093` | Comunicação do controller (KRaft) |
| `EXTERNAL` | `9094` | Acesso externo (ex.: da máquina host) |

### Bancos de Dados

Cada serviço com persistência possui seu próprio banco PostgreSQL dedicado:

| Banco | Variáveis |
|---|---|
| `patient-service-db` | `SPRING_DATASOURCE_*` (ver Patient Service) |
| `auth-service-db` | `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` |
 
---

## ✅ Pré-requisitos

- **JDK 17+**
- **Maven 3.9+**
- **Docker** e **Docker Compose**
- **IntelliJ IDEA** (recomendado, para debug remoto e execução de containers)
- Portas livres: `5005` (debug), `9005` (gRPC Billing), `9092`/`9093`/`9094` (Kafka), `5432` (Postgres)
---

## 🚀 Como Executar

1. Clone o repositório e acesse a pasta raiz do projeto.
2. Configure as variáveis de ambiente de cada serviço conforme descrito nas seções acima (via `.env`, Docker Compose ou configuração do container na IDE).
3. Suba a infraestrutura (Kafka e bancos de dados) primeiro.
4. Construa os serviços com Maven:
```bash
   ./mvnw clean install
```
5. Inicie os serviços na seguinte ordem recomendada:
    1. **Auth Service** (+ `auth-service-db`)
    2. **Billing Service**
    3. **Patient Service** (+ `patient-service-db`)
    4. **Notification Service**
6. Verifique os logs de cada serviço para confirmar a conexão com o Kafka e os bancos de dados.
---

## 🔑 Variáveis de Ambiente — Resumo

<details>
<summary>Clique para expandir a lista completa</summary>
**Patient Service**
```bash
BILLING_SERVICE_ADDRESS=billing-service
BILLING_SERVICE_GRPC_PORT=9005
JAVA_TOOL_OPTIONS=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
SPRING_DATASOURCE_PASSWORD=password
SPRING_DATASOURCE_URL=jdbc:postgresql://patient-service-db:5432/db
SPRING_DATASOURCE_USERNAME=admin_user
SPRING_JPA_HIBERNATE_DDL_AUTO=update
SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
SPRING_SQL_INIT_MODE=always
```

**Notification Service**
```bash
SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
```

**Auth Service**
```bash
SPRING_DATASOURCE_PASSWORD=password
SPRING_DATASOURCE_URL=jdbc:postgresql://auth-service-db:5432/db
SPRING_DATASOURCE_USERNAME=admin_user
SPRING_JPA_HIBERNATE_DDL_AUTO=update
SPRING_SQL_INIT_MODE=always
```

**Auth Service DB**
```bash
POSTGRES_DB=db
POSTGRES_PASSWORD=password
POSTGRES_USER=admin_user
```

**Kafka**
```bash
KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,EXTERNAL://localhost:9094
KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka:9093
KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT
KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093,EXTERNAL://:9094
KAFKA_CFG_NODE_ID=0
KAFKA_CFG_PROCESS_ROLES=controller,broker
```

</details>
---
