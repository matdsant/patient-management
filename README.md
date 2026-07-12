# Patient Management System — Microsserviços Java/Spring

---

## 📖 Sobre o Projeto

> Este projeto é um sistema de **microsserviços** construído com Spring Boot, que gerencia o cadastro de pacientes e dispara automaticamente um fluxo de eventos entre serviços para tratar **cobrança** e **notificações**.

A comunicação entre os serviços combina dois modelos:

- **Assíncrona (event-driven)** via **Apache Kafka** — usada para propagar eventos, como a criação de um paciente, para os serviços interessados (ex.: notificações).
- **Síncrona** via **gRPC / Protobuf** — usada quando um serviço precisa chamar outro diretamente e aguardar uma resposta (ex.: Patient Service → Billing Service).

### Fluxo típico

1. Um paciente/usuário é criado (Auth Service / Patient Service).
2. O serviço responsável publica um **evento no Kafka** descrevendo o novo cadastro.
3. Serviços consumidores (ex.: **Notification Service**) reagem ao evento e executam suas ações (enviar notificação, etc.).
4. Chamadas síncronas via **gRPC** são usadas quando é necessária uma resposta imediata (ex.: geração de cobrança no **Billing Service**).

---

## 🧱 Arquitetura de Serviços

| Serviço | Responsabilidade | Comunicação |
|---|---|---|
| **Auth Service** | Autenticação e gerenciamento de usuários (JWT) | REST + banco próprio |
| **Patient Service** | Cadastro e gestão de pacientes | Kafka (producer) + gRPC (client do Billing) |
| **Billing Service** | Geração de cobranças | gRPC (server) |
| **Notification Service** | Envio de notificações | Kafka (consumer) |

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

### 🩺 Patient Service

**Variáveis de ambiente:**

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

**Dependências gRPC** (adicionar em `<dependencies>`):

```xml
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

**Configuração do `<build>`** (substituir a seção existente):

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

**Kafka Producer** — adicionar em `application.properties`:

```properties
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.ByteArrayDeserializer
```

---

### 💳 Billing Service

Usa a **mesma configuração de gRPC e `<build>`** do Patient Service (dependências e plugin `protobuf-maven-plugin` acima).

---

### 🔔 Notification Service

**Variáveis de ambiente:**

```bash
SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
```

**Dependências** (adicionar às já existentes):

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

**Configuração do `<build>`** (pom.xml):

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

### 🔐 Auth Service

**Dependências** (adicionar às já existentes):

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

**Variáveis de ambiente:**

```bash
SPRING_DATASOURCE_PASSWORD=password
SPRING_DATASOURCE_URL=jdbc:postgresql://auth-service-db:5432/db
SPRING_DATASOURCE_USERNAME=admin_user
SPRING_JPA_HIBERNATE_DDL_AUTO=update
SPRING_SQL_INIT_MODE=always
```

**`data.sql`** (cria a tabela `users` e insere um usuário de teste, caso não exista):

```sql
-- Garante a existência da tabela 'users'
CREATE TABLE IF NOT EXISTS "users" (
                                       id UUID PRIMARY KEY,
                                       email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL
    );

-- Insere o usuário caso não exista um com o mesmo id ou email
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

---

### 🗄️ Auth Service DB

**Variáveis de ambiente:**

```bash
POSTGRES_DB=db
POSTGRES_PASSWORD=password
POSTGRES_USER=admin_user
```

---

## 🐳 Kafka (Container)

Variáveis de ambiente para rodar o container do Kafka localmente (ex.: via IntelliJ):

```bash
KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,EXTERNAL://localhost:9094
KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka:9093
KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT
KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093,EXTERNAL://:9094
KAFKA_CFG_NODE_ID=0
KAFKA_CFG_PROCESS_ROLES=controller,broker
```

---

## ✅ Resumo do Fluxo (explicação rápida)

> Sistema de microsserviços com comunicação assíncrona via **Kafka** e chamadas síncronas via **gRPC**: quando um paciente é criado, um evento é publicado e serviços como cobrança e notificações reagem a esse evento para completar o fluxo de cadastro e faturamento.

---

