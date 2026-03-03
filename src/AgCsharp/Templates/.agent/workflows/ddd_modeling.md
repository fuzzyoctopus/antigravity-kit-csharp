---
description: A comprehensive workflow for modeling domains using Domain-Driven Design principles. Guides through Event Storming discovery sessions (identifying domain events, commands, and aggregates), defining bounded contexts with context maps, designing aggregates in detail (root entities, child entities, value objects, domain events, invariants), defining repository interfaces and domain services, creating application layer commands and queries with DTOs, and implementation order with continuous refinement. Includes strategic and tactical design checklists and an event storming legend.
---

# Workflow: DDD Modeling

## Overview
Process for modeling a domain using Domain-Driven Design principles.

---

## Step 1: Event Storming (Discovery)

### Gather Stakeholders
- Domain experts
- Developers
- Product owners

### Identify Domain Events (Orange)
Events are things that happened in the past that the business cares about.

```
Examples:
- OrderPlaced
- PaymentReceived
- OrderShipped
- CustomerRegistered
- ProductAddedToCatalog
```

### Identify Commands (Blue)
Commands trigger events. They represent user intentions.

```
Examples:
- PlaceOrder
- ProcessPayment
- ShipOrder
- RegisterCustomer
```

### Identify Aggregates (Yellow)
Group related events and commands around aggregates.

```
Order Aggregate:
  Commands: PlaceOrder, AddItem, RemoveItem, SubmitOrder, CancelOrder
  Events: OrderCreated, ItemAdded, ItemRemoved, OrderSubmitted, OrderCancelled

Customer Aggregate:
  Commands: RegisterCustomer, UpdateProfile, DeactivateAccount
  Events: CustomerRegistered, ProfileUpdated, AccountDeactivated
```

---

## Step 2: Identify Bounded Contexts

### Group Related Aggregates
```
┌─────────────────────┐  ┌─────────────────────┐
│   Sales Context     │  │  Shipping Context   │
├─────────────────────┤  ├─────────────────────┤
│ - Order             │  │ - Shipment          │
│ - OrderItem         │  │ - Package           │
│ - Customer (view)   │  │ - Address           │
└─────────────────────┘  └─────────────────────┘

┌─────────────────────┐  ┌─────────────────────┐
│  Catalog Context    │  │  Billing Context    │
├─────────────────────┤  ├─────────────────────┤
│ - Product           │  │ - Invoice           │
│ - Category          │  │ - Payment           │
│ - Inventory         │  │ - Transaction       │
└─────────────────────┘  └─────────────────────┘
```

### Define Context Map
```
Sales Context <--[Upstream]--> Catalog Context
       |
       | [Publishes OrderPlaced]
       v
Shipping Context
       |
       | [Consumes from Sales]
       v
Billing Context
```

---

## Step 3: Define Aggregates in Detail

### For Each Aggregate, Define:

#### 1. Aggregate Root
```csharp
public class Order : AggregateRoot
{
    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money TotalAmount { get; private set; }
    
    private readonly List<OrderItem> _items = [];
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
}
```

#### 2. Child Entities
```csharp
public class OrderItem
{
    public Guid Id { get; private set; }
    public Guid ProductId { get; private set; }
    public string ProductName { get; private set; }
    public Money UnitPrice { get; private set; }
    public int Quantity { get; private set; }
}
```

#### 3. Value Objects
```csharp
public class Money : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }
}

public class Address : ValueObject
{
    public string Street { get; }
    public string City { get; }
    public string PostalCode { get; }
    public string Country { get; }
}
```

#### 4. Domain Events
```csharp
public record OrderSubmittedEvent(Guid OrderId, Guid CustomerId, Money Total);
public record OrderCancelledEvent(Guid OrderId, string Reason);
```

#### 5. Invariants (Business Rules)
```csharp
// Invariants enforced in Order aggregate:
// - Cannot add items to non-draft order
// - Cannot submit empty order
// - Cannot cancel shipped order
// - Total must equal sum of items
```

---

## Step 4: Define Repository Interfaces

```csharp
// One repository per aggregate root
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<Order?> GetByIdWithItemsAsync(Guid id, CancellationToken ct = default);
    Task AddAsync(Order order, CancellationToken ct = default);
    void Update(Order order);
    void Remove(Order order);
}
```

---

## Step 5: Define Domain Services

For operations that don't belong to a single aggregate:

```csharp
public class PricingService
{
    public Money CalculateOrderTotal(Order order, Customer customer)
    {
        var subtotal = order.Items.Sum(i => i.LineTotal);
        var discount = GetCustomerDiscount(customer);
        var tax = CalculateTax(subtotal, order.ShippingAddress);
        
        return subtotal.Subtract(discount).Add(tax);
    }
}

public class TransferService
{
    public void Transfer(Account source, Account destination, Money amount)
    {
        source.Debit(amount);
        destination.Credit(amount);
    }
}
```

---

## Step 6: Define Application Layer

### Commands and Queries
```csharp
// Commands (write operations)
public record CreateOrderCommand(Guid CustomerId) : IRequest<Guid>;
public record AddItemCommand(Guid OrderId, Guid ProductId, int Quantity) : IRequest;
public record SubmitOrderCommand(Guid OrderId) : IRequest;

// Queries (read operations)
public record GetOrderQuery(Guid Id) : IRequest<OrderDto?>;
public record GetCustomerOrdersQuery(Guid CustomerId) : IRequest<IEnumerable<OrderDto>>;
```

### DTOs (Data Transfer Objects)
```csharp
public record OrderDto(
    Guid Id,
    Guid CustomerId,
    string Status,
    decimal TotalAmount,
    IReadOnlyList<OrderItemDto> Items);
```

---

## Step 7: Implement and Iterate

### Implementation Order
1. Domain entities with business logic
2. Repository interfaces
3. EF Core configurations
4. Repository implementations
5. Application handlers
6. UI Forms and Presenters
7. Integration events

### Continuous Refinement
- Review with domain experts
- Update ubiquitous language
- Refine aggregate boundaries
- Extract or merge contexts as needed

---

## DDD Checklist

### Strategic Design
- [ ] Bounded contexts identified
- [ ] Context map created
- [ ] Ubiquitous language defined
- [ ] Core vs supporting domains classified

### Tactical Design
- [ ] Aggregates designed with roots
- [ ] Value objects extracted
- [ ] Domain events defined
- [ ] Repository per aggregate
- [ ] Domain services for cross-aggregate logic

### Implementation
- [ ] Rich domain model (not anemic)
- [ ] Invariants enforced in aggregates
- [ ] Factory methods for creation
- [ ] Private setters for encapsulation
- [ ] Domain events raised on state changes

---

## Event Storming Legend

| Color | Element | Description |
|-------|---------|-------------|
| 🟠 Orange | Domain Event | Something that happened |
| 🔵 Blue | Command | User intention |
| 🟡 Yellow | Aggregate | Cluster of entities |
| 🟣 Purple | Policy | "When X happens, do Y" |
| 🟢 Green | Read Model | Query/View |
| 🔴 Red | Hot Spot | Problem/Question |
| 🩷 Pink | External System | Third-party service |
