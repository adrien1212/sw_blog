+++
title = "Part 1 - Layered Architecture"
series = ["Domain Driven Design"]
date = "2025-05-18"
+++

Dans cet série d'article nous allons voir comment refactor une Architecture en Couche :
1. Partir d'une architecture classique en couche technique
2. Montrer qu'il est compliqué d'écrire un test unitaire
3. Ajouter des notions de DDD pour devenir domain-centric et faire les tests
4. Améliorer en séparant Application Layer et Domain Layer

Aujourd'hui, de nombreuses applications adoptent une architecture en couches, reposant sur les annotations @Controller, @Service et @Repository. Dans ce modèle, la logique métier est centralisée au sein de la couche Service, tandis que les entités ne contiennent généralement que des getters et des setters.

À première vue, cette séparation des responsabilités semble pertinente. Cependant, lorsqu'on souhaite écrire des tests unitaires sur cette fameuse couche Service, on se heurte rapidement à certaines difficultés.


Pour illustrer mes proposer regardons l'application suivante https://github.com/adrien1212/DDD-order-management. Elle permet de gérer les commandes d’un utilisateur. Voici ci-dessous un aperçu de la logique métier impliquée lors de la création d’une nouvelle commande :
- Vérifier si le client existe
- Vérifier s'il reste assez de stock
- Calculer le prix de chaque ligne (prix*quantité)
  - appliquer 5% de réduction s'il est VIP
- Créer la commande
  - Ajouter chaque ligne à la commande
  - Mettre à jour le total de la commande
  - persister la commande

```java
@Service
public class OrderService {
    public Order placeOrder(OrderRequestModel orderRequestModel) {
        Customer customer = customerRepository.findById(orderRequestModel.getCustomerId())
            .orElseThrow(() -> new IllegalArgumentException("Customer not found"));

        // Create the order
        Order order = new Order();
        order.setCustomer(customer);

        BigDecimal total = BigDecimal.ZERO;
        for (OrderItemRequestModel dto : orderRequestModel.getItems()) {
            // Check stock
            Inventory inventory = inventoryService.getInventoryByArticleId(dto.getArticleId());
            if (inventory.getStock() < dto.getQuantity()) {
                throw new IllegalStateException("Item out of stock");
            } else {
                inventory.setStock(inventory.getStock() - dto.getQuantity());
                inventory.setLastUpdate(LocalDate.now());
            }

            Article article = articleService.getArticle(dto.getArticleId());
            BigDecimal price = article.getPrice();
            BigDecimal lineTotal = price.multiply(BigDecimal.valueOf(dto.getQuantity()));
            if (customer.isVip()) {
                lineTotal = lineTotal.multiply(new BigDecimal("0.95")); // 5% discount for VIPs
            }

            // Add the item to the order
            OrderItem orderItem = new OrderItem();
            orderItem.setArticle(article);
            orderItem.setQuantity(dto.getQuantity());
            orderItem.setPrice(price);
            orderItem.setLineTotal(lineTotal);
            orderItem.setOrder(order);

            order.getItems().add(orderItem);

            total = total.add(lineTotal);
        }

        // Set the total of the order
        order.setTotal(total);

        return orderRepository.save(order);
    }
}
```

## Réaliser des tests unitaires
Un test unitaire consiste à tester une méthode ou une portion de code de façon isolée, sans dépendre des autres composants du système (ex : base de données, services externes).
Pour cela, on "mock" (simule) les dépendances de la classe testée afin de contrôler leur comportement et leurs retours. Ici, `placeOrder()` dépend de plusieurs composants externes `CustomerRepository`, `InventoryService`, `ArticleService` et `OrderRepository`.

Ensuite, on peut se demander les scénarios de tests intéressants :
- Vérifier que le total est bien calculé.
- Vérifier que le total est bien calculé + réduction

### Calculer le total
Pour vérifier que le calcul du total est correct, nous devons écrire le test suivant, où l'ensemble des appels externes sont mock.

```java
@Test
void placeOrder_shouldCalculateTotal() {
    // Arrange: préparer les mocks
    Customer customer = new Customer();
    customer.setId(1L);
    customer.setVip(false);

    Article article = new Article();
    article.setId(10L);
    article.setPrice(new BigDecimal("100"));

    Inventory inventory = new Inventory();
    inventory.setArticle(article);
    inventory.setStock(10);

    OrderRequestModel request = new OrderRequestModel();
    request.setCustomerId(1L);

    OrderItemRequestModel item = new OrderItemRequestModel();
    item.setArticleId(10L);
    item.setQuantity(2);
    request.setItems(List.of(item));

    when(customerRepository.findById(1L)).thenReturn(Optional.of(customer));
    when(articleService.getArticle(10L)).thenReturn(article);
    when(inventoryService.getInventoryByArticleId(10L)).thenReturn(inventory);
    when(orderRepository.save(any(Order.class))).thenAnswer(inv -> inv.getArgument(0));

    // Act
    Order order = orderService.placeOrder(request);

    // Assert
    BigDecimal expectedLineTotal = new BigDecimal("100").multiply(new BigDecimal(2));
    assertEquals(expectedLineTotal, order.getTotal());
    assertEquals(1, order.getItems().size());
    assertEquals(expectedLineTotal, order.getItems().get(0).getLineTotal());
}
```

## Améliorer via le DDD (part-2)
Le fait d'avoir positionner toute la logique métier dans la couche Service à conduit à écrire des [Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html), en d'autre termes nos entités ne contiennent que des getter et setter. 
Essayons de redonner à nos entité de la logique métier et regardons comment cela va permettre de faciliter la création de tests unitaires

### Logique métier dans Order
Nous allons recoder la classe `Order` pour qu'elle contienne de la logique métier 
```java
public class Order {
    public void addItem(Article article, int quantity, boolean isVip) {
        if (article == null) {
            throw new IllegalArgumentException("Article cannot be null");
        }
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }

        BigDecimal lineTotal = article.getPrice().multiply(BigDecimal.valueOf(quantity));
        if(isVip) {
            lineTotal = lineTotal.multiply(new BigDecimal("0.95"));
        }

        OrderItem orderItem = new OrderItem();
        orderItem.setArticle(article);
        orderItem.setQuantity(quantity);
        orderItem.setPrice(article.getPrice());
        orderItem.setLineTotal(lineTotal);
        orderItem.setOrder(this);
        items.add(orderItem);

        total = total.add(lineTotal);
    }
}
```

Puis modifier la couche Service pour appeler cette nouvelle méthode `addItem()`

```java
public class OrderService {
    public Order placeOrder(OrderRequestModel orderRequestModel) {
        Customer customer = customerRepository.findById(orderRequestModel.getCustomerId())
            .orElseThrow(() -> new IllegalArgumentException("Customer not found"));

        // Create the order
        Order order = new Order();
        order.setCustomer(customer);

        for (OrderItemRequestModel dto : orderRequestModel.getItems()) {
            Inventory inventory = inventoryService.getInventoryByArticleId(dto.getArticleId());
            if (inventory.getStock() < dto.getQuantity()) {
                throw new IllegalStateException("Item out of stock");
            } else {
                inventory.setStock(inventory.getStock() - dto.getQuantity());
                inventory.setLastUpdate(LocalDate.now());
            }

            Article article = articleService.getArticle(dto.getArticleId());

            order.addItem(article, dto.getQuantity() , customer.isVip());
        }

        return orderRepository.save(order);
    }
}
```

On remarque que :
- la méthode `placeOrder()` est plus courte
- car elle ne contient plus la logique métier associé à la création d'une commande et l'ajout d'une ligne à une commande


### Écriture d'un test unitaire
Si on passe le test unitaire précédent, il fonctionne toujours mais nous allons le modifier pour ne tester que la méthode `addItem()`. Plus besoin de mock car le domaine métier ne dépend de rien !
L'écrire du test devient très simple

```java
@Test
void addItem_shouldCalculateTotal() {
    Article article = new Article();
    article.setPrice(new BigDecimal("100"));

    Order order = new Order();

    order.addItem(article, 2, false); // Client VIP

    BigDecimal expectedLineTotal = new BigDecimal("100").multiply(BigDecimal.valueOf(2));
    assertEquals(expectedLineTotal, order.getTotal());
    assertEquals(1, order.getItems().size());
    assertEquals(expectedLineTotal, order.getItems().get(0).getLineTotal());
}
```

Mais vous allez dire, que là nous testons uniquement la classe `Order` sans vérifier l'ensemble (OrderService) est notamment que fait que l'inventaire soit mis à jour !

avec ce modèle DDD, les tests sur la classe Order ne couvrent que la logique métier interne à l’entité (calcul du total, gestion des articles…).
Ils ne testent pas l’orchestration globale, notamment :
- L’appel à l’inventaire et la mise à jour du stock
- La récupération des articles/clients
- La sauvegarde de la commande via le repository

C’est là que le découpage DDD montre toute sa puissance :
On sépare bien les tests : 
- Tests unitaires du domaine (Order, OrderItem)
- Tests unitaires du service d’application (OrderService)

Et pour le moment nous n'avons écrit qu'un test unitaire du domaine

### Test unitaire du service
- On ne vérifie plus si chaque ajout de ligne met à jour le total, c'est le test unitaire du domaine qui en est responsable
- par contre, on vérifie que le stock est bien décrémenté de deux, que le total final ets correct et que l'appel à la base de donnée est réalisé

```java
@Test
void placeOrder_shouldUpdateInventoryAndSaveOrder() {
    // Arrange: préparer les mocks
    Customer customer = new Customer();
    customer.setId(1L);
    customer.setVip(false);

    Article article = new Article();
    article.setId(10L);
    article.setPrice(new BigDecimal("100"));

    Inventory inventory = new Inventory();
    inventory.setArticle(article);
    inventory.setStock(10);

    OrderRequestModel request = new OrderRequestModel();
    request.setCustomerId(1L);

    OrderItemRequestModel item = new OrderItemRequestModel();
    item.setArticleId(10L);
    item.setQuantity(2);
    request.setItems(List.of(item));

    when(customerRepository.findById(1L)).thenReturn(Optional.of(customer));
    when(articleService.getArticle(10L)).thenReturn(article);
    when(inventoryService.getInventoryByArticleId(10L)).thenReturn(inventory);
    when(orderRepository.save(ArgumentMatchers.any(Order.class))).thenAnswer(inv -> inv.getArgument(0));

    // Act
    Order order = orderService.placeOrder(request);

    // Assert
    BigDecimal expectedLineTotal = new BigDecimal("100").multiply(new BigDecimal(2));
    assertEquals(expectedLineTotal, order.getTotal());

    assertEquals(8, inventory.getStock()); // Stock decrease

    verify(orderRepository).save(order);  // repository must be called
}
```

Le test unitaire du service ne vérifie que l'orchestration

### Résumé
- Tests du domaine → règles métier pures (pas d’infrastructure)
- Tests du service → orchestration et appels aux dépendances (avec mocks)

### Note
On peut également faire la même chose pour la mise à jour du stock dans l'inventaire

```java
class Inventory {
    public void decreaseStock(int quantity) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }

        if (stock < quantity) {
            throw new IllegalStateException("Item out of stock");
        }

        stock -= quantity;
        lastUpdate = LocalDate.now();
    }
}
```

Et la logique métier sur la gestion des stock n’apparaît plus dans la couche service

```java
public Order placeOrder(OrderRequestModel orderRequestModel) {
    Customer customer = customerRepository.findById(orderRequestModel.getCustomerId())
        .orElseThrow(() -> new IllegalArgumentException("Customer not found"));

    // Create the order
    Order order = new Order();
    order.setCustomer(customer);

    for (OrderItemRequestModel dto : orderRequestModel.getItems()) {
        Inventory inventory = inventoryService.getInventoryByArticleId(dto.getArticleId());

        inventory.decreaseStock(dto.getQuantity());

        Article article = articleService.getArticle(dto.getArticleId());

        order.addItem(article, dto.getQuantity() , customer.isVip());
    }

    return orderRepository.save(order);
}
```