# Mapper Protocol

## Purpose

Build a lightweight structural map of the codebase that tells the reviewers
what to review, in what chunks, and with what boundary context. The mapper
never reads business logic — only structure, imports, and interface shapes.

## Step 1: Discover Source Files

Identify all source files in the project. Respect `.gitignore` and skip
generated/vendored directories.

```bash
# Discover files (adjust extensions for the project's stack)
find . -type f \( -name "*.py" -o -name "*.ts" -o -name "*.tsx" \
  -o -name "*.js" -o -name "*.jsx" -o -name "*.go" -o -name "*.rs" \
  -o -name "*.java" -o -name "*.rb" -o -name "*.swift" \) \
  ! -path "*/node_modules/*" \
  ! -path "*/.git/*" \
  ! -path "*/dist/*" \
  ! -path "*/build/*" \
  ! -path "*/__pycache__/*" \
  ! -path "*/vendor/*" \
  ! -path "*/.next/*" \
  | sort
```

Also discover config and schema files — these are often review-critical:
```bash
find . -type f \( -name "*.json" -o -name "*.yaml" -o -name "*.yml" \
  -o -name "*.toml" -o -name "*.env*" -o -name "Dockerfile*" \
  -o -name "*.sql" -o -name "*.prisma" -o -name "*.graphql" \) \
  ! -path "*/node_modules/*" ! -path "*/.git/*" | sort
```

Record the total file count. This determines the adaptation level.

## Step 2: Extract Import Graph

For each source file, extract only the import/require lines. Do not read
the rest of the file.

**Python:**
```bash
head -50 "$file" | grep -E "^(import |from .+ import )"
```

**TypeScript/JavaScript:**
```bash
head -60 "$file" | grep -E "^(import |const .+ = require\(|export .+ from )"
```

**Go:**
```bash
sed -n '/^import/,/^)/p' "$file"
```

**Rust:**
```bash
head -40 "$file" | grep -E "^(use |mod |extern crate )"
```

**Swift:**
```bash
head -30 "$file" | grep -E "^import "
```

For each file, produce a record:
```json
{
  "file": "src/services/orders.py",
  "imports": [
    "src/services/payment.py",
    "src/models/order.py",
    "src/db/connection.py",
    "external:stripe",
    "external:redis"
  ]
}
```

Classify imports as **internal** (resolves to a project file) or **external**
(third-party package). Only internal imports matter for the dependency graph.
External imports matter for the boundary brief (they indicate integration points).

## Step 3: Build Adjacency List

From the import records, construct a directed graph:
- **Nodes:** source files
- **Edges:** file A imports file B → edge from A to B

This does not need to be a formal graph data structure. A dictionary
mapping each file to its list of dependencies is sufficient:

```json
{
  "src/services/orders.py": ["src/services/payment.py", "src/models/order.py", "src/db/connection.py"],
  "src/services/payment.py": ["src/db/connection.py", "src/models/payment.py"],
  "src/routes/orders.py": ["src/services/orders.py", "src/middleware/auth.py"]
}
```

## Step 4: Identify Module Boundaries

Group files into chunks based on directory structure and internal cohesion.

**Default strategy:** Use the first meaningful directory level as the boundary.

For a project like:
```
src/
  routes/
  services/
  models/
  db/
  middleware/
  utils/
```

Each top-level directory under `src/` becomes a chunk. Files in `src/` root
(if any) become a "core" chunk.

**When to go deeper:** If a single directory has >50 files, split along its
subdirectories. For example, if `src/services/` contains `orders/`,
`payments/`, `inventory/`, each subdirectory becomes its own chunk.

**When to merge up:** If a directory has <5 files and is tightly coupled to
another directory (>80% of its imports come from one other module), merge it
into that module's chunk.

**Monorepo detection:** If the project root contains multiple independent
packages (e.g., `packages/api/`, `packages/web/`, `packages/shared/`), each
package is a top-level chunk. Then apply the above rules within each package.

## Step 5: Detect Cross-Cutting Files

Calculate fan-in for each file (how many other files import it).

Files with fan-in > 30% of total file count are **cross-cutting**. Common
examples: auth middleware, database connection, config, logger, type
definitions, shared utilities.

Cross-cutting files are included in EVERY chunk's boundary brief but are
NOT assigned to any single chunk for review. Instead, they form a special
**shared context** chunk that gets its own Tier 1 review (because high
fan-in means high blast radius).

## Step 6: Extract Interface Signatures

For each module's exported/public files, extract only the interface shapes.
Read function signatures, class definitions, type exports, and constant
declarations. Do NOT read function bodies.

**Python:**
```bash
grep -E "^(def |async def |class |[A-Z_]+ = )" "$file"
```

Plus type hints from stub files (`.pyi`) if they exist.

**TypeScript:**
```bash
grep -E "^export (function|const|class|interface|type|enum)" "$file"
```

**Go:**
```bash
grep -E "^func |^type .+ (struct|interface)" "$file"
```

For each chunk, compile a **boundary brief**:

```markdown
## Module: services/orders
### Exposes
- `create_order(user_id: str, items: List[OrderItem]) -> Order`
- `get_order(order_id: str) -> Order | None`
- `cancel_order(order_id: str) -> bool`

### Consumes
- `PaymentService.charge(amount: int, currency: str) -> ChargeResult`
  (from services/payment)
- `db.get_connection() -> AsyncConnection`
  (from db/connection — shared)
- `verify_token(token: str) -> UserClaims`
  (from middleware/auth — shared)

### External Integrations
- stripe (payment processing)
- redis (caching)

### Data Touched
- Table: orders (writes)
- Table: order_items (writes)
- Cache key: order:{id} (read/write)
```

## Step 7: Produce review-plan.json

Assemble everything into a structured review plan:

```json
{
  "project": "<project-name>",
  "total_files": 247,
  "total_chunks": 8,
  "adaptation_level": "medium",
  "language": "python",
  "framework_hints": ["fastapi", "sqlalchemy", "redis"],
  "shared_context": {
    "files": ["src/middleware/auth.py", "src/db/connection.py", "src/config.py"],
    "interfaces": "<compiled boundary brief for shared files>"
  },
  "chunks": [
    {
      "name": "services/orders",
      "files": ["src/services/orders.py", "src/services/order_utils.py"],
      "file_count": 2,
      "boundary_brief": "<the boundary brief from Step 6>",
      "complexity_score": 6,
      "review_priority": "high",
      "priority_reason": "writes to database, calls external payment service"
    }
  ],
  "detected_patterns": {
    "has_auth": true,
    "has_database": true,
    "has_external_apis": true,
    "has_event_system": false,
    "has_background_jobs": false,
    "has_file_uploads": false,
    "has_websockets": false
  }
}
```

The `complexity_score` is calculated as:
`file_count + (boundary_count × 2) + (external_integration_count × 3)`

Chunks are sorted by complexity score descending. High-complexity chunks
get reviewed first because they're most likely to contain issues.

`review_priority` flags:
- **critical**: touches auth, secrets, payments, or data deletion
- **high**: writes to database, calls external services
- **medium**: internal logic with moderate boundary surface
- **low**: utilities, helpers, types, constants

## Context Budget Check

Before moving to Phase 2, estimate total context cost:

```
mapper_cost = file_count × 3 (avg lines per import extract)
              + chunk_count × 40 (avg boundary brief lines)
              + shared_context × 60 (interface signatures)
```

If `mapper_cost > 10,000 lines`, increase chunk granularity by merging
small modules or grouping into domain-level chunks.

Report the plan to the architect before proceeding:

> "Mapped <N> files into <M> chunks. Highest priority: <chunk> (reason).
> Shared context: <list>. Ready to review, or want to adjust the plan?"

Wait for confirmation before starting Phase 2.
