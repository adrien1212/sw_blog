+++
title = "Never use Getter to access @OneToMany List"
series = ["JPA"]
date = "2025-01-24"
+++

Un Client à plusieurs Commande et nous souhaitons pouvoir récupérer l'ensemble des commandes d'un client depuis un endpoint REST `GET /customers/{id}/orders`. En nous appuyant sur Spring et une architecture en couche nous allons voir qu'un cas aussi simpliste que celui-ci va nous réserver quelque surprise et nous interroger sur l'organisation de notre code.

### Entités
Commençons par représenter nos entités métiers. Par soucis de simplicité nous nous affranchissons de DTO de retour.

```java
@Entity
@Table(name = "customers")
public class Customer {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @JsonManagedReference // to avoid infinite reference
    @OneToMany(mappedBy = "customer", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Order> orders;
}
```

```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String productName;

    @JsonBackReference // to avoid infinite reference
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;
```

## H1 : récupérer le client puis utiliser le getOrders()
Une première hypothèse consisterait à utiliser tous simplement le getter après avoir récupérer le client

```java
@Service
public class ClientService {

    @Autowired
    private CustomerRepository customerRepository;

    public List<Order> getOrdersByCustomerId(Long customerId) {
        Customer customer = customerRepository.findById(customerId)
                .orElseThrow(() -> new RuntimeException("Customer not found with ID: " + customerId));

        List<Order> orders = customer.getOrders(); // Use getter

        return orders;
    }
}
```

```java
@RestController
@RequestMapping("/customers")
public class CustomerController {

    @Autowired
    private CustomerService customerService;

    @GetMapping("/{customerId}/orders")
    public List<Order> getCustomerOrders(@PathVariable Long customerId) {
        List<Order> orders = customerService.getOrdersByCustomerId(customerId);
        return orders;
    }
}
```

Si nous exécutons ce code, il fonctionne et nous renvoie bien la liste des clients, MAIS

### Désactiver le Open Session In View
Par défaut, Spring active le Open Session In View. Or, cette option est considérée comme un anti-pattern (voir sources en bas de page); Ainsi, dans votre `application.properties` désactivez cette option en ajoutant

```yml
spring.jpa.open-in-view=false
```

En rappelant l'endpoint vous obtenez l’exception suivante `LazyInitializationException`

```
 [org.springframework.http.converter.HttpMessageNotWritableException: 
    Could not write JSON: failed to lazily initialize a collection of role: 
        fr.adriencaubel.springdatajpatest.entity.Customer.orders: could not initialize proxy - no Session]
```

### Explication
1. **FetchType.LAZY par défaut**
   - Dans une relation `@OneToMany`, le chargement par défaut est défini sur `FetchType.LAZY`. Cela signifie que les données associées (dans ce cas, la liste orders) ne sont pas chargées immédiatement lorsque l'entité principale (Customer) est récupérée.
   - Au lieu de cela, Hibernate crée un proxy pour la collection orders, qui n'est initialisé que si l'on tente d'y accéder explicitement.

2. **Fermeture de la transaction**
   - Lorsque la méthode findById(customerId) est appelée, Hibernate ouvre une session pour exécuter la requête et retourne l'entité Customer. Cependant, si l'option Open Session In View est désactivée (spring.jpa.open-in-view=false), la session est fermée immédiatement après la fin de la transaction.
   - Une fois la session fermée, les relations paresseuses (LAZY) ne peuvent plus être initialisées car Hibernate ne peut pas interagir avec la base de données pour récupérer les données manquantes.

3. **LazyInitializationException**
   - Lorsque vous essayez d'accéder à `customer.getOrders()` après la fermeture de la session, Hibernate tente de charger les données associées à partir de la base de données. Cependant, comme la session est déjà fermée, une exception `LazyInitializationException` est levée.

## Résolution du LazyInitializationException
### Fetch.EAGER
Une première solution consisterait à utiliser `FETCH.EAGER`, mais ceci impactera les performances de notre application.

```java
@OneToMany(mappedBy = "customer", cascade = CascadeType.ALL, fetch = FetchType.EAGER)
private List<Order> orders;
```

### Encapsuler dans une transaction
Rajouter l'annotation `@Transactional` sur notre méthode que la session Hibernate reste ouverte jusqu'à la fin de la méthode.

```java
@Transactional
public List<Order> getOrdersByCustomerId(Long customerId) {
    Customer customer = customerRepository.findById(customerId)
            .orElseThrow(() -> new RuntimeException("Customer not found with ID: " + customerId));

    List<Order> orders = customer.getOrders(); // Use getter

    return orders;
}
```

### Utiliser un JOIN FETCH
Nous pouvons écrire une requête avec ``JOIN FETCH`` pour récupérer explicitement l'association LAZY.

```java
public interface CustomerRepository extends JpaRepository<Customer, Long> {
    @Query("SELECT c FROM Customer c JOIN FETCH c.orders WHERE c.id = :id")
    Optional<Customer> findByIdWithOrders(@Param("id") Long id);
}
```

### Utiliser @EntityGraph
Nous pouvons utiliser `@EntityGraph` de JPA pour spécifier que certaines relations `LAZY` doivent être "fetched eagerly" pour une requête spécifique.

```java
public interface CustomerRepository extends JpaRepository<Customer, Long> {
    @EntityGraph(attributePaths = "orders")
    Optional<Customer> findById(Long id);
}
```

### Conclusion
L'hypothèse 1, nous a conduis à l'exception `LazyInitializationException` et plusieurs hypothèse pour la résoudre. Les deux dernières semblent les plus intéressante pour résoudre notre problème.

## H2 : coder un OrderRepository
Pour le moment nous nous sommes intéressé qu'au `CustomerRepository` et en supposant que c'était le rôle de ce repository de nous fournir la liste des commandes pour un client. Nous pouvons envisager de déléguer cette fonctionnalité à un autre repository 

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByCustomer(Customer customer);
}

public class OrderService implements IOrderService {
    public List<Order> findByCustomer(Long customerId) {
        return orderRepository.findByCustomer(customer);
    }
}
```

```java
@Service
public class CustomerService {
    @Autowired
    private final IOrderService orderService;

    public List<Order> getOrdersByCustomerId(Long customerId) {
        // Fetch the customer by ID
        Customer customer = customerRepository.findById(customerId)
                .orElseThrow(() -> new RuntimeException("Customer not found with ID: " + customerId));

        return orderService.findByCustomer(customer);
    }
}
```

### Contrainte de couplage
La principale contrainte de cette solution est le couplage logiciel entre `CustomerService` et `OrderService` là où avec `JOINT FETCH` ou `@EntityGraph` il était au niveau de la base de données.

A mon sens si vous êtes dans une application monolithe ce couplage n'est pas dérangeant. Il advient de se poser la question si nous souhaitons avoir deux microservices (appel réseau) distinct. Dans ce cas là l'appel http pour contacter le service commande depuis un client sera coûteux.

## Conclusion
Depuis un petit moment je me pose la question "où doit-on coder la récupération d'une sous-ressource" ?
- est-ce le rôle de `CustomerRepostiory` 
- ou préférer un `List<Order> findOrdersByCustomer(Customer customerId)` dans le `OrderRepostiory`

Le fait, de désactiver le Open Session In View, m'a permis de répondre à cette question (moyennement un compromis)
- Si nous mettons la récupération de la commande dans le `CustomerRepository` alors nous devons soit utiliser `JOINT FETCH` soit `@EntityGraph`
- D'un autre côté si le couplage entre `CustomerService` et `OrderRepository` n'est pas dérangeant alors cette solution est à préférer.

## Further reading
- [The Open Session In View Anti-Pattern](https://vladmihalcea.com/the-open-session-in-view-anti-pattern/)
- [Spring Boot Best Practice – disable OSIV to start receiving LazyInitializationException warnings again](https://codesoapbox.dev/spring-boot-best-practice-disable-osiv-to-start-receiving-lazyinitializationexception-warnings-again/)
- [A Guide to Spring’s Open Session in View - voir partie 6](https://www.baeldung.com/spring-open-session-in-view)