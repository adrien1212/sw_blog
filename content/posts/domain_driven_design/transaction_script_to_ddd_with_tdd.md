+++
title = "Transaction Script vers DDD avec TDD"
series = ["Domain Driven Design"]
date = "2026-07-10"
+++

{{< notice info >}} 
Cet article est l'amélioration de l'article [Transaction Script vers DDD](../transaction_script_to_ddd)
{{< /notice >}} 


Les articles sur le DDD montrent souvent directement le modèle final : agrégats, services de domaine, invariants… Pourtant, lorsqu'on travaille en TDD, on ne connaît pas ce modèle dès le départ.

Dans cet article, nous allons partir d'un Transaction Script classique et laisser le TDD faire émerger progressivement les responsabilités du domaine jusqu'à obtenir un design DDD.

## Point de départ : un Transaction Script

Tout développeur a déjà été confronté à des méthodes de plusieurs centaines de lignes pour réaliser une action. Ceci porte le nom de  [Transaction Script](https://martinfowler.com/eaaCatalog/transactionScript.html) pattern. Très utile pour développer des applications CRUD mais rend la maintenance difficile lorsque notre application intègre de la logique métier.

### Règles métiers
- **Règle 1 : Disponibilité de l'article** : Lorsqu'une commande est créée, les articles concernés doivent être en stock et disponible à la vente.
- **Règle 2 : Existence du client** : Lorsqu'une requête de création de commande est reçue (contenant l’identifiant de l’article, la quantité et l’identifiant du client) il est nécessaire de vérifier que ce client est bien enregistré dans le système.
- **Règle 3 : Décrémenter le stock** : Lorsqu'une requête de création de commande est reçue on décrémentera le stock de l'article
- **Règle 4 : Passer commande au fournisseur** : Lorsque la quantité en stock d'un article est inférieure à 5 unité, passer commande chez le fournisseur (note : sera implémenté uniquement dans la partie DDD)

## Développement en Transaction Script
{{< notice tip >}} 
- GitHub project : [https://github.com/adrien1212/spring-transaction-script-to-ddd-with-tdd](https://github.com/adrien1212/spring-transaction-script-to-ddd-with-tdd)
{{< /notice >}} 

Et nous pouvons donner le code Java suivant. Je suis convaincu que beaucoup se reconnaîtront dans cette façon de coder (moi le premier). Comme dit, elle n'est pas mauvaise quand la logique métier est simple. 

```java
@RequiredArgsConstructor
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final ClientService clientService;
    private final ArticleService articleService;

    @Transactional
    public void createOrder(CreateOrderRequestModel requestModel) {
        // Règle 2 : Vérifier que le client existe
        if(!clientService.existsById(requestModel.clientId())) {
            throw new IllegalArgumentException("Client inconnu : " + requestModel.clientId());
        }

        // Récupérer l'article
        Article article = articleService.findById(requestModel.articleId());

        // Règle 1 : Vérifier que l'article est disponible
        if (!article.getMaterialState().getValue().equals("AVAILABLE")) {
            throw new IllegalStateException("Article non disponible : " + requestModel.articleId());
        }

        // Règle 3 : Décrémenter le stock
        if (article.getStock() < requestModel.quantity()) {
            throw new IllegalStateException("Stock insuffisant pour l'article : " + requestModel.articleId());
        }
        article.setStock(article.getStock() - requestModel.quantity());

        // Créer la ligne de commande
        Order order = new Order(requestModel.clientId());
        OrderLine line = new OrderLine(order, requestModel.articleId(), requestModel.quantity());
        order.getLines().add(line);

        orderRepository.save(order);
    }
}
```

Par exemple, le code suivant permettrait de créer une quantité négative :
`new OrderLine(order, articleId, -10);`


De la même manière, un autre cas d’usage pourrait modifier directement le stock :
`article.setStock(-100);`

=> Le modèle ne protège donc pas ses propres invariants.

{{% notice warning %}}
Notre objectif est de faire émerger le design via le TDD
{{% /notice %}}

## Protéger les règles métiers
En DDD, une responsabilité essentielle de l’agrégat est de garantir que ses invariants restent vrais après chaque changement d’état. Toutes les modifications de l’agrégat doivent passer par sa racine.

### Première étape TDD : faire émerger l’agrégat Order
Nous commençons par la règle la plus simple 

> Une commande ne peut contenir qu’une quantité strictement positive.

#### Red : écrire un test qui échoue

```java
public class OrderTest {
    @Test
    public void should_reject_a_line_with_a_zero_quantity() {
        Order order = new Order(1L);

        Assertions.assertThrows(
                InvalidOrderQuantityException.class,
                () -> order.addLine(1L, -10)
        );
    }
}
```

L'écriture de ce test force l'apparition :
- d'une méthode métier `addLine`
- d'une exception

#### Green : implémentation minimale

```java
public class InvalidOrderQuantityException extends RuntimeException {
    public InvalidOrderQuantityException(int quantity) { super("La quantité doit être strictement positive : " + quantity); }
}
```

```java
class Order {
    ...
    public void addLine(long articleId, int quantity) {
        if(quantity < 0) {
            throw new InvalidOrderQuantityException(quantity);
        }

        OrderLine orderLine = new OrderLine(this, articleId, quantity);
        lines.add(orderLine);
    }
}
```

### Deuxième étape TDD : faire émerger l'agrégat Article
La règle suivante est

> Un article ne peut être réservé que si le stock disponible est suffisant.

#### Red : test qui échoue
```java
public class ArticleTest {
    @Test
    public void should_reject_a_reservation_when_stock_is_insufficient() {
        Article article = new Article("pelle", new MaterialState("AVAILABLE"), 5);

        Assertions.assertThrows(
                InsufficientStock.class,
                () -> article.reserve(10)
        );
    }

    @Test
    public void should_reject_a_reservation_when_stock_is_zero() {
        Article article = new Article("pelle", new MaterialState("AVAILABLE"), 5);

        Assertions.assertThrows(
                InvalidOrderQuantityException.class,
                () -> article.reserve(0)
        );
    }

    @Test
    public void should_reject_a_reservation_when_materiel_not_available() {
        Article article = new Article("pelle", new MaterialState("OUTOFSTOCK"), 5);

        Assertions.assertThrows(
                ArticleNoLongerAvailableException.class,
                () -> article.reserve(1)
        );
    }

    @Test
    public void should_decrease_stock_when_article_is_reserved() {
        Article article = new Article("pelle", new MaterialState("AVAILABLE"), 5);

        article.reserve(1);

        Assertions.assertEquals(4, article.getStock());
    }
}
```

On fait émerger les concept 
- réservation sur l'article
- de plusieurs cas pouvant lever des exceptions

Et on définit dans le dernier test, le cas passant

#### Green : implémentation minimale
Dans la classe `Article` notre méthode `reserve` devient

```java
public void reserve(int quantity) {
    if(quantity <= 0) {
        throw new InvalidOrderQuantityException(quantity);
    }

    if(this.stock - quantity < 0) {
        throw new InsufficientStock(quantity);
    }

    if(!materialState.isAvailable()) {
        throw new ArticleNoLongerAvailableException();
    }

    this.stock -= quantity;
}
```

### Troisème étape DDD : Faire émerger le service applicatif
Après avoir placé les invariants dans les agrégats, nous pouvons écrire un test du cas d’usage complet.

Le service applicatif doit :
- vérifier l’existence du client ;
- charger l’article ;
- demander à l’article de réserver la quantité ;
- créer la commande ;
- sauvegarder les agrégats modifiés.

#### Fixer l'existant
Avant de refactorer `OrderService`, on commence par écrire des tests de characterization pour documenter le comportement actuel et sécuriser la suite.

Les 4 cas couverts sont :
- **Client inconnu** : la création de commande échoue avec une exception.
- **Article non disponible** : la création de commande échoue si l’article n’est pas dans l’état `AVAILABLE`.
- **Stock insuffisant** : la création de commande échoue si la quantité demandée dépasse le stock disponible.
- **Cas nominal** : la commande est créée et le stock de l’article est décrémenté quand toutes les règles sont respectées.

À ce stade, on ne cherche pas encore à améliorer le design. L’objectif est simplement de **figer le comportement du transaction script** pour disposer d’un filet de sécurité avant le refactoring DDD.

Voir branche : [part-2-test-existing-service](https://github.com/adrien1212/spring-transaction-script-to-ddd-with-tdd/tree/part-2-test-existing-service)

```java
public class OrderServiceTest {

    private final OrderRepository orderRepository = mock(OrderRepository.class);
    private final ClientService clientService = mock(ClientService.class);
    private final ArticleService articleService = mock(ArticleService.class);
    private final OrderService orderService = new OrderService(orderRepository, clientService, articleService);

    @Test
    void should_throw_when_client_is_unknown() {

    }

        @Test
    void should_throw_when_article_is_not_available() {

    }

    @Test
    void should_throw_when_stock_is_insufficient() {

    }

    @Test
    void should_create_order_and_decrease_stock_when_request_is_valid() {

    }
```

#### Déléguer aux agrégats
Une fois le comportement de `OrderService` figé par les tests, on peut commencer le refactoring interne sans changer le contrat du service.

Dans cette étape, `OrderService` ne gère plus lui-même :
- la vérification de disponibilité de l’article
- la vérification du stock
- la décrémentation du stock
- l’ajout manuel d’une ligne de commande

=> Ces responsabilités sont désormais déléguées aux comportements métier déjà présents dans les agrégats

```java
@Transactional
public void createOrder(CreateOrderRequestModel requestModel) {
    // Règle 2 : Vérifier que le client existe
    if(!clientService.existsById(requestModel.clientId())) {
        throw new IllegalArgumentException("Client inconnu : " + requestModel.clientId());
    }

    // Récupérer l'article
    Article article = articleService.findById(requestModel.articleId());

    // Règle 1+3 : Vérifier que l'article est disponible et décrémenter stock
    article.reserve(requestModel.quantity());

    // Créer la ligne de commande
    Order order = new Order(requestModel.clientId());
    order.addLine(article.getId(), requestModel.quantity());

    orderRepository.save(order);
    articleRepository.save(article);
}
```

Les premiers tests de characterization ont figé le comportement initial du transaction script.

Après avoir déplacé les règles métier dans les agrégats, une décision de design apparaît :
- soit le service d’application conserve son contrat historique et traduit les exceptions du domaine
- soit il expose désormais explicitement les exceptions métier du domaine

Dans notre cas, nous choisissons de laisser remonter les exceptions métier. Les tests évoluent donc, non pas parce que le refactoring aurait été mal conduit, mais parce que nous avons fait évoluer le contrat applicatif du service.

En parallèle, comme le use case modifie maintenant deux agrégats (`Order` et `Article`), les tests du service doivent aussi vérifier la persistance des deux objets.
- On adapte `should_create_order_and_decrease_stock_when_request_is_valid` pour prendre en compte à la fois la persistance de `Order` et de `Article`
- On peut ajouter un test `should_persist_order_and_article_when_order_is_created`

### Conclusion intermédiaire

Nous sommes partis d’un transaction script.

Grâce au TDD, nous avons d’abord identifié les agrégats et fait émerger leurs invariants métier. Le design n’a pas été décidé à l’avance : il s’est révélé progressivement à travers les tests, en faisant apparaître des comportements métier explicites comme `reserve` sur `Article` et `addLine` sur `Order`.

Nous avons ensuite abordé une étape plus délicate : le refactoring de la couche service. Pour sécuriser cette transformation, nous avons commencé par écrire des tests de characterization afin de figer le comportement existant. Une fois ce filet de sécurité en place, nous avons pu faire évoluer `OrderService` pour qu’il délègue les règles métier aux agrégats au lieu de les porter lui-même.

À ce stade, le service devient plus clairement un service d’application : il orchestre le cas d’usage, tandis que les invariants métier sont désormais portés par le modèle de domaine.

*Voir le code [part-3-refactor-service](https://github.com/adrien1212/spring-transaction-script-to-ddd-with-tdd/tree/part-3-refactor-service)*


## Ajouter un peu de complexité
Jusqu’à présent, la possibilité de réserver un article dépendait uniquement de son propre état :
- son état devait autoriser la vente ;
- son stock devait être suffisant.

Ces règles pouvaient naturellement être protégées par l’agrégat Article : `article.reserve(quantity);`

Le besoin métier évolue désormais. Pour déterminer si une vente est autorisée, nous devons appliquer les règles suivantes :
- l’article doit être disponible ;
- le client doit être majeur ;
- l’article ne doit pas être interdit dans la région du client.

**=> Nous allons faire émerger la solution une avec le TDD**

### Ajouter des tests sur les nouvelles contraintes
Dans `OrderServiceTest` nous ajoutons plusieurs tests 
- `should_reject_order_when_client_is_minor`
- `should_reject_order_when_article_is_forbidden_in_client_region`

Ces deux tests passent bien au rouge, aucune exception n'est levée malgré que le client est mineur ou que l'article soit interdit dans la région.

```java
// Exemple du premier test
@Test
void should_reject_order_when_client_is_minor() {
    CreateOrderRequestModel request = new CreateOrderRequestModel(10L, 2, 99L);
    Client minorClient = new Client("Tom", LocalDate.now().minusYears(17), Region.FRANCE);
    Article article = new Article("pelle", new MaterialState("AVAILABLE"), 10);
    article.setId(10L);

    when(clientService.existsById(99L)).thenReturn(true);
    when(clientService.findById(99L)).thenReturn(minorClient);
    when(articleService.findById(10L)).thenReturn(article);

    assertThrows(
            MinorClientCannotOrder.class,
            () -> orderService.createOrder(request)
    );

    verify(orderRepository, never()).save(any());
    verify(articleRepository, never()).save(any());
}
```

Une fois les nouveaux tests écrits, la manière la plus simple de les faire passer consiste à ajouter temporairement les nouvelles vérifications directement dans `OrderService` :
- charger le client ;
- refuser la commande si le client est mineur ;
- refuser la commande si l’article est interdit dans la région du client.

```java
@Transactional
public void createOrder(CreateOrderRequestModel requestModel) {
    // Règle 2 : Vérifier que le client existe
    if(!clientService.existsById(requestModel.clientId())) {
        throw new IllegalArgumentException("Client inconnu : " + requestModel.clientId());
    }

    Client client = clientService.findById(requestModel.clientId());

    // Récupérer l'article
    Article article = articleService.findById(requestModel.articleId());

    if (!client.isAdult()) {
        throw new MinorClientCannotOrder();
    }

    if (article.isForbiddenIn(client.getRegion())) {
        throw new ArticleForbiddenInRegion(client.getRegion());
    }

    // Règle 1+3 : Vérifier que l'article est disponible et décrémenter stock
    article.reserve(requestModel.quantity());

    // Créer la ligne de commande
    Order order = new Order(requestModel.clientId());
    order.addLine(article.getId(), requestModel.quantity());

    orderRepository.save(order);
    articleRepository.save(article);
}
```

Voir le code [https://github.com/adrien1212/spring-transaction-script-to-ddd-with-tdd/tree/part-4-add-tests-for-new-requirement](https://github.com/adrien1212/spring-transaction-script-to-ddd-with-tdd/tree/part-4-add-tests-for-new-requirement)

### Poser la question de responsabilité
Le premier Green nous a permis de faire passer les tests rapidement, mais il fait apparaître un problème de design.

`OrderService` contient désormais les règles suivantes :
- le client doit être majeur ;
- l’article ne doit pas être interdit dans la région du client.

Or ces règles ne relèvent pas naturellement du service applicatif.  
Le rôle de `OrderService` est d’orchestrer le cas d’usage, pas de prendre lui-même une décision métier.

Peut-on alors déplacer cette logique dans un agrégat ?

- **Dans `Article` ?**  
  Ce serait lui donner connaissance du client et de sa région, alors que sa responsabilité naturelle est de protéger son état propre, comme la disponibilité ou le stock.

- **Dans `Client` ?**  
  Ce serait lui donner connaissance des contraintes de vente propres à l’article, ce qui sortirait également de son périmètre.

Aucun des deux agrégats ne peut donc porter seul cette décision sans élargir artificiellement sa responsabilité.

Nous sommes face à une règle métier qui traverse plusieurs concepts du domaine :
- `Article`
- `Client`
- `Region`

Cette situation révèle qu’il ne s’agit ni d’un invariant local de `Article`, ni d’un invariant local de `Client`, mais d’une **politique métier transverse**.

C’est précisément le type de décision qui constitue un bon candidat pour un **Domain Service**.

#### Domain Service : OrderPolicy

```java
@Component
public class OrderPolicy {
    public void checkThatClientCanOrder(Article article, Client client) {
        if (!client.isAdult()) {
            throw new MinorClientCannotOrder();
        }

        if (article.isForbiddenIn(client.getRegion())) {
            throw new ArticleForbiddenInRegion(client.getRegion());
        }
    }
}
```

`OrderService` redevient alors un service d’application plus lisible : il orchestre le cas d’usage, tandis que la politique métier transverse est portée explicitement par le domaine.

```java
@Transactional
public void createOrder(CreateOrderRequestModel requestModel) {
    // Règle 2 : Vérifier que le client existe
    if(!clientService.existsById(requestModel.clientId())) {
        throw new IllegalArgumentException("Client inconnu : " + requestModel.clientId());
    }

    Client client = clientService.findById(requestModel.clientId());

    // Récupérer l'article
    Article article = articleService.findById(requestModel.articleId());

    orderPolicy.checkThatClientCanOrder(article, client);

    // Règle 1+3 : Vérifier que l'article est disponible et décrémenter stock
    article.reserve(requestModel.quantity());

    // Créer la ligne de commande
    Order order = new Order(requestModel.clientId());
    order.addLine(article.getId(), requestModel.quantity());

    orderRepository.save(order);
    articleRepository.save(article);
}
```

Jusqu’ici, `OrderServiceTest` prouve que le cas d’usage fonctionne. Nous n'avons pas à modifier nos tests existants `should_reject_order_when_client_is_minor`, `should_reject_order_when_article_is_forbidden_in_client_region` qui nous assure la non régression lors du déplacement.

Mais maintenant que la décision métier a été extraite dans OrderPolicy, il faut aussi la tester directement, sans passer par le service applicatif. Une fois la décision extraite dans `OrderPolicy`, il devient possible de la tester directement, sans passer par `OrderService`.

C’est un bon indicateur de design : la règle métier transverse est désormais portée par un objet

Déplacer dans `OrderPolicyTest`
- should_reject_order_when_client_is_minor
- should_reject_order_when_article_is_forbidden_in_client_region
- should_allow_order_when_client_is_adult_and_article_is_allowed_in_region

Voir le code [https://github.com/adrien1212/spring-transaction-script-to-ddd-with-tdd/tree/part-5-add-domain-service](https://github.com/adrien1212/spring-transaction-script-to-ddd-with-tdd/tree/part-5-add-domain-service)

### Conclusion de la partie
Le TDD ne nous a pas conduit immédiatement au Domain Service. Il nous a d’abord conduit à faire fonctionner le cas d’usage, puis à constater qu’aucun agrégat ne pouvait porter proprement cette décision.  

Le Domain Service n’est donc pas une idée posée à l’avance : il apparaît comme la réponse à un problème de responsabilité mis en évidence par les tests et par le premier Green.

## Conclusion

Nous sommes partis d’un `Transaction Script`, dans lequel les règles métier et l’orchestration applicative étaient mélangées dans `OrderService`.

Dans une première étape, le TDD nous a permis de faire émerger les invariants métier au bon endroit : dans les agrégats. Les comportements `reserve` sur `Article` et `addLine` sur `Order` ne viennent pas d’un design décidé à l’avance, mais des tests qui ont révélé où ces règles devaient vivre.

Nous avons ensuite refactoré `OrderService`. Pour le faire proprement, nous avons d’abord écrit des tests de characterization afin de figer le comportement existant. Cela nous a permis de déplacer la logique métier vers le domaine sans perdre le contrôle sur le comportement du cas d’usage.

Enfin, nous avons ajouté un peu de complexité métier : la possibilité de vendre un article ne dépendait plus seulement de son état interne, mais aussi du client et de sa région. Cette évolution a fait émerger une nouvelle question de responsabilité. Aucun agrégat ne pouvait porter seul cette décision sans sortir de son périmètre. Le design a alors évolué naturellement vers un `Domain Service`, `OrderPolicy`, chargé d’exprimer cette politique métier transverse.

**C’est là l’apport principal du TDD dans cette démarche : il ne sert pas seulement à vérifier du code, mais à faire apparaître progressivement un design DDD plus juste,** en partant des règles métier et des responsabilités réelles du domaine.