# Microservices E-commerce Spring Boot

## 📋 Vue d'ensemble du Projet

Ce projet est une **architecture microservices complète** pour une application e-commerce utilisant:
- **Spring Boot 3.4.3**
- **Java 21/23**
- **MySQL & MongoDB**
- **Kafka** pour la messagerie asynchrone
- **Docker & Docker Compose** pour la containerisation
- **Observabilité** avec Prometheus, Grafana, Loki, Tempo
- **Résilience** avec Circuit Breaker (Resilience4J)
- **Sécurité OAuth2** avec Keycloak
- **Gateway pattern** avec Spring Cloud Gateway MVC

---

## 🏗️ Architecture Globale

```
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway (Port 9000)                  │
│         - Spring Cloud Gateway MVC                          │
│         - OAuth2 Resource Server (Keycloak)                 │
│         - Circuit Breaker + Resilience4J                    │
└─────────────────────────────────────────────────────────────┘
        │                    │                    │
        ▼                    ▼                    ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ PRODUCT SERVICE  │  │  ORDER SERVICE   │  │ INVENTORY SERVICE│
│   (Port 8080)    │  │   (Port 8081)    │  │   (Port 8082)    │
├──────────────────┤  ├──────────────────┤  ├──────────────────┤
│   MongoDB        │  │   MySQL          │  │   MySQL          │
│ - Port 27017     │  │ - order_service  │  │ - inventory_svc  │
│                  │  │                  │  │                  │
│ Models:          │  │ Models:          │  │ Models:          │
│ - Product        │  │ - Order          │  │ - Inventory      │
│                  │  │                  │  │                  │
│ Logic:           │  │ Logic:           │  │ Logic:           │
│ - CRUD Products  │  │ - Place Order    │  │ - Check Stock    │
│ - List Products  │  │ - Call Inventory │  │                  │
│                  │  │ - Send to Kafka  │  │                  │
└──────────────────┘  └──────────────────┘  └──────────────────┘
                             │
                      ┌──────▼──────┐
                      │    Kafka    │
                      │ (Port 9092) │
                      │ Topic:      │
                      │order-placed │
                      └─────────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ NOTIFICATION SVC │
                    │   (En création)  │
                    └──────────────────┘
```

---

## 📦 Services Microservices

### 1️⃣ **PRODUCT SERVICE** (Port 8080)
**Responsabilité**: Gestion complète des produits

#### 📂 Structure:
```
product-service/
├── src/main/java/com/springcommerce/product_service/
│   ├── ProductServiceApplication.java     (Main Application)
│   ├── controller/
│   │   └── ProductController.java        (REST Endpoints)
│   ├── service/
│   │   └── ProductService.java           (Business Logic)
│   ├── repository/
│   │   └── ProductRepository.java        (Data Access - MongoDB)
│   ├── model/
│   │   └── Product.java                  (Entity MongoDB)
│   ├── dto/
│   │   ├── ProductRequest.java           (Request DTO)
│   │   └── ProductResponse.java          (Response DTO)
│   └── config/
│       └── ObservationConfig.java        (Monitoring)
├── src/main/resources/
│   ├── application.properties            (Configuration)
│   └── logback-spring.xml                (Logging + Loki)
└── pom.xml                               (Maven Dependencies)
```

#### 🗄️ Entité: **Product**
```java
@Document(value = "product")
- id (String) - MongoDB ID
- name (String)
- description (String)
- price (BigDecimal)
```

#### 📡 APIs:
| Méthode | Endpoint | Description |
|---------|----------|-------------|
| POST | `/api/product` | Créer un produit |
| GET | `/api/product` | Lister tous les produits |

#### 💾 Base de Données:
- **MongoDB** (Port 27017)
- Database: `product-service`
- Collection: `product`
- Auth: root/password

#### 📚 Dépendances clés:
- spring-boot-starter-data-mongodb
- spring-boot-starter-web
- lombok
- testcontainers (MongoDB)
- micrometer-registry-prometheus
- micrometer-tracing-bridge-brave

---

### 2️⃣ **ORDER SERVICE** (Port 8081)
**Responsabilité**: Gestion des commandes avec orchestration inter-services

#### 📂 Structure:
```
order-service/
├── src/main/java/com/springcommerce/order_service/
│   ├── OrderServiceApplication.java      (Main Application)
│   ├── controller/
│   │   └── OrderController.java          (REST Endpoints)
│   ├── service/
│   │   └── OrderService.java             (Business Logic)
│   ├── repository/
│   │   └── OrderRepository.java          (Data Access - JPA)
│   ├── model/
│   │   └── Order.java                    (Entity JPA)
│   ├── dto/
│   │   └── OrderRequest.java             (Request DTO)
│   ├── client/
│   │   └── InventoryClient.java          (HTTP Client + Resilience4J)
│   ├── event/
│   │   └── OrderPlacedEvent.java         (Kafka Event)
│   └── config/
│       ├── RestClientConfig.java         (HTTP Client Config)
│       └── ObservationConfig.java        (Monitoring)
├── src/main/resources/
│   ├── application.properties            (Configuration)
│   ├── db/migration/
│   │   └── V1__init.sql                  (Flyway Migration)
│   ├── avro/
│   │   └── order-placed.avsc             (Avro Schema for Kafka)
│   └── logback-spring.xml                (Logging + Loki)
└── pom.xml
```

#### 🗄️ Entité: **Order**
```java
@Entity
@Table(name = "t_orders")
- id (Long) - Primary Key
- orderNumber (String) - UUID
- skuCode (String) - Product Code
- price (BigDecimal)
- quantity (Integer)
```

#### 📡 APIs:
| Méthode | Endpoint | Description |
|---------|----------|-------------|
| POST | `/api/order` | Placer une commande |

#### 🔌 Intégrations:
- **InventoryClient**: Appel HTTP à Inventory Service pour vérifier le stock
  - Circuit Breaker avec fallback (retourne false si service indisponible)
  - Retry automatique (3 tentatives avec délai de 2s)
  - Timeout: 3 secondes

#### 📨 Kafka Integration:
- **Topic**: `order-placed`
- **Format**: Apache Avro
- **Événement**: OrderPlacedEvent (orderNumber, email, firstName, lastName)
- **Schema Registry**: http://localhost:8085

#### ⚙️ Processus de Commande:
1. Validation de la demande
2. Vérifier le stock via InventoryClient
3. Si stock disponible:
   - Créer et persister la commande en BD
   - Publier OrderPlacedEvent sur Kafka
4. Si pas de stock: Lancer une exception RuntimeException

#### 💾 Base de Données:
- **MySQL** (Port 3306)
- Database: `order_service`
- Table: `t_orders`
- Migrations: Flyway (V1__init.sql)

#### 📚 Dépendances clés:
- spring-boot-starter-data-jpa
- spring-boot-starter-web
- mysql-connector-j
- flyway-core / flyway-mysql
- spring-kafka + kafka-avro-serializer
- resilience4j-circuitbreaker + resilience4j-retry
- spring-cloud-starter-contract-stub-runner (Tests)
- micrometer-tracing

---

### 3️⃣ **INVENTORY SERVICE** (Port 8082)
**Responsabilité**: Gestion du stock/inventaire

#### 📂 Structure:
```
inventory-service/
├── src/main/java/com/springcommerce/inventory_service/
│   ├── InventoryServiceApplication.java  (Main Application)
│   ├── controller/
│   │   └── InventoryController.java      (REST Endpoints)
│   ├── service/
│   │   └── InventoryService.java         (Business Logic)
│   ├── repository/
│   │   └── InventoryRepository.java      (Data Access - JPA)
│   ├── model/
│   │   └── Inventory.java                (Entity JPA)
│   └── config/
│       └── ObservationConfig.java        (Monitoring)
├── src/main/resources/
│   ├── application.properties            (Configuration)
│   ├── db/migration/
│   │   ├── V1__init.sql                  (Flyway - Init table)
│   │   └── V2__add_inventory.sql         (Flyway - Insert data)
│   └── logback-spring.xml                (Logging + Loki)
└── pom.xml
```

#### 🗄️ Entité: **Inventory**
```java
@Entity
@Table(name = "t_inventory")
- id (Long) - Primary Key
- skuCode (String) - Product SKU Code
- quantity (Integer) - Available quantity
```

#### 📡 APIs:
| Méthode | Endpoint | Description |
|---------|----------|-------------|
| GET | `/api/inventory?skuCode=...&quantity=...` | Vérifier si le stock est disponible |

#### 💾 Base de Données:
- **MySQL** (Port 3307)
- Database: `inventory_service`
- Table: `t_inventory`
- Migrations: Flyway
  - V1: Création de la table
  - V2: Insertion de données d'exemple

#### 📚 Dépendances clés:
- spring-boot-starter-data-jpa
- spring-boot-starter-web
- mysql-connector-j
- flyway-core / flyway-mysql
- micrometer-tracing

---

### 4️⃣ **API GATEWAY** (Port 9000)
**Responsabilité**: Point d'entrée unique, routing, sécurité et résilience

#### 📂 Structure:
```
api-gateway/
├── src/main/java/com/springcommerce/api_gateway/
│   ├── ApiGatewayApplication.java        (Main Application)
│   ├── routes/
│   │   └── Routes.java                   (Route Configuration)
│   └── config/
│       ├── SecurityConfig.java           (OAuth2 Security)
│       └── ObservationConfig.java        (Monitoring)
├── src/main/resources/
│   ├── application.properties            (Configuration)
│   └── logback-spring.xml                (Logging + Loki)
└── pom.xml
```

#### 🛣️ Routing Configuration:
```
Gateway (9000)
├── /api/product/** → Product Service (8080)
│   - Circuit Breaker: productServiceCircuitBreaker
│   - Fallback: /fallbackRoute (503 Service Unavailable)
│
├── /api/order/** → Order Service (8081)
│   - Circuit Breaker: orderServiceCircuitBreaker
│   - Fallback: /fallbackRoute (503 Service Unavailable)
│
└── /api/inventory/** → Inventory Service (8082)
    - Circuit Breaker: inventoryServiceCircuitBreaker
    - Fallback: /fallbackRoute (503 Service Unavailable)
```

#### 📚 Dépendances clés:
- spring-cloud-starter-gateway-mvc
- spring-boot-starter-oauth2-resource-server
- spring-cloud-starter-circuitbreaker-resilience4j
- spring-boot-starter-actuator
- micrometer-registry-prometheus
- micrometer-tracing-bridge-brave

---

### 5️⃣ **NOTIFICATION SERVICE** (À implémenter)
**Responsabilité**: Traiter les événements Kafka et envoyer les notifications

#### 🎯 Objectif:
- Consommer les événements `order-placed` depuis Kafka
- Envoyer des emails de confirmation de commande
- Intégrer avec un service d'email (ex: SendGrid, AWS SES)

---

## 🗄️ Configuration des Bases de Données

### MySQL
```
Host: localhost
Port: 3306 (Order Service), 3307 (Inventory Service)
Username: root
Password: mysql

Databases:
- order_service
- inventory_service
```

### MongoDB
```
Host: localhost
Port: 27017
Username: root
Password: password
Authentication Source: admin

Database: product-service
Collections:
- product
```

---

## 📊 Observabilité et Monitoring

### 1. **Prometheus** (Métriques)
- Endpoint: `/actuator/prometheus` sur chaque service
- Scrape: Tous les services exposent les métriques Prometheus

### 2. **Grafana** (Visualisation)
- Affiche les métriques Prometheus
- Tableaux de bord pour la performance

### 3. **Loki** (Logs)
- Centralisation des logs
- Configuration: `logback-spring.xml` avec Loki appender
- Intégration Grafana pour la recherche de logs

### 4. **Tempo** (Distributed Tracing)
- Traçage distribué des requêtes
- Micrometer Brave Bridge: Envoie les traces à Tempo
- 100% sampling (tous les appels tracés)

### 5. **Actuator Endpoints**
Disponibles sur chaque service:
```
/actuator/health
/actuator/metrics
/actuator/prometheus
/actuator/health/circuitbreakers (API Gateway, Order Service)
```

---

## 🔄 Flux de Données - Cas d'usage: Placer une Commande

```
1. Client
   └─> POST /api/order (via API Gateway)
       └─> Authentification OAuth2 (Keycloak)

2. API Gateway (Port 9000)
   └─> Route /api/order/** vers Order Service (8081)
       └─> Circuit Breaker "orderServiceCircuitBreaker"

3. Order Service
   ├─> Recevoir OrderRequest (skuCode, price, quantity, userDetails)
   │
   ├─> Appeler InventoryClient.isInStock(skuCode, quantity)
   │   └─> GET /api/inventory (vers Inventory Service)
   │       └─> InventoryService.isInStock()
   │       └─> Query: SELECT quantity FROM t_inventory WHERE skuCode = ? AND quantity >= ?
   │       └─> Retour: true/false
   │           └─> Si service indisponible: CircuitBreaker fallback → false
   │
   ├─> Si stock disponible:
   │   ├─> Créer Order entity (orderNumber=UUID, skuCode, price, quantity)
   │   ├─> Persist en BD MySQL (t_orders)
   │   ├─> Créer OrderPlacedEvent (orderNumber, email, firstName, lastName)
   │   ├─> Publier sur Kafka topic "order-placed" (format Avro)
   │   │   └─> Schema Registry: http://localhost:8085
   │   └─> Log: "Sending OrderPlacedEvent to Kafka topic order-placed"
   │
   └─> Si stock indisponible:
       └─> Throw RuntimeException "Product ... is not in stock"

4. Kafka
   └─> Topic: order-placed
       └─> Événement disponible pour les consumers

5. Notification Service (à implémenter)
   └─> Consumer du topic "order-placed"
       └─> Traiter l'événement
       └─> Envoyer email de confirmation
```

---

## 🛠️ Stack Technologique Complet

### Framework & Runtime
- **Java 21/23**
- **Spring Boot 3.4.3**
- **Spring Cloud 2024.0.0**

### Web & API
- **Spring Web MVC**
- **Spring Cloud Gateway MVC**
- **REST** avec DTOs (Records)

### Données
- **Spring Data MongoDB** (Document DB)
- **Spring Data JPA** (Relational DB)
- **MySQL 8** (Order, Inventory Services)
- **MongoDB 5+** (Product Service)
- **Flyway** (Database Migrations)

### Intégration
- **Spring Kafka** (Event-driven)
- **Apache Avro** (Data serialization)
- **Confluent Schema Registry** (Avro schema management)
- **RestClient** (HTTP calls between services)

### Résilience & Stabilité
- **Resilience4J** (Circuit Breaker, Retry, Timeout)
- **Spring Boot Actuator** (Health checks)

### Sécurité
- **OAuth2 Resource Server**
- **Keycloak** (IAM)
- **JWT tokens**

### Testing
- **JUnit 5**
- **TestContainers** (MongoDB, MySQL)
- **REST Assured** (API testing)
- **Hamcrest** (Matchers)
- **Spring Cloud Contract Stub Runner** (Contract testing)

### Observabilité
- **Micrometer Prometheus** (Metrics)
- **Micrometer Tracing** (Distributed Tracing)
- **Brave** (Tracing client)
- **Zipkin Reporter** (Trace transport)
- **Loki** (Log aggregation)
- **Prometheus** (Time-series DB)
- **Grafana** (Visualization)
- **Tempo** (Trace backend)

### Logging
- **SLF4J** (Logging facade)
- **Logback** (Logging implementation)
- **Loki Appender** (Send logs to Loki)

### Build & DevOps
- **Maven 3.8+**
- **Docker & Docker Compose**
- **Lombok** (Reduce boilerplate)

---

## 🚀 Docker Compose

Le projet utilise Docker Compose pour orchestrer tous les services:

**Services lancés par docker-compose up:**
1. **MongoDB** - Product data store
2. **MySQL** - Order & Inventory data store
3. **Kafka** - Message broker
4. **Schema Registry** - Avro schema management
5. **Prometheus** - Metrics collector
6. **Grafana** - Metrics visualization
7. **Loki** - Log aggregation
8. **Tempo** - Distributed tracing
9. **Keycloak** - Identity & Access Management

---

## 📋 DTOs et Records

### ProductRequest / ProductResponse
```java
public record ProductRequest(String id, String name, String description, BigDecimal price)
public record ProductResponse(String id, String name, String description, BigDecimal price)
```

### OrderRequest (avec User Details)
```java
public record OrderRequest(
    Long id, 
    String orderNumber, 
    String skuCode, 
    BigDecimal price, 
    Integer quantity, 
    UserDetails userDetails
) {
    public record UserDetails(String email, String firstName, String lastName) {}
}
```

### OrderPlacedEvent (Kafka)
```java
- orderNumber: String
- email: String
- firstName: String
- lastName: String
```

---

## 🔍 Patterns Utilisés

1. **Microservices Pattern**: Chaque service indépendant avec sa BD
2. **API Gateway Pattern**: Point d'entrée unique (API Gateway)
3. **Database per Service**: Chaque service gère sa propre BD
4. **Async Communication**: Kafka pour la communication asynchrone (Order → Notification)
5. **Synchronous Communication**: REST/HTTP pour les requêtes synchrones (Order → Inventory)
6. **Saga Pattern**: Order Service utilise Kafka pour le workflow distribué
7. **Circuit Breaker Pattern**: Resilience4J pour éviter les cascades de défaillances
8. **Service Discovery**: Spring Cloud Gateway pour le routing
9. **Centralized Logging**: Loki pour l'agrégation des logs
10. **Distributed Tracing**: Tempo pour tracer les requêtes cross-service
11. **Observability**: Prometheus + Grafana + Loki + Tempo (les 3 piliers)

---

## 🔐 Flux de Sécurité

```
Client Request
  ↓
API Gateway (9000) - Vérifie JWT Token (Keycloak)
  ├─ Valid Token → Forward to service
  └─ Invalid Token → 401 Unauthorized

Service Processing
  ↓
Database Access (avec credentials stockés en config)
```

