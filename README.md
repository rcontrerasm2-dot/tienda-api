# Tienda API

Servicio web **API REST** para la gestión de **inventario y ventas** del *Minimercado La Económica*.
Trabajo Colaborativo Contextualizado de **Ingeniería de Software** — Universidad de Cartagena
(Grupo CIPA: **Mentes Digitales**, noviembre de 2026).

**Tecnologías:** Java 17 · Spring Boot 3 · Spring Data JPA (Hibernate) · PostgreSQL 16 · Maven · Docker / Docker Compose · JUnit 5 + Mockito · Postman.

## ¿Qué hace?

- CRUD de **categorías** y **productos** (código único, precio > 0, stock y stock mínimo).
- Registro de **ventas** con varios productos: calcula el total en el servidor, guarda el precio unitario de cada línea y **descuenta el stock** en una sola transacción (si algo falla, no se guarda nada).
- Consulta de ventas por **día** y por identificador.
- Alerta de **stock bajo** (productos con `stock <= stock_minimo`).
- Errores en JSON uniforme con el código HTTP adecuado (400, 404, 409, 500).

## Arquitectura

Arquitectura en capas: `controller` (HTTP/JSON) → `service` (reglas de negocio, `@Transactional`) → `repository` (Spring Data JPA) → PostgreSQL.
Entre controlador y servicio los datos viajan en DTO (`record` de Java); un `GlobalExceptionHandler` traduce las excepciones a respuestas HTTP.

```
tienda-api/
├── src/main/java/com/laeconomica/tienda/
│   ├── controller/   entity/   dto/   repository/   service/   exception/
│   └── TiendaApiApplication.java
├── src/main/resources/application.properties
├── src/test/java/com/laeconomica/tienda/service/   (pruebas unitarias PU-01 a PU-07)
├── postman/Tienda-La-Economica.postman_collection.json
├── Dockerfile
├── docker-compose.yml
├── pom.xml
└── README.md
```

## Ejecución con Docker (recomendada)

Requisitos: Docker y Docker Compose.

```bash
git clone https://github.com/<tu-usuario>/tienda-api.git
cd tienda-api
docker compose up --build
```

- API: <http://localhost:8080/api/v1>
- PostgreSQL: `localhost:5432` (base `tienda_db`)
- Detener: `docker compose down` (los datos se conservan en el volumen `pgdata`; para borrarlos: `docker compose down -v`).

Opcional: copia `.env.example` como `.env` para cambiar usuario y contraseña de la base de datos.

## Ejecución sin Docker

Requisitos: Java 17, Maven 3.9+ y un PostgreSQL con una base llamada `tienda_db`.

```bash
# Variables opcionales (por defecto: localhost:5432, usuario postgres, clave postgres)
export DB_URL=jdbc:postgresql://localhost:5432/tienda_db
export DB_USER=postgres
export DB_PASSWORD=postgres

mvn spring-boot:run
```

## Pruebas unitarias

```bash
mvn test
```

Las pruebas usan JUnit 5 y Mockito (no necesitan base de datos).

## Endpoints (`/api/v1`)

| Método | Ruta | Descripción | Código |
|---|---|---|---|
| POST | `/categorias` | Crea una categoría | 201 |
| GET | `/categorias` | Lista las categorías | 200 |
| GET | `/categorias/{id}` | Consulta una categoría | 200 |
| PUT | `/categorias/{id}` | Actualiza una categoría | 200 |
| DELETE | `/categorias/{id}` | Elimina una categoría (409 si tiene productos) | 204 |
| POST | `/productos` | Crea un producto | 201 |
| GET | `/productos` | Lista el inventario | 200 |
| GET | `/productos/stock-bajo` | Productos en alerta de stock | 200 |
| GET | `/productos/{id}` | Consulta un producto | 200 |
| PUT | `/productos/{id}` | Actualiza un producto | 200 |
| DELETE | `/productos/{id}` | Elimina un producto | 204 |
| POST | `/ventas` | Registra una venta y descuenta stock | 201 |
| GET | `/ventas?fecha=AAAA-MM-DD` | Lista las ventas de un día (hoy si no se envía) | 200 |
| GET | `/ventas/{id}` | Consulta una venta | 200 |

### Ejemplo rápido

```bash
# 1. Crear categoría
curl -X POST http://localhost:8080/api/v1/categorias \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Abarrotes","descripcion":"Arroz, azúcar, aceite"}'

# 2. Crear producto (usa el id de la categoría creada)
curl -X POST http://localhost:8080/api/v1/productos \
  -H "Content-Type: application/json" \
  -d '{"codigo":"P-001","nombre":"Arroz 500g","precio":2500,"stock":10,"stockMinimo":3,"categoriaId":1}'

# 3. Registrar una venta
curl -X POST http://localhost:8080/api/v1/ventas \
  -H "Content-Type: application/json" \
  -d '{"metodoPago":"EFECTIVO","items":[{"productoId":1,"cantidad":2}]}'

# 4. Ver productos con stock bajo
curl http://localhost:8080/api/v1/productos/stock-bajo
```

Ejemplo de error (stock insuficiente, HTTP 409):

```json
{
  "timestamp": "2026-11-20T10:35:02",
  "status": 409,
  "error": "Conflict",
  "mensaje": "Stock insuficiente para 'Arroz 500g'. Disponible: 3, solicitado: 5",
  "detalles": []
}
```

## Pruebas con Postman

Importa `postman/Tienda-La-Economica.postman_collection.json`. La colección usa la variable `baseUrl` (por defecto `http://localhost:8080/api/v1`) e incluye el flujo completo y los casos negativos PF-01 a PF-09.

## Reglas de negocio implementadas

RN-01 no vender más del stock disponible · RN-02 precio > 0 y stock ≥ 0 · RN-03 código de producto único · RN-04 todo producto pertenece a una categoría existente · RN-05 subtotal = precio × cantidad y total = suma de líneas · RN-06 una venta tiene al menos un producto y cantidades ≥ 1 · RN-07 alerta cuando stock ≤ stock mínimo · RN-08 no se elimina una categoría con productos.

## Limitaciones conocidas

Sin autenticación/autorización, sin control de concurrencia avanzado (dos ventas simultáneas del mismo producto), `ddl-auto=update` en lugar de migraciones (Flyway) y sin documentación OpenAPI. Se plantean como trabajo futuro en el documento del proyecto.

## Integrantes

Wilfran Sebastián Calderón Mejía · Edwar Daniel Caraballo Feria · Ronal José Contreras Moreno · Manuel David Gil Palomeque.
Tutor: Ranfis Enrique Ibarra Davila.
