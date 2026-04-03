# Clean Architecture in Go

## When to Use

Apply clean architecture when you need strong separation between business logic and infrastructure — typically medium-to-large services (2K+ lines) where testability, framework independence, and clear dependency direction matter. Do NOT use for small CLI tools or scripts.

Clean architecture is not the default folder tree for every API. Start feature-first first; introduce this structure only when the domain and infrastructure boundaries are actually causing pain.

## The Dependency Rule

Dependencies point inward only. Inner layers never import outer layers.

```
Frameworks & Drivers  →  Interface Adapters  →  Use Cases  →  Entities
(HTTP, DB, gRPC)         (handlers, repos)      (app logic)   (domain)
```

Each layer defines interfaces for what it needs. Outer layers implement those interfaces.

## Project Structure

Keep feature locality even when using clean architecture. Prefer a vertical slice per capability over one global `handler/`, `repository/`, `usecase/` tree.

```
order-service/
├── cmd/
│   └── server/
│       └── main.go                  # Wiring only — builds the dependency graph
├── internal/
│   ├── order/
│   │   ├── entity.go                # Order entity + business rules
│   │   ├── port.go                  # Interfaces this use case depends on
│   │   ├── place.go                 # PlaceOrderUseCase
│   │   ├── cancel.go                # CancelOrderUseCase
│   │   ├── http_handler.go          # HTTP handler — calls use cases
│   │   ├── postgres_repo.go         # OrderRepository — implements port
│   │   └── payment_client.go        # External payment API client
│   └── platform/
│       ├── router.go               # HTTP router setup
│       ├── database.go             # DB connection
│       └── config.go               # Config loading
├── go.mod
└── go.sum
```

## Code Examples

### Entity — pure domain logic, zero dependencies

```go
// internal/entity/order.go
package entity

type Order struct {
    ID     string
    Items  []Item
    Status OrderStatus
}

func (o *Order) Cancel() error {
    if o.Status == StatusShipped {
        return ErrCannotCancelShipped
    }
    o.Status = StatusCancelled
    return nil
}

func (o *Order) Total() int64 {
    var sum int64
    for _, item := range o.Items {
        sum += item.Price * int64(item.Quantity)
    }
    return sum
}
```

### Use Case — orchestrates business operations

```go
// internal/order/port.go
package order

// Ports — interfaces defined by the use case, implemented by adapters
type OrderRepository interface {
    Save(ctx context.Context, order *entity.Order) error
    FindByID(ctx context.Context, id string) (*entity.Order, error)
}

type PaymentGateway interface {
    Charge(ctx context.Context, orderID string, amount int64) error
}
```

```go
// internal/order/place.go
package order

type PlaceOrderUseCase struct {
    orders   OrderRepository
    payments PaymentGateway
}

func NewPlaceOrderUseCase(orders OrderRepository, payments PaymentGateway) *PlaceOrderUseCase {
    return &PlaceOrderUseCase{orders: orders, payments: payments}
}

func (uc *PlaceOrderUseCase) Execute(ctx context.Context, orderID string) error {
    order, err := uc.orders.FindByID(ctx, orderID)
    if err != nil {
        return fmt.Errorf("finding order: %w", err)
    }

    if err := uc.payments.Charge(ctx, order.ID, order.Total()); err != nil {
        return fmt.Errorf("charging payment: %w", err)
    }

    order.Status = entity.StatusPlaced
    return uc.orders.Save(ctx, order)
}
```

### Adapter — implements a port

```go
// internal/order/postgres_repo.go
package repository

type OrderPostgres struct {
    db *sql.DB
}

func NewOrderPostgres(db *sql.DB) *OrderPostgres {
    return &OrderPostgres{db: db}
}

func (r *OrderPostgres) FindByID(ctx context.Context, id string) (*entity.Order, error) {
    // SQL query, scan into entity.Order
}

func (r *OrderPostgres) Save(ctx context.Context, order *entity.Order) error {
    // SQL upsert
}
```

### Handler — translates HTTP to use case calls

```go
// internal/order/http_handler.go
package handler

type OrderHandler struct {
    placeOrder *usecase.PlaceOrderUseCase
}

func (h *OrderHandler) HandlePlaceOrder(w http.ResponseWriter, r *http.Request) {
    orderID := chi.URLParam(r, "id")

    if err := h.placeOrder.Execute(r.Context(), orderID); err != nil {
        // Map domain errors to HTTP status codes
        http.Error(w, err.Error(), mapToHTTPStatus(err))
        return
    }

    w.WriteHeader(http.StatusOK)
}
```

## Key Principle

Interfaces live where they are consumed, not where they are implemented. The `internal/order/port.go` file defines `OrderRepository` — the concrete repo in `internal/order/postgres_repo.go` implements it. This keeps the use case layer free from infrastructure imports.

The more important rule is still locality: clean architecture should not force one order-related change to jump through five unrelated top-level directories.

## Wiring

All dependency construction happens in `cmd/server/main.go`.
