# Distributed SQL Query Planner

A federated SQL query engine that parses ANSI SQL, plans distributed execution across multiple PostgreSQL backends, and streams results using Volcano-style iterators — achieving sub-second latency on multi-table cross-shard joins.

```
                  ┌→ PostgreSQL A (users, regions, reviews)
Client → Planner ─┼→ PostgreSQL B (orders, products, payments)
                  └→ PostgreSQL C (inventory, warehouses, shipments)
```

## Architecture

```
SQL Query
   ↓
Parser (JSqlParser → Logical Plan)
   ↓
Planner (Partition-aware pushdown + cross-shard join planning)
   ↓
Physical Plan (Backend scans, hash joins, filters, projections)
   ↓
Execution Engine (Streaming iterators — Volcano model)
   ↓
┌────────┬────────┬────────┐
│ DB A   │ DB B   │ DB C   │
└────────┴────────┴────────┘
   ↓
Merge / Hash Join (incremental, pipelined)
   ↓
Final Result (+ partial failure metadata)
```

### Key Design Decisions

| Component | Approach | Rationale |
|-----------|----------|-----------|
| **Parser** | JSqlParser (ANSI SQL) | Battle-tested grammar; focus engineering on distributed execution, not lexing |
| **IR** | Sealed logical/physical plan trees | Type-safe plan representation with exhaustive pattern matching |
| **Planner** | Partition-aware pushdown | Co-located joins execute entirely on one backend; cross-shard joins use hash join |
| **Execution** | Volcano-style iterators | Rows flow incrementally — no full materialization of intermediate results |
| **Failure handling** | Configurable degradation modes | `PARTIAL_RESULTS`, `SKIP_BACKEND`, or `FAIL_FAST` with structured error metadata |
| **Observability** | Micrometer + Prometheus | Query latency histograms, backend health endpoints |

### Streaming Iterator Model

Instead of loading entire result sets into memory:

```
Backend A ─→ row ─→ row ─→ row ─→
                     ↓
                HashJoin operator
                     ↑
Backend B ─→ row ─→ row ─→ row ─→
```

Each operator implements a pull-based `RowIterator`:

```java
public interface RowIterator extends AutoCloseable {
    Schema schema();
    Row next();  // null when exhausted
}
```

This enables pipelined execution with bounded memory regardless of result set size.

### Partial Failure Semantics

When a backend becomes unavailable mid-query:

| Mode | Behavior |
|------|----------|
| `PARTIAL_RESULTS` | Return rows from healthy backends; annotate response with failure metadata |
| `SKIP_BACKEND` | Skip the failed shard entirely; continue with remaining backends |
| `FAIL_FAST` | Abort the entire query immediately |

The response includes structured failure details: backend ID, failure phase (connection/execution/streaming), and timestamp.

## Tech Stack

- **Backend:** Java 21, Spring Boot 3.3, JSqlParser, HikariCP, Micrometer
- **Frontend:** React 18, TypeScript, Vite
- **Database:** PostgreSQL 16 (3 shards via Docker Compose)
- **Testing:** JUnit 5, Testcontainers (integration tests against real PostgreSQL)

## Quick Start

### Prerequisites

- Java 21+
- Maven 3.9+
- Node.js 18+ (for frontend)
- Docker & Docker Compose

### 1. Start PostgreSQL shards

```bash
docker compose up -d postgres-a postgres-b postgres-c
```

### 2. Run the backend

```bash
cd backend
mvn spring-boot:run
```

### 3. Run the frontend (dev mode)

```bash
cd frontend
npm install
npm run dev
```

Open http://localhost:3000

### Full stack via Docker

```bash
cd backend && mvn package -DskipTests
cd .. && docker compose up --build
```

Open http://localhost:8080

## Example Queries

**Cross-shard join (2 backends):**
```sql
SELECT u.name, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
```

**10-table join (all 3 backends):**
```sql
SELECT u.name, r.name AS region, p.name AS product, c.name AS category,
       o.total, pay.method, inv.quantity, w.name AS warehouse, s.status
FROM users u
JOIN regions r ON u.region_id = r.id
JOIN orders o ON u.id = o.user_id
JOIN products p ON o.product_id = p.id
JOIN categories c ON p.category_id = c.id
JOIN payments pay ON o.id = pay.order_id
JOIN inventory inv ON p.id = inv.product_id
JOIN warehouses w ON inv.warehouse_id = w.id
JOIN shipments s ON o.id = s.order_id
JOIN reviews rev ON p.id = rev.product_id AND u.id = rev.user_id
```

## API

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/query` | POST | Execute SQL query |
| `/api/explain` | POST | Show logical + physical plan |
| `/api/catalog/tables` | GET | List cataloged tables and their backends |
| `/api/backends/health` | GET | Backend health status |
| `/actuator/prometheus` | GET | Prometheus metrics |

## Testing

```bash
cd backend
mvn test
```

- **Unit tests:** Parser, planner routing, hash join iterator
- **Integration tests:** Testcontainers with real PostgreSQL — cross-backend joins, partial failure modes

## Project Structure

```
distributed-sql-planner/
├── backend/
│   └── src/main/java/com/dsqp/
│       ├── parser/          # SQL → Logical Plan
│       ├── ir/              # Logical & Physical plan nodes
│       ├── planner/         # Logical → Physical plan transformation
│       ├── execution/       # Streaming iterator engine
│       │   ├── iterator/    # RowIterator implementations
│       │   └── backend/     # Connection pooling
│       ├── failure/         # Partial failure handling
│       ├── catalog/         # Table → backend routing
│       ├── api/             # REST controllers
│       └── service/         # Query orchestration
├── frontend/                # React query console
├── docker/                  # Shard initialization SQL
└── docker-compose.yml       # 3 PostgreSQL instances
```

## License

MIT
