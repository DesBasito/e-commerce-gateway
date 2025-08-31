# Gateway Service

## 📋 Описание

Gateway Service - это API Gateway для e-commerce микросервисной системы, построенный на Spring Cloud Gateway. Он служит единой точкой входа для всех клиентских запросов и обеспечивает маршрутизацию, балансировку нагрузки, аутентификацию и другие cross-cutting функции.

## 🎯 Основные функции

- Единая точка входа для всех API запросов
- Маршрутизация запросов к соответствующим микросервисам
- Интеграция с Eureka для service discovery
- Автоматическая балансировка нагрузки
- Фильтрация запросов и ответов
- CORS поддержка
- Rate limiting (при необходимости)

## ⚙️ Технический стек

- **Java 17**
- **Spring Boot 3.5.4**
- **Spring Cloud Gateway**
- **Spring Cloud Config Client**
- **Spring Cloud Netflix Eureka Client**
- **Maven**

## 🚀 Запуск сервиса

### Предварительные условия
- Config Server (http://localhost:8888)
- Discovery Service (http://localhost:8761)
- Все микросервисы зарегистрированы в Eureka
- Java 17+
- Maven 3.6+

### Локальный запуск

```bash
# Клонирование репозитория
git clone <repository-url>
cd services/gateway

# Сборка проекта
./mvnw clean install

# Запуск сервиса
./mvnw spring-boot:run
```

## 🔧 Конфигурация

### application.yml
```yaml
spring:
  config:
    import: optional:configserver:http://localhost:8888
  application:
    name: gateway-service
```

### gateway-service.yml (в Config Server)
```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
      routes:
        - id: customer-service
          uri: lb://customer-service
          predicates:
            - Path=/api/v1/customers/**
        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/api/v1/products/**
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/v1/orders/**,/api/v1/order-lines/**
        - id: payment-service
          uri: lb://payment-service
          predicates:
            - Path=/api/v1/payments/**

server:
  port: 8888

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
```

## 🗺️ Маршрутизация

### Настроенные маршруты

#### Customer Service
- **Path**: `/api/v1/customers/**`
- **Target**: `http://customer-service/api/v1/customers/**`
- **Port**: 8090

#### Product Service
- **Path**: `/api/v1/products/**`
- **Target**: `http://product-service/api/v1/products/**`
- **Port**: 8050

#### Order Service
- **Path**: `/api/v1/orders/**`, `/api/v1/order-lines/**`
- **Target**: `http://order-service/api/v1/orders/**`
- **Port**: 8070

#### Payment Service
- **Path**: `/api/v1/payments/**`
- **Target**: `http://payment-service/api/v1/payments/**`
- **Port**: 8060

### Пример использования

Вместо прямого обращения к сервисам:
```bash
# Прямое обращение
curl http://localhost:8090/api/v1/customer

# Через Gateway
curl http://localhost:8888/api/v1/customers
```

## 🏗️ Архитектура

### Структура пакетов
```
kg.manurov.gateway/
└── GatewayApplication.java
```

### Основные компоненты

#### Spring Cloud Gateway
Автоматически обеспечивает:
- **Route Predicates** - условия для маршрутизации
- **Route Filters** - обработка запросов и ответов
- **Load Balancer** - балансировка нагрузки через Eureka
- **Circuit Breaker** - отказоустойчивость

#### Eureka Integration
Gateway автоматически обнаруживает сервисы через Eureka и обновляет маршруты при изменении топологии.

## 🔍 Функциональность Gateway

### Service Discovery Integration
```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true  # Автоматическое создание маршрутов из Eureka
          lower-case-service-id: true  # Использование lowercase для service ID
```

### Load Balancing
Gateway использует `lb://` префикс для автоматической балансировки нагрузки:
```yaml
routes:
  - id: customer-service
    uri: lb://customer-service  # Load balanced routing
```

### Path Rewriting
При необходимости можно изменять пути:
```yaml
routes:
  - id: customer-service
    uri: lb://customer-service
    predicates:
      - Path=/customers/**
    filters:
      - RewritePath=/customers/(?<segment>.*), /api/v1/customer/${segment}
```

## 🔧 Дополнительные фильтры

### CORS Configuration
```java
@Configuration
public class GatewayConfig {
    
    @Bean
    public CorsWebFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowCredentials(true);
        config.addAllowedOriginPattern("*");
        config.addAllowedHeader("*");
        config.addAllowedMethod("*");
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        
        return new CorsWebFilter(source);
    }
}
```

### Request/Response Logging
```yaml
logging:
  level:
    org.springframework.cloud.gateway: DEBUG
    org.springframework.web.reactive: DEBUG
```

### Custom Filters
```java
@Component
public class LoggingFilter implements GlobalFilter, Ordered {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        log.info("Request: {} {}", request.getMethod(), request.getURI());
        
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            ServerHttpResponse response = exchange.getResponse();
            log.info("Response: {}", response.getStatusCode());
        }));
    }
    
    @Override
    public int getOrder() {
        return -1;
    }
}
```

## 📊 Мониторинг

### Health Check
```http
GET /actuator/health
```

**Ответ:**
```json
{
  "status": "UP",
  "components": {
    "gateway": {
      "status": "UP"
    },
    "eureka": {
      "status": "UP"
    }
  }
}
```

### Gateway Routes Information
```http
GET /actuator/gateway/routes
```

**Ответ:**
```json
[
  {
    "route_id": "customer-service",
    "route_definition": {
      "id": "customer-service",
      "uri": "lb://customer-service",
      "predicates": [
        {
          "name": "Path",
          "args": {
            "pattern": "/api/v1/customers/**"
          }
        }
      ]
    }
  }
]
```

### Metrics
```http
GET /actuator/metrics/spring.cloud.gateway.requests
```

## 🐛 Устранение неполадок

### Частые проблемы

1. **Gateway не может найти сервисы**
    - Убедитесь, что Eureka Server запущен
    - Проверьте, что сервисы зарегистрированы в Eureka
    - Проверьте конфигурацию eureka.client.service-url

2. **404 ошибки при маршрутизации**
    - Проверьте правильность Path предикатов
    - Убедитесь, что целевые сервисы доступны
    - Проверьте логи Gateway для деталей маршрутизации

3. **Load Balancing не работает**
    - Убедитесь, что используется `lb://` префикс в URI
    - Проверьте наличие нескольких экземпляров сервиса в Eureka

4. **Timeout ошибки**
    - Настройте timeout для Gateway:
   ```yaml
   spring:
     cloud:
       gateway:
         httpclient:
           connect-timeout: 10000
           response-timeout: 60s
   ```

5. **CORS ошибки**
    - Добавьте CORS конфигурацию
    - Проверьте allowed origins и headers

### Отладка маршрутизации
```yaml
logging:
  level:
    org.springframework.cloud.gateway.route: DEBUG
    org.springframework.cloud.gateway.filter: DEBUG
```

## 🔧 Дополнительные возможности

### Circuit Breaker Integration
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: customer-service
          uri: lb://customer-service
          predicates:
            - Path=/api/v1/customers/**
          filters:
            - name: CircuitBreaker
              args:
                name: customerServiceCB
                fallbackUri: forward:/fallback/customers
```

### Rate Limiting
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: customer-service
          uri: lb://customer-service
          predicates:
            - Path=/api/v1/customers/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
```

### Request/Response Transformation
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: customer-service
          uri: lb://customer-service
          predicates:
            - Path=/api/v1/customers/**
          filters:
            - AddRequestHeader=X-Gateway-Version, 1.0
            - AddResponseHeader=X-Processed-By, Gateway
```

## 📁 Структура проекта

```
gateway/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── kg/manurov/gateway/
│   │   │       └── GatewayApplication.java
│   │   └── resources/
│   │       └── application.yml
│   └── test/
├── target/
├── pom.xml
└── README.md
```

## 🔗 Интеграция с микросервисами

### Автоматическое обнаружение сервисов
Gateway автоматически создает маршруты для всех сервисов, зарегистрированных в Eureka:

- `customer-service` → `/customer-service/**`
- `product-service` → `/product-service/**`
- `order-service` → `/order-service/**`
- `payment-service` → `/payment-service/**`

### Использование клиентами
Клиентские приложения могут обращаться ко всем API через единый endpoint:

```javascript
// Frontend JavaScript
const API_BASE = 'http://localhost:8888';

// Customers API
fetch(`${API_BASE}/api/v1/customers`)

// Products API  
fetch(`${API_BASE}/api/v1/products`)

// Orders API
fetch(`${API_BASE}/api/v1/orders`)
```