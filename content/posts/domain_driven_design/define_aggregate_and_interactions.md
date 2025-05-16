+++
title = "REST Endpoint with Aggregate, Application Layer and Domain Layer"
series = ["Domain Driven Design"]
date = "2025-05-07"
+++

<!-- 
Amélioration à faire, en faire un autre article ?
https://claude.ai/share/64893062-f68a-4cdf-9330-6ac4a933ba1a -->

## Application to build
In the following article you will learn how to avoid building [Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html) and [Transaction Script](https://martinfowler.com/eaaCatalog/transactionScript.html) code when you have business logic in your application. We have
- `Order` has several `OrderLine`
- Each `OrderLine` contains the `Article` and the quantity bought
- Apply a discount if the client is good

And we want to create a new Order through endpoint API `POST /orders`
```json
{
  "clientId": 1,
  "linesOrder": [
    {
      "articleId": 1,
      "quantity": 2
    },
    ...
  ]
}
```

Then is the second part of the article we will add two use-cases :
- remove an `OrderLine` of an `Order`
- add the notion of tracking Article Movements



## Aggregate
> Aggregate is a pattern in Domain-Driven Design. A DDD aggregate is a cluster of domain objects that can be treated as a single unit. [^1]

In Domain-Driven Design (DDD), Aggregates are a core tactical building block. They help manage consistency boundaries and enforce invariants within a domain model. Based on your description, let’s walk through how Aggregates could be modeled for your case.

### Order Aggregate
Order aggregate contains both `Order` and `OrderLine` object
- All LineOrder instances are owned by the Order.
- You likely want to enforce rules like: "An Order cannot exceed 10 LineOrders", "An Order’s total must not exceed a limit", etc.
- You always manipulate LineOrders through the Order.

=> Aggregate root `Order`

### Article Aggregate
- Articles are referenced by LineOrders but not owned by them.
- Articles are likely updated independently (e.g. inventory, name changes).

=> Aggregate Root `Article`

### Guiding Principles
- Only access aggregate roots directly from repositories.
- Cross-aggregate consistency must be handled eventually or via domain events.
- Keep aggregates small enough to fit in memory and be saved atomically.

## 1. Create a new Order
Now, we want to handle the creation of an Order aggregate based on input JSON. Here's a full breakdown of how to structure this in line with DDD principles.

### Domain Model Entities
```java
class Order {
    private Long id;
    private Long clientId;
    private List<LineOrder> lineOrders = new ArrayList<>();
    private BigDecimal total;

    public Order(Long clientId) {
        this.clientId = clientId;
        this.total = BigDecimal.ZERO;
    }

    public void addLineOrder(LineOrder lineOrder) {
        lineOrders.add(lineOrder);
        total = total.add(lineOrder.getLineTotal());
    }

    public void applyDiscount(BigDecimal discountRate) {
        total = total.subtract(total.multiply(discountRate));
    }
}
```

```java
class LineOrder {
    private Long articleId;
    private int quantity;
    private BigDecimal unitPrice;

    public LineOrder(Long articleId, int quantity, BigDecimal unitPrice) {
        this.articleId = articleId;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
    }

    public BigDecimal getLineTotal() {
        return unitPrice.multiply(BigDecimal.valueOf(quantity));
    }
}
```

### Service Layer (Application Service)
The Application Service is orchestrating the use case, not containing domain logic.

```java
@Service
public class OrderApplicationService {

    private final OrderRepository orderRepository;
    private final ArticleRepository articleRepository;
    private final ClientRepository clientRepository;

    public OrderApplicationService(OrderRepository orderRepository,
                                   ArticleRepository articleRepository,
                                   ClientRepository clientRepository) {
        this.orderRepository = orderRepository;
        this.articleRepository = articleRepository;
        this.clientRepository = clientRepository;
    }

    @Transactional
    public Long createOrder(CreateOrderRequest request) {
        // Step 1: Create Order
        Order order = new Order(request.getClientId());

        // Step 2: Add LineOrders
        for (CreateLineOrderDTO lineDTO : request.getLineOrders()) {
            Article article = articleRepository.findById(lineDTO.getArticleId())
                    .orElseThrow(() -> new RuntimeException("Article not found"));

            LineOrder lineOrder = new LineOrder(
                article.getId(),
                lineDTO.getQuantity(),
                article.getPrice()
            );

            order.addLineOrder(lineOrder);
        }

        // Step 3: Apply discount if client is good
        Client client = clientRepository.findById(request.getClientId())
                .orElseThrow(() -> new RuntimeException("Client not found"));

        if (client.isGoodClient()) {
            order.applyDiscount(BigDecimal.valueOf(0.10)); // 10%
        }

        // Step 4: Save Order, only Aggregate root persisted !
        orderRepository.save(order);

        return order.getId();
    }
}
```

The application service layer
- have access to the repository, **only aggregate roots are saved**
- orchestre *application logic* : get data, create object, persist data
  - but not handle domain logic like calculate the total (managed by `Order`)

## 2. Delete lineOrder
How develop the new feature to remove a lineOrder to an Order ? `DELETE orders/{orderId}/lines/{lineOrderId}`

### Application Service
```java
@Transactional
public void deleteLineOrder(Long orderId, Long lineOrderId) {
    // Step 1: Load Order Aggregate
    Order order = orderRepository.findById(orderId)
        .orElseThrow(() -> new RuntimeException("Order not found"));

    // Step 2: Modify Aggregate
    order.removeLineOrder(lineOrderId);

    // Step 3: Save Order, only access to Aggregate Root
    orderRepository.save(order); // assumes aggregate pattern
}
```

No change is needed in your repositories unless you're persisting LineOrder separately. In typical aggregate design, it's embedded and saved via `Order`.
Like `Order` is the aggregate root we have to go through it to modify the order (add or remove line).

### Domain Logic in Aggregate Root (Order class)
Add this method in your Order class
```java
public class Order {
    public void removeLineOrder(Long lineOrderId) {
    LineOrder toRemove = lineOrders.stream()
        .filter(line -> line.getId().equals(lineOrderId))
        .findFirst()
        .orElseThrow(() -> new RuntimeException("LineOrder not found"));

    total = total.subtract(toRemove.getLineTotal());
    lineOrders.remove(toRemove);
}
```

If needed, enforce additional logic after removal (e.g., cannot remove last item, or Order total must stay positive).

## 3. Tracking Article Movements
We want track (make an history) of all sell on an article.
- when an article is add to an order, create a new Movement with the articleId and the quantity
- when an article is remove from an order, just remove the Movement

### Domain Layer
Add a new Domain Entity

```java
public class Mvt {
    private Long id;
    private Long articleId;
    private int quantity;
    // getter and setter
}
```

### Application Service (createOrder + Mvt tracking)
The Application Service Layer orchestrate the business logic

```java
@Transactional
public Long createOrder(CreateOrderRequest request) {
    Order order = new Order(request.getClientId());

    for (CreateLineOrderDTO lineDTO : request.getLineOrders()) {
        Article article = articleRepository.findById(lineDTO.getArticleId())
                .orElseThrow(() -> new RuntimeException("Article not found"));

        LineOrder lineOrder = new LineOrder(
            article.getId(),
            lineDTO.getQuantity(),
            article.getPrice()
        );

        order.addLineOrder(lineOrder);

        // Track movement
        Mvt mvt = new Mvt(article.getId(), lineDTO.getQuantity());
        mvtRepository.save(mvt);
    }

    Client client = clientRepository.findById(request.getClientId())
            .orElseThrow(() -> new RuntimeException("Client not found"));

    if (client.isGoodClient()) {
        order.applyDiscount(BigDecimal.valueOf(0.10));
    }

    orderRepository.save(order);
    return order.getId();
}
```

### Application Service (deleteLineOrder + Mvt cleanup)
```java
@Transactional
public void deleteLineOrder(Long orderId, Long lineOrderId) {
    Order order = orderRepository.findById(orderId)
        .orElseThrow(() -> new RuntimeException("Order not found"));

    LineOrder toRemove = order.getLineOrderById(lineOrderId); // helper method
    if (toRemove == null) {
        throw new RuntimeException("LineOrder not found");
    }

    // Remove LineOrder from Order
    order.removeLineOrder(lineOrderId);

    // Remove associated movement
    mvtRepository.deleteByArticleIdAndQuantity(toRemove.getArticleId(), toRemove.getQuantity());

    orderRepository.save(order);
}
```

```java
class Order {
    public LineOrder getLineOrderById(Long lineOrderId) {
        return lineOrders.stream()
            .filter(l -> l.getId().equals(lineOrderId))
            .findFirst()
            .orElse(null);
    }
}
```

### Alternative: Use Domain Events
If you want to decouple `OrderApplicationService` from `MvtRepository`, raise a `LineOrderCreatedEvent` and handle it in a separate component.

#### On creation
```java
@Transactional
public Long createOrder(CreateOrderRequest request) {
    Order order = new Order(request.getClientId());

    for (CreateLineOrderDTO lineDTO : request.getLineOrders()) {
        Article article = articleRepository.findById(lineDTO.getArticleId())
                .orElseThrow(() -> new RuntimeException("Article not found"));

        LineOrder lineOrder = new LineOrder(article.getId(), lineDTO.getQuantity(), article.getPrice());
        order.addLineOrder(lineOrder);

        // Publish creation event
        eventPublisher.publish(new LineOrderCreatedEvent(article.getId(), lineDTO.getQuantity()));
    }

    Client client = clientRepository.findById(request.getClientId())
            .orElseThrow(() -> new RuntimeException("Client not found"));

    if (client.isGoodClient()) {
        order.applyDiscount(BigDecimal.valueOf(0.10));
    }

    orderRepository.save(order);
    return order.getId();
}
```

We could also raise the event in the Domain, in the `addLineOrder()` method 

#### Event Listeners
```java
@Component
public class MovementEventHandler {

    private final MvtRepository mvtRepository;

    public MovementEventHandler(MvtRepository mvtRepository) {
        this.mvtRepository = mvtRepository;
    }

    @EventListener
    public void onLineOrderCreated(LineOrderCreatedEvent event) {
        Mvt mvt = new Mvt(event.getArticleId(), event.getQuantity());
        mvtRepository.save(mvt);
    }

    @EventListener
    public void onLineOrderDeleted(LineOrderDeletedEvent event) {
        mvtRepository.deleteByArticleIdAndQuantity(event.getArticleId(), event.getQuantity());
    }
}
```

[^1]: https://martinfowler.com/bliki/DDD_Aggregate.html

