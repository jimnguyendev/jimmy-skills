# Architecture Patterns

## Choose the Right Level of Architecture

Architecture complexity MUST match project scope — don't over-architect small projects. When starting a new project, ask the developer what architecture they prefer:

| Project Size | Recommended Approach |
| --- | --- |
| Script / small CLI (<500 lines) | Flat `main.go` + a few files, no layers |
| Medium service (500-5K lines) | Feature-first packages, minimal abstractions, split only when pain appears |
| Large service / monolith (5K+ lines) | Feature-first bounded areas, then optionally clean architecture, hexagonal, or DDD where complexity justifies it |

A 100-line CLI does not need a domain layer, ports and adapters, or dependency injection frameworks. Start simple and refactor when complexity demands it.

For APIs, the default is still locality by business capability. Architectural patterns are overlays for specific complexity, not the starting folder tree.

## Keep Domain Pure

Domain logic MUST remain pure — no framework or infrastructure dependencies. The domain layer contains business logic and types:

```go
// domain/order.go — pure business logic, no imports from infrastructure
package domain

type Order struct {
    ID     string
    Items  []Item
    Status OrderStatus
}

func (o *Order) AddItem(item Item) error {
    if o.Status != StatusDraft {
        return ErrOrderNotEditable
    }
    o.Items = append(o.Items, item)
    return nil
}
```

Infrastructure concerns (database queries, HTTP clients, message queues) live in separate packages that depend on the domain — never the reverse. Keep these boundaries inside a feature area when possible; avoid scattering one feature across multiple top-level technical folders.

## Two-Layer Validation

Validation splits into two distinct layers, each with a clear responsibility. Do NOT mix them — and do NOT duplicate the same check in both layers.

### Layer 1: Syntactic Validation (Handler)

The handler validates that the request **can be processed** — correct format, required fields present, values within basic bounds. This catches malformed input before it reaches business logic.

**Responsibility:** Can I read this? Is the shape correct? Are required fields present?

```go
// internal/booking/handler.go — syntactic checks only
func (h *Handler) BookHomestay(w http.ResponseWriter, r *http.Request) {
    var req BookingRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "invalid JSON")
        return
    }
    if req.HomestayID == "" {
        respondError(w, http.StatusBadRequest, "homestay_id is required")
        return
    }
    if req.CheckinDate.IsZero() || req.CheckoutDate.IsZero() {
        respondError(w, http.StatusBadRequest, "checkin_date and checkout_date are required")
        return
    }
    if req.Guests <= 0 {
        respondError(w, http.StatusBadRequest, "guests must be positive")
        return
    }

    // All params structurally valid — hand off to service
    booking, err := h.service.Book(r.Context(), req)
    if errors.Is(err, ErrHomestayNotFound) {
        respondError(w, http.StatusNotFound, "homestay not found")
        return
    }
    if errors.Is(err, ErrHomestayNotActive) {
        respondError(w, http.StatusNotFound, "homestay not active")
        return
    }
    if errors.Is(err, ErrHomestayBusy) {
        respondError(w, http.StatusConflict, "homestay not available for selected dates")
        return
    }
    if err != nil {
        logger.Error("book homestay failed", zap.Error(err))
        respondError(w, http.StatusInternalServerError, "internal error")
        return
    }
    respondJSON(w, http.StatusCreated, booking)
}
```

### Layer 2: Semantic Validation (Service)

The service validates **business rules** that require domain knowledge, entity state, or database queries. These checks cannot be done with format validation alone.

**Responsibility:** Is this operation allowed by business rules? Does the entity exist and is it in the right state?

```go
// internal/booking/service.go — business rule checks
func (s *Service) Book(ctx context.Context, req BookingRequest) (*Booking, error) {
    // Rule: check-in cannot be in the past
    if req.CheckinDate.Before(time.Now().Truncate(24 * time.Hour)) {
        return nil, ErrCheckinDateInvalid
    }
    // Rule: cannot book more than 365 nights ahead
    nights := int(req.CheckoutDate.Sub(req.CheckinDate).Hours() / 24)
    if nights > 365 {
        return nil, ErrNightsExceeded
    }

    // Rule: homestay must exist and be active
    homestay, err := s.homestayRepo.GetByID(ctx, req.HomestayID)
    if err != nil {
        return nil, err // includes ErrHomestayNotFound
    }
    if !homestay.IsActive {
        return nil, ErrHomestayNotActive
    }
    if homestay.MaxGuests < req.Guests {
        return nil, ErrGuestsExceeded
    }

    // Rule: all dates must be available (with pessimistic lock)
    available, err := s.availRepo.FindAndLockAvailable(ctx, req.HomestayID, req.CheckinDate, req.CheckoutDate)
    if err != nil {
        return nil, fmt.Errorf("checking availability: %w", err)
    }
    if len(available) < nights {
        return nil, ErrHomestayBusy
    }

    // All rules passed — proceed with booking
    // ...
}
```

### What Goes Where — Decision Table

| Check | Layer | Why |
| --- | --- | --- |
| JSON parseable, required fields present | Handler | Reject malformed requests before touching business logic |
| Field format (email regex, date format, string length) | Handler | Pure format — no domain knowledge needed |
| Numeric bounds (guests > 0, price >= 0) | Handler | Basic sanity — no DB query needed |
| Date logic (check-in not in past, not too far ahead) | Service | Business rule — "too far ahead" is a domain policy |
| Entity exists and is in correct state | Service | Requires DB query |
| Cross-entity constraints (availability, capacity) | Service | Requires querying multiple records |
| Uniqueness (duplicate booking) | Service/Repository | Requires DB query or unique constraint |

### Anti-patterns to Avoid

**Do NOT duplicate the same check in both layers.** If the handler already validated `guests > 0`, the service should not check it again. Each check lives in exactly one place.

```go
// ❌ Bad — redundant check
// handler.go
if req.Guests <= 0 { return error }
// service.go
if req.Guests <= 0 { return error }  // never fires — handler already caught it

// ✅ Good — each layer owns its checks, no overlap
// handler.go: guests > 0 (format)
// service.go: guests <= homestay.MaxGuests (business rule)
```

**Do NOT put DB-dependent checks in the handler.** The handler should not import or query repositories — it delegates to the service.

**Do NOT put format checks in the service.** If the service is checking whether a string is empty or a number is positive, that check belongs in the handler.

## Make Illegal States Unrepresentable

Use Go's type system to prevent invalid states from being expressible in code:

```go
// Bad — status is a raw string, anything goes
type Order struct {
    Status string // "pending"? "PENDING"? "active"? anything?
}

// Good — typed enum constrains the values
type OrderStatus int

const (
    OrderStatusUnknown   OrderStatus = iota // 0 = invalid
    OrderStatusDraft                        // 1
    OrderStatusConfirmed                    // 2
    OrderStatusShipped                      // 3
)

type Order struct {
    Status OrderStatus
}
```

```go
// Bad — email is a raw string, could be anything
func SendEmail(to string, body string) error { ... }

// Good — validated type enforces the constraint
type Email struct {
    address string // unexported: can only be created via constructor
}

func NewEmail(raw string) (Email, error) {
    if !isValidEmail(raw) {
        return Email{}, fmt.Errorf("invalid email: %s", raw)
    }
    return Email{address: raw}, nil
}
```

## Breaking Circular Dependencies

Go enforces a strict rule: package imports must form a directed acyclic graph (DAG). If package A imports B, B cannot import A. The compiler rejects cycles outright.

**Why Go prohibits cycles** (Rob Pike): "Import cycles can be convenient but their cost can be catastrophic."

- **Compilation speed** — the compiler navigates dependency graphs more efficiently without cycles
- **Better architecture** — cycles signal tight coupling; preventing them forces cleaner design
- **Simpler maintenance** — non-circular structures make versioning and updates straightforward

### Solution 1: Separate Concerns Properly

When two packages form a cycle, one of them is handling a responsibility that belongs elsewhere:

```go
// ❌ Cycle: product imports inventory, inventory imports product
// package product
func (p *Product) IsAvailable() bool {
    return inventory.CheckStock(p.ID) > 0  // product → inventory
}

// package inventory
func CheckStock(productID string) int {
    p := product.Get(productID)  // inventory → product — CYCLE!
    return p.StockCount
}

// ✅ Fix: stock checking belongs in inventory, not product
// package inventory — accepts product ID, no need to import product
func CheckStock(productID string) int {
    return getStockCount(productID)
}
```

### Solution 2: Merge Inseparable Packages

When two packages are deeply intertwined with no clean split, consolidate them:

```go
// ❌ product/ and inventory/ are so coupled they import each other
// ✅ Merge into a single package that handles both
// package goods
type Product struct { ... }
type Stock struct { ... }
func (p *Product) IsAvailable() bool { ... }
```

Merging is often better than preserving a fake boundary that creates two-way knowledge and poor locality.

### Solution 3: Dependency Injection via Interfaces

Define interfaces at the consumer side. Wire concrete implementations in `main`:

```go
// package invoice — defines what it needs, does NOT import user
type UserLookup interface {
    GetName(ctx context.Context, id string) (string, error)
}

type Service struct {
    users UserLookup
}

// package user — implements the interface without knowing about invoice
type Service struct { db *sql.DB }
func (s *Service) GetName(ctx context.Context, id string) (string, error) { ... }

// cmd/api/main.go — wires concrete implementations
userSvc := user.NewService(db)
invoiceSvc := invoice.NewService(userSvc) // user.Service satisfies invoice.UserLookup
```

### Prevention Strategies

- **Feature-first layout** — packages organized by business capability naturally avoid cycles
- **One-way dependency direction** — higher layers depend on lower layers, never the reverse
- **Consumer-side interfaces** — depend on abstractions you define, not concrete packages
- **Small, focused packages** — the smaller the surface area, the fewer cross-dependencies

## Detailed Architecture Guides

For projects that warrant a formal architecture (typically 5K+ lines), see the dedicated guides:

- [Domain-Driven Design (DDD)](./ddd.md) — aggregates, value objects, bounded contexts
- [Clean Architecture](./clean-architecture.md) — use cases, dependency rule, layered adapters
- [Hexagonal Architecture](./hexagonal-architecture.md) — ports, adapters, domain core isolation

## 12-Factor App Principles

→ See `jimmy-skills@backend-go-project-layout` for 12-Factor App conventions.

## Explicit Over Implicit

Go favors explicitness. Code should express its intent clearly without requiring the reader to know hidden conventions:

```go
// Bad — implicit behavior hidden in struct tags and reflection
type Config struct {
    Port int `default:"8080"`
}

// Good — explicit defaults visible in code
func NewConfig() Config {
    return Config{Port: 8080}
}
```

```go
// Bad — implicit dependency via global
func HandleRequest(w http.ResponseWriter, r *http.Request) {
    user := globalDB.FindUser(r.Context(), userID) // where does globalDB come from?
}

// Good — explicit dependency via injection
func (h *Handler) HandleRequest(w http.ResponseWriter, r *http.Request) {
    user := h.db.FindUser(r.Context(), userID) // clear: db is a field on Handler
}
```

→ See `jimmy-skills@backend-go-project-layout` skill for directory structure and layout patterns.
