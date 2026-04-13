<!--
CONFIG
FULL_NAME:Julian Andres Guerra Garcia
GITHUB_USER: julianguerra1231186@gmail.com
CODE_ORGANIZATION: code-corhuila
-->

![](https://github.com/julianguerra1231186-crypto/ms-products/blob/main/api-gateway/miroservicio2.png)
# ms-products — Microservicio de Catálogo de Productos

### Se deja evidencia de todas las Historias de Usuario en la mesa de trabajo:
   - [Mesa De Trabajo](https://julianguerra1231186-1773894024267.atlassian.net/?continue=https%3A%2F%2Fjulianguerra1231186-1773894024267.atlassian.net%2Fwelcome%2Fsoftware%3FprojectId%3D10000&atlOrigin=eyJpIjoiOTdhMWY4ZGU5N2YwNDQ0MDk3NTZjODkxYTU5ZWVlZWQiLCJwIjoiamlyYS1zb2Z0d2FyZSJ9)
     
## Descripción general

`ms-products` es el microservicio responsable del catálogo de pulpas de fruta de PulpApp. Gestiona los productos y sus categorías, expone una API REST consumida tanto por el frontend como por `ms-orders` cuando necesita consultar el precio real de un producto al momento de crear un pedido.

Es el único microservicio del sistema que no tiene Spring Security propio. La protección de sus rutas de escritura es delegada al token JWT que valida `ms-users` antes de que el request llegue aquí.

---

## Puerto

| Entorno | Puerto |
|---------|--------|
| Local | 8082 |
| Docker (interno) | 8082 |
| Acceso desde host | http://localhost:8082 |
| Acceso vía Gateway | http://localhost:8090/products |

---

## Stack tecnológico

| Tecnología | Versión | Propósito |
|-----------|---------|-----------|
| Java | 17 | Lenguaje base |
| Spring Boot | 4.0.3 | Framework principal |
| Spring Data JPA | Incluido | Persistencia con Hibernate |
| Spring Validation | Incluido | Validaciones con @Valid |
| PostgreSQL | 15 | Base de datos relacional |
| Liquibase | Incluido | Versionado del esquema de BD |
| Lombok | Incluido | Reducción de código boilerplate |

---

## Estructura de paquetes

```
com.pulpapp.msproducts/
│
├── config/
│   ├── CorsConfig.java
│   │     Habilita CORS para todos los orígenes, métodos y headers.
│   │     Permite que el frontend y ms-orders consuman la API sin bloqueos.
│   │
│   └── LiquibaseConfig.java
│         Configuración de Liquibase para el versionado del esquema.
│
├── controller/
│   └── ProductController.java
│         Expone el CRUD completo de productos en la ruta /products:
│         GET    /products      → lista todos los productos
│         GET    /products/{id} → obtiene un producto por ID
│         POST   /products      → crea un nuevo producto (ADMIN)
│         PUT    /products/{id} → actualiza un producto existente (ADMIN)
│         DELETE /products/{id} → elimina un producto (ADMIN)
│
├── service/
│   └── ProductService.java
│         Contiene toda la lógica de negocio del microservicio:
│
│         findAll()   → consulta todos los productos y los convierte a DTO
│         findById()  → busca por ID, lanza ResourceNotFoundException si no existe
│         create()    → valida nombre único, aplica trim() a los textos y persiste
│         update()    → valida nombre único excluyendo el propio producto, actualiza
│         delete()    → busca el producto y lo elimina
│
│         validateUniqueName() → verifica que no exista otro producto con el mismo
│         nombre (ignorando mayúsculas). En creación usa existsByNameIgnoreCase(),
│         en actualización usa existsByNameIgnoreCaseAndIdNot() para excluir
│         el producto que se está editando.
│
│         applyDtoToEntity() → aplica trim() a name, description e imageUrl
│         antes de persistir para evitar espacios innecesarios.
│
├── entity/
│   ├── Product.java
│   │     Entidad JPA mapeada a la tabla products.
│   │     Campos: id, name (unique), description, price, stock, available,
│   │     imageUrl, category (ManyToOne → Category, nullable).
│   │     Hibernate convierte imageUrl → image_url automáticamente (snake_case).
│   │
│   └── Category.java
│         Entidad JPA mapeada a la tabla category.
│         Campos: id, name (unique), description.
│         Relación OneToMany con Product (1 categoría → N productos).
│         Si se elimina una categoría, los productos quedan con category_id = NULL
│         gracias al onDelete: SET NULL configurado en Liquibase.
│
├── dto/
│   ├── ProductRequestDTO.java
│   │     Entrada para crear y actualizar productos.
│   │     Validaciones:
│   │       name        → @NotBlank, máx 120 caracteres
│   │       description → @NotBlank, máx 255 caracteres
│   │       price       → @NotNull, mayor que 0 (@DecimalMin)
│   │       stock       → @NotNull, mayor o igual a 0
│   │       available   → @NotNull
│   │       imageUrl    → opcional, máx 255 caracteres
│   │
│   └── ProductResponseDTO.java
│         Salida con todos los campos del producto:
│         {id, name, description, price, stock, available, imageUrl}
│
├── repository/
│   ├── ProductRepository.java
│   │     Extiende JpaRepository<Product, Long>.
│   │     Métodos adicionales:
│   │       existsByNameIgnoreCase(name)
│   │       existsByNameIgnoreCaseAndIdNot(name, id)
│   │     Spring Data JPA genera el SQL automáticamente a partir del nombre del método.
│   │
│   └── CategoryRepository.java
│         Extiende JpaRepository<Category, Long>. CRUD básico.
│         Disponible para futuras operaciones sobre categorías.
│
└── exception/
    ├── GlobalExceptionHandler.java
    │     @RestControllerAdvice que captura excepciones y las convierte en JSON.
    │     Maneja: ResourceNotFoundException (404), ResponseStatusException (409 para
    │     nombre duplicado), MethodArgumentNotValidException (400), Exception (500).
    │
    └── ResourceNotFoundException.java
          Excepción lanzada cuando no se encuentra un producto por ID.
```

---

## Liquibase — Versionado de base de datos

### ¿Qué es Liquibase?

Liquibase es una herramienta que gestiona los cambios en la estructura de la base de datos de forma controlada y reproducible. En lugar de ejecutar scripts SQL manualmente, se definen **changesets** en archivos YAML. Al arrancar la aplicación, Liquibase revisa cuáles changesets ya fueron aplicados (los guarda en la tabla `databasechangelog`) y ejecuta solo los nuevos.

La propiedad `onFail: MARK_RAN` hace que si una tabla ya existe, el changeset se registre como ejecutado sin lanzar error. Esto hace los changesets **idempotentes** — seguros para ejecutar múltiples veces.

### Changesets de ms-products

#### Changeset 1 — Tabla `category`

Crea la tabla que clasifica los productos. El nombre de categoría es único.

```
category
├── id          BIGINT PK autoincrement
├── name        VARCHAR(100) NOT NULL UNIQUE
└── description VARCHAR(255) nullable
```

#### Changeset 2 — Tabla `products`

Crea el catálogo de pulpas con todos sus campos base.

```
products
├── id          BIGINT PK autoincrement
├── name        VARCHAR(120) NOT NULL UNIQUE
├── description VARCHAR(255) NOT NULL
├── price       DOUBLE PRECISION NOT NULL
├── stock       INTEGER NOT NULL
├── available   BOOLEAN NOT NULL
└── image_url   VARCHAR(255) nullable
```

#### Changeset 3 — FK `products → category`

Agrega la columna `category_id` a `products` y crea la clave foránea.
`onDelete: SET NULL` significa que si se elimina una categoría, los productos
quedan con `category_id = NULL` en lugar de eliminarse.

#### Changeset 4 — Seed de categorías

Inserta las 4 categorías iniciales si la tabla está vacía. Usa `sqlCheck` para
verificar que `COUNT(*) = 0` antes de insertar, evitando duplicados.

| ID | Nombre | Descripción |
|----|--------|-------------|
| 1 | Tropicales | Mango, maracuyá, lulo |
| 2 | Cítricas | Naranja, limón |
| 3 | Berries | Fresa, mora |
| 4 | Exóticas | Frutas poco comunes |

---

## Modelo de datos

### Tabla `category`

| Columna | Tipo | Restricción |
|---------|------|-------------|
| id | BIGINT | PK, autoincrement |
| name | VARCHAR(100) | NOT NULL, UNIQUE |
| description | VARCHAR(255) | nullable |

### Tabla `products`

| Columna | Tipo | Restricción | Descripción |
|---------|------|-------------|-------------|
| id | BIGINT | PK, autoincrement | Identificador único |
| name | VARCHAR(120) | NOT NULL, UNIQUE | Nombre del producto |
| description | VARCHAR(255) | NOT NULL | Descripción |
| price | DOUBLE PRECISION | NOT NULL | Precio de venta |
| stock | INTEGER | NOT NULL | Unidades disponibles |
| available | BOOLEAN | NOT NULL | Si está activo en el catálogo |
| image_url | VARCHAR(255) | nullable | Ruta o URL de la imagen |
| category_id | BIGINT | FK → category (SET NULL) | Categoría del producto |

### Relación entre entidades

```
Category (1) ──────────────── (N) Product
    id                               id
    name                             name
    description                      description
                                     price
                                     stock
                                     available
                                     image_url
                                     category_id (FK, nullable)
```

---

## Endpoints con ejemplos

### GET /products — Listar todos

```json
// Response 200
[
  {
    "id": 1,
    "name": "Pulpa de Mango",
    "description": "Pulpa natural sin conservantes",
    "price": 8500.0,
    "stock": 100,
    "available": true,
    "imageUrl": "img/mango.png"
  }
]
```

### POST /products — Crear producto (requiere ROLE_ADMIN)

```json
// Request — Header: Authorization: Bearer <token_admin>
{
  "name": "Pulpa de Mora",
  "description": "Pulpa natural de mora seleccionada",
  "price": 7900.0,
  "stock": 50,
  "available": true,
  "imageUrl": "img/mora.png"
}

// Response 201
{
  "id": 2,
  "name": "Pulpa de Mora",
  "description": "Pulpa natural de mora seleccionada",
  "price": 7900.0,
  "stock": 50,
  "available": true,
  "imageUrl": "img/mora.png"
}
```

### Nombre duplicado — Error 409

```json
// Response 409 Conflict
{
  "error": "A product with that name already exists"
}
```

---

## Manejo de errores

| Excepción | HTTP | Cuándo ocurre |
|-----------|------|---------------|
| ResourceNotFoundException | 404 | Producto no encontrado por ID |
| ResponseStatusException (CONFLICT) | 409 | Nombre de producto duplicado |
| MethodArgumentNotValidException | 400 | Campos inválidos en el request |
| Exception (fallback) | 500 | Error interno no controlado |

---

## Configuración

```properties
spring.application.name=ms-products
server.port=${SERVER_PORT:8082}

spring.datasource.url=${SPRING_DATASOURCE_URL:jdbc:postgresql://localhost:5434/pulpapp_db}
spring.datasource.username=${SPRING_DATASOURCE_USERNAME:postgres}
spring.datasource.password=${SPRING_DATASOURCE_PASSWORD:1234}

# Liquibase gestiona el esquema; Hibernate no debe modificar tablas
spring.jpa.hibernate.ddl-auto=none
spring.jpa.show-sql=true

spring.liquibase.change-log=classpath:db/changelog/changelog-master.yml
spring.liquibase.enabled=true
```

---

## Levantar el servicio

```bash
docker-compose up --build ms-products
docker-compose logs -f ms-products
```

# Autor
Autor:
Julian Guerra
