+++
title = "Transaction Script vers DDD"
series = ["Domain Driven Design"]
date = "2025-12-09"
+++

Dans les lignes qui suivent nous allons comprendre ensemble comment commencer à développer de façon DDD et arrêter de penser [Transaction Script](https://martinfowler.com/eaaCatalog/transactionScript.html)

## Pourquoi lire cet article !
- Toute la logique métier est dans les classes services `@Service` avec des méthodes très longue et d'une complexité cyclomatique élevé
- Vous n'arrivez pas à écrire des tests unitaires sur vos classes `@Service`

Si vous vous reconnaissez dans ces situations, cet article est pour vous. Nous allons partir d’une application existante répondant à ces critères, et la refactorer pour recentrer la logique sur le domaine métier, en nous appuyant sur les principes du Domain-Driven Design.

### Pourquoi le Transaction Script peut-il devenir problématique ?
Le pattern Transaction Script convient parfaitement aux applications simples, reposant principalement sur des opérations CRUD. Toutefois, dès que l'application commence à intégrer une logique métier plus complexe, continuer à s'appuyer sur ce modèle devient rapidement problématique. En effet, la logique métier tend alors à s’accumuler dans des méthodes procédurales de plusieurs dizaines, voire centaines de lignes. Cela s’oppose aux principes de la programmation orientée objet, selon lesquels les objets doivent encapsuler à la fois leurs données (*data*) et les comportements (*behaviors*) qui leur sont associés.

Ce glissement vers une logique de type Transaction Script est d’autant plus accentué par l’usage de certains frameworks populaires, tels que Spring. En effet, en proposant une structuration fondée sur des annotations telles que `@Controller`, `@Service` et `@Repository`, ces frameworks induisent souvent une séparation technique du code plutôt qu’une modélisation centrée sur le métier.

L’objectif de cet article est donc de revisiter les fondements de la programmation orientée objet — tels qu’abordés dès les premières années d’université — non pas comme une fin en soi, mais comme un moyen de recentrer la conception du code sur le métier. Car derrière cette volonté de mieux modéliser les comportements métier se cache une approche plus globale et structurante : celle du Domain-Driven Design (DDD).

{{< notice info >}} 
Vous n'avez pas tout compris ? Pas d'inquiétude, dans la section suivante nous allons coder.
{{< /notice >}} 

## Introduction de notre métier
Dans le domaine que nous étudions, il s'agit de gérer la création de commandes. 
- Une commande est composée de plusieurs lignes, chacune faisant référence à un article et à un client. 
- Lorsqu’un client souhaite passer une commande, il sélectionne un ou plusieurs articles, en précisant la quantité désirée.
- Chaque article possède un état matériel, qui indique par exemple s’il est disponible, réservé ou endommagé et un quantité en stock

### Règles métiers
- **Règle 1 : Disponibilité de l'article** : Lorsqu'une commande est créée, les articles concernés doivent être en stock et disponible à la vente.
- **Règle 2 : Existence du client** : Lorsqu'une requête de création de commande est reçue (contenant l’identifiant de l’article, la quantité et l’identifiant du client) il est nécessaire de vérifier que ce client est bien enregistré dans le système.
- **Règle 3 : Décrémenter le stock** : Lorsqu'une requête de création de commande est reçue on décrémentera le stock de l'article
- **Règle 4 : Passer commande au fournisseur** : Lorsque la quantité en stock d'un article est inférieure à 5 unité, passer commande chez le fournisseur (note : sera implémenté uniquement dans la partie DDD)

<!-- MaterielState annexe au stock ça permet de savoir si un article est disponible à la vente ou non (par exemple un rappel sur un produit alimentaire == RECALLS) -->

## Développement en Transaction Script
{{< notice tip >}} 
- GitHub project : [https://github.com/adrien1212/spring-transaction-script-to-ddd](https://github.com/adrien1212/spring-transaction-script-to-ddd)
{{< /notice >}} 

### Structuration
La structure du projet est assez classique pour du Transaction Script
```
fr.adriencaubel.tstoddd

    ├── TstodddApplication.java
    ├── controller
    │   ├── OrderController.java
    │   └── dto
    │       └── CreateOrderRequestModel.java
    ├── domain
    │   ├── Article.java
    │   ├── Client.java
    │   ├── MaterialState.java
    │   ├── OrderDocument.java
    │   └── OrderLine.java
    ├── repository
    │   ├── ArticleRepository.java
    │   ├── ClientRepository.java
    │   └── OrderRepository.java
    └── service
        └── DocumentService.java
```

### Développement
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

En exécutant la requête HTTP suivante on constate qu'une commande est bien créée.

```
curl --request POST \
  --url http://localhost:8075/orders \
  --header 'content-type: application/json' \
  --data '{
  "articleId": 1,
  "quantity": 5,
  "clientId": 1
}'
```

### Constat sur l’approche Transaction Script
#### Une testabilité réduite
Le premier écueil concerne les tests unitaires. Dans cette structure, la méthode `createOrder()` concentre l’ensemble des règles métier, tout en orchestrant des appels à plusieurs services et repositories. Tester finement chaque règle nécessite donc :
- de mettre en place des mocks pour tous les services dépendants (`ClientService`, `ArticleService`, `OrderRepository`)
- de simuler des scénarios métier complexes en maximisant le nombre de branches couvertes avec un minimum de tests

Au final, ce couplage fort entre la logique métier et l’infrastructure rend difficile l’écriture de tests réellement unitaires — on se rapproche en réalité de tests d'intégration partielle, coûteux à maintenir.

#### Une complexité métier difficile à contenir
Actuellement, le service traite un cas simple : un client commande un article disponible en quantité suffisante. Mais que se passe-t-il lorsque les règles métier évoluent ? Imaginons que l’on doive :
- appliquer des promotions conditionnelles sur certains articles (ex : "2 achetés = 1 offert"),
- vérifier que le client est éligible à une remise selon son ancienneté ou son statut,
- ou encore déclencher une notification si le stock devient critique.

Chacune de ces nouvelles exigences risque d’alourdir la méthode existante. Le service va se transformer en une suite de blocs if imbriqués, de validations conditionnelles, et de logiques dispersées dans le code, rendant l’ensemble difficile à lire et à maintenir.

#### Conclusion
Le pattern Transaction Script est parfaitement adapté aux cas simples, peu évolutifs, ou orientés "CRUD". Mais il devient un frein dès que le besoin métier devient riche, changeant, ou porteur de logique métier structurante. Cette constatation nous pousse naturellement à envisager une modélisation plus riche du domaine, où la logique métier est encapsulée au sein des objets, et organisée autour d'invariants : c’est précisément ce que propose l’approche Domain-Driven Design (DDD).


<!---  Développement en DDD. --->


## Développement en DDD
{{< notice note >}} 
Cet article n’a pas pour ambition d’être un cours exhaustif sur le Domain-Driven Design (DDD). Certains aspects sont volontairement simplifiés ou omis, afin de se concentrer sur l’objectif principal : questionner nos habitudes de développement et réfléchir à une > meilleure structuration du code autour du métier.

À titre personnel, c’est la découverte de certains principes du DDD, ainsi que la lecture de Anemic Domain Model de Martin Fowler, qui m’ont amené à repenser ma pratique de la programmation orientée objet, et à m’éloigner progressivement du pattern Transaction Script.

Si cette démarche vous interpelle, je ne peux que vous encourager à approfondir le sujet en explorant les travaux de Eric Evans ou de Vaughn Vernon.
{{< /notice >}} 

### Domain Model
Adopter une approche Domain-Driven Design (DDD), c’est avant tout réfléchir à qui porte les règles métier dans notre application. En d'autres termes, il s'agit de déterminer où les invariants doivent être protégés et par quels objets.

Pour cela, le DDD introduit la notion d’agrégat : une grappe d’entités cohérentes qui forment une unité logique autour d’un invariant métier. L’agrégat encapsule l’ensemble des règles qui doivent être maintenues consistantes lors d’une opération métier (création, modification, suppression, etc.)

La définition des agrégats est un moment crucial dans la modélisation. Il ne s'agit pas seulement de regrouper des entités "logiquement proches", mais de construire des unités de cohérence métier, responsables de leurs propres invariants.
Ces règles **ne doivent pas être dispersées dans différents services techniques**. Elles **doivent être encapsulées dans un ou plusieurs agrégats métiers**, capables de les faire respecter localement.

> An aggregate is a group of associated objects which are considered as one unit with regards to data changes. The aggregate is demarcated by a boundary which separates the objects inside from those outside (Eric Evans, Domain-Driven Design, 2004)

> The aggregate is a pattern that encourages designing for consistency boundaries and transactional integrity. (Vaughn Vernon, Implementing Domain-Driven Design, 2013)

> Aggregate is a cluster of domain objects that can be treated as a single unit. An example may be an order and its line-items, these will be separate objects, but it's useful to treat the order (together with its line items) as a single aggregate. (Martin Fowler)
 
![Aggregate](../images/transaction_script_to_ddd/aggregate.png)

#### Principes pour identifier un agrégat
Voici quelques principes largement acceptés pour bien définir les agrégats :
1. **Un agrégat est une unité de cohérence métier :**
Toutes les règles métier (invariants) qu’il contient doivent être vérifiables localement et garanties après chaque transaction.

2. **Un seul point d’entrée :**
Seule la racine d’agrégat (Aggregate Root) peut être référencée ou modifiée par d’autres parties du code. Les entités internes ne sont accessibles qu’à travers elle garantissant les invariants métier internes à l’agrégat.

3. **Limites transactionnelles claires :**
Une modification d’agrégat doit pouvoir se faire sans avoir à modifier un autre agrégat dans la même transaction.

4. **Stabilité et autonomie :**
Un agrégat doit pouvoir évoluer indépendamment des autres, et ses règles ne doivent pas dépendre d'un état externe ou d'autres agrégats.

### Quels seront nos agrégats ?
| Agrégat   | Racine  | Entités internes | Règles / Invariants                        |
|-----------|---------|------------------|--------------------------------------------|
| Article   | Article | MaterialState    | Un article ne peut être réservé que s’il est disponible.                   |
| Order     | Order   | OrderLine        | Au moins une ligne, quantité ligne > 0     |
| Client    | Client  | —                | -                                          |


#### Développement de Article
```java
protected Article() {}

public Article(String label, MaterialState materialState, int stock) {
    this.label = label;
    this.materialState = materialState;
    this.stock = stock;
}

public boolean isAvailableToSell() {
    return this.materialState.isAvailable() && this.stock > 0;
}

public void decrementStock(int amount) {
    if (amount <= 0) {
        throw new IllegalArgumentException("Amount must be positive");
    }
    if (this.stock < amount) {
        throw new IllegalArgumentException("No longer in stock");
    }
    if(!materialState.isAvailable()) {
        throw new IllegalArgumentException("Article is not available to be sold");
    }
    this.stock -= amount;
}
```

#### Développement de Order

```java
public Order(Long clientId, Article article, Integer quantity) {
    this.clientId = clientId;
    this.createdAt = LocalDateTime.now();
    addLine(article, quantity);
}

protected Order() {
    // for JPA
}

/**
 * On passe l'objet Article complet pour s'assurer de l'invariant (est disponible)
 */
public void addLine(Article article, Integer quantity) {
    if(quantity <= 0) {
        throw new IllegalArgumentException("Quantity must be positive");
    }

    if(article == null) {
        throw new IllegalArgumentException("Article cannot be null");
    }

    if(!article.isAvailableToSell()) {
        throw new IllegalArgumentException("Article is not available to sell");
    }

    OrderLine orderLine = new OrderLine(this, article.getId(), quantity);
    lines.add(orderLine);

    // ❌ : Remarque importante - voir Domain Service
    article.decrementStock(quantity)
}
```

#### Invariant nécessitant d'autres Agrégats
Dans la classe `Order`
- On passe l'objet Article complet pour s'assurer de l'invariant (est disponible)
- Par analogie, il peut sembler naturel de gérer également la décrémentation du stock directement au sein de l’agrégat Order. Toutefois, cette pratique, bien que techniquement envisageable, n’est pas conforme aux principes du Domain-Driven Design. **En effet, un agrégat ne doit jamais modifier l’état d’un autre agrégat racine.** 
    - Une telle responsabilité incombe à une couche supérieure de l’architecture (typiquement un Application Service ou un Domain Service), chargée d’orchestrer les interactions entre agrégats tout en préservant leur autonomie.
    - Ou en publiant un événement de domaine (ex. : `StockDecrementRequested`) que la couche application traitera en invoquant l’agrégat concerné (Article) de manière explicite.

### Domain Service
> Encapsulates business logic that doesn't naturally fit within a domain object, and are NOT typical CRUD operations

> This is where domain services can be helpful. You can attribute to them any business decisions which require additional information from the external world and which cannot be made by entities and value objects because of that

Le point central à retenir est que le Domain Service **appartient à la couche la plus centrale du modèle : le domaine**. Il contient de la logique métier pure, mais qui ne trouve pas naturellement sa place dans une unique entité ou un value object.

#### Retour sur la règle métier 4
Pour rappel, nous avons la règles suivante :
- **Règle 4 : Passer commande au fournisseur** : Lorsque la quantité en stock d'un article est inférieure à 5 unité, passer commande chez le fournisseur

Cette règle
- interagit avec deux concepts métiers : l’article et le fournisseur.
- déclenche un effet externe (passer une commande fournisseur).

Intégrer cette logique directement dans la classe `Article` violerait le principe de cohésion d’un agrégat. Une entité doit gérer son propre état et ses invariants internes, mais pas coordonner plusieurs agrégats ou dépendre d’un service externe. 
Cette règle doit donc être implémentée dans un Domain Service, qui agit comme coordonnateur métier. Il orchestre l’exécution de la règle sans surcharger les entités.

```java
@Component
public class StockManagementService {
    private final supplierService supplierService;
    /**
     * Ensures the article is available and decrements stock.
     * @throws IllegalStateException if stock cannot be decremented
     */
    public void checkAndDecrementStock(Article article, int quantity) {
        if (article == null) {
            throw new IllegalArgumentException("Article cannot be null");
        }

        if (!article.isAvailableToSell()) {
            throw new IllegalStateException("Article is not available for sale");
        }

        if (article.getStock() < 5) {
            supplierService.passCommand(article);
        }

        article.decrementStock(quantity); // Aggregate performs the state change
    }
}
```

Conséquence sur la classe `Order` : 
- `article.decrementStock(quantity)` ne va pas dans `addLine()` 
- mais géré par une couche supérieur *Domain Service* qui sera responsable de décrémenter la quantité 
- et le tout sera coordonnée par le *Application Service*

### Application Service

> The main difference between them is that domain services hold domain logic whereas application services don’t. [...] Domain services, therefore, participate in the decision-making process the same way entities and value objects do. And application services orchestrate those decisions the same way they orchestrate decisions made by entities and value objects.

La citation est tirée de l'article [Domain services vs Application services](https://enterprisecraftsmanship.com/posts/domain-vs-application-services/) qui explique très bien la différence. **Pour résumer, si l'aggregat rentre dans un état incohérent (par exemple, une ligne de commande avec un article non disponible à la vente) alors le code qui vérifie cette règle doit être dans le Domain Service.**

L'Application Service, à un rôle d'orchestrateur, souvent se traduisant par
1. Appel à la base de données pour récupérer des information (`articleRepository.findById(articleId)`)
2. Appel à la couche domaine (soit DomainService soit directement l'Aggregat Root)
3. Appel à la base de donnée pour sauvegarder (`orderRepository.save(order)`)

```java
@Service
@RequiredArgsConstructor
/**
 * OrderService --> OrderUseCase pour bien distinguer ApplicationService (use-case) de DomainService
 */
public class OrderUseCase {

    private final OrderRepository orderRepository;

    private final ArticleRepository articleRepository;

    private final StockManagementService stockManagementService;

    @Transactional
    public Order createOrder(Long clientId, Long articleId, int quantity) {
        // 1. Load Article aggregate
        Article article = articleRepository.findById(articleId)
                .orElseThrow(() -> new IllegalArgumentException("Article not found"));

        // 2. Use domain service to validate and decrement stock
        stockManagementService.checkAndDecrementStock(article, quantity);

        // 3. Create Order aggregate
        Order order = new Order(clientId, article, quantity);

        // 4. Persist aggregates
        orderRepository.save(order);
        articleRepository.save(article);

        return order;
    }
}
```

### Agrégat vs DomainService vs ApplicationService
| Préoccupation                          | Localisation                  | Justification                                                                 |
|---------------------------------------|-------------------------------|--------------------------------------------------------------------------------|
| Orchestration et gestion des transactions | `OrderApplicationService`     | Coordonne plusieurs agrégats et repositories et définit les frontières transactionnelles |
| Logique métier liée aux règles de stock   | `StockManagementService`      | Encapsule des règles métier transverses ne relevant pas d’un agrégat unique    |
| Mutation de l’état métier                | Agrégat `Article`             | Garantit la cohérence interne et l’application des invariants de l’agrégat     |

![DDD Circle](../images/transaction_script_to_ddd/ddd_circle.png)

### Réorganisation des packages
Pour le moment nos package sont organisés comme suit
```
fr.adriencaubel.tstoddd

    ├── TstodddApplication.java
    ├── controller
    │   ├── OrderController.java
    │   └── dto
    │       └── CreateOrderRequestModel.java
    ├── domain
    │   ├── Article.java
    │   ├── Client.java
    │   ├── MaterialState.java
    │   ├── OrderDocument.java
    │   └── OrderLine.java
    ├── repository
    │   ├── ArticleRepository.java
    │   ├── ClientRepository.java
    │   └── OrderRepository.java
    └── service
        └── DocumentService.java
```

Dans une volonté de renforcer l’orientation domain-centric, nous proposons une réorganisation des packages existants. Cette réorganisation vise à

| Couche           | Rôle                                                                                                      |
|------------------|-----------------------------------------------------------------------------------------------------------|
| `domain.model`   | Contient les entités, value objects, agrégats, définissant le cœur du domaine métier.                    |
| `domain.service` | Contient les Domain Services, pour les règles métiers complexes ne s’intégrant pas naturellement dans une entité ou un agrégat. |
| `application`    | Contient les cas d’usage métier (*Application Services*) orchestrant les appels aux agrégats. Cette couche est faible en logique, mais forte en orchestration. |
| `infrastructure` | Contient les détails techniques d’implémentation (persistance, contrôleurs web, adaptateurs, etc.). Elle dépend du domaine, mais jamais l'inverse. |

 
```
fr.adriencaubel. tstoddd/
    ├── TstodddApplication.java
    ├── application/
    │   └── order/
    │       └── OrderUseCase.java
    ├── domain/
    │   ├── model/
    │   │   ├── article/
    │   │   │   ├── Article.java
    │   │   │   └── MaterialState.java
    │   │   ├── client/
    │   │   │   └── Client.java
    │   │   └── order/
    │   │       ├── Order.java
    │   │       └── OrderLine.java
    │   └── service/
    │       └── StockManagementService.java
    └── infrastructure/
        ├── db/
        │   ├── ArticleRepository.java
        │   ├── ClientRepository.java
        │   └── OrderRepository.java
        └── web/
            ├── OrderController.java
            └── dto/
                └── CreateOrderRequestModel.java
```

### Créer une commande
```
curl --request POST \
  --url http://localhost:8075/orders \
  --header 'content-type: application/json' \
  --data '{
  "articleId": 1,
  "quantity": 5,
  "clientId": 1
}'
```

### Tests unitaires
Pour conclure cette section dédiée au Développement orienté DDD, il est essentiel de souligner l’un des avantages majeurs de cette architecture : la facilité de tester la logique métier de manière isolée. Grâce à la séparation stricte entre le domaine (logique métier pure) et l’infrastructure (détails techniques), il n’est plus nécessaire de mocker l’ensemble de l’application ou de faire appel à des composants externes pour écrire des tests pertinents.

#### Classe Order
```java
public class OrderTest {

    @Test
    void shouldCreateOrderWithValidArticleAndQuantity() {
        Article article = mock(Article.class);
        when(article.isAvailableToSell()).thenReturn(true);
        when(article.getId()).thenReturn(123L);

        Order order = new Order(1L, article, 2);

        assertEquals(1, order.getLines().size());
        assertEquals(123L, order.getLines().get(0).getArticleId());
    }

    ...
}
```

#### Classe StockManagementService
```java
@ExtendWith(MockitoExtension.class)
class StockManagementServiceTest {

    @Mock
    private SupplierService supplierService;

    @Mock
    private Article article;

    @InjectMocks
    private StockManagementService stockService;

    @Test
    void shouldDecrementStock_whenArticleIsAvailable_andStockIsSufficient() {
        // given
        when(article.isAvailableToSell()).thenReturn(true);
        when(article.getStock()).thenReturn(10);

        // when
        stockService.checkAndDecrementStock(article, 2);

        // then
        verify(article).decrementStock(2);
        verifyNoInteractions(supplierService);
    }

    @Test
    void shouldTriggerSupplierCommand_whenStockIsLow() {
        // given
        when(article.isAvailableToSell()).thenReturn(true);
        when(article.getStock()).thenReturn(3);

        // when
        stockService.checkAndDecrementStock(article, 1);

        // then
        verify(supplierService).passCommand(article);
        verify(article).decrementStock(1);
    }
    ...
}
```

Dans le cadre d’un test unitaire du service de domaine `StockManagementService`, l’objectif est de vérifier le comportement de coordination. C'est pour cette raison que nous ne vérifions pas que l'attribut `article.stock` doit bien décrémenter. La validation de la mutation d’état (stock) relève quant à elle des tests dédiés à l’agrégat `Article`, où l’on manipule des objets réels sans mocking.

#### Classe OrderUseCase

Il existe deux manières complémentaires de tester un Use Case :
- Test de coordination (puriste) : Le test vérifie uniquement l’orchestration : appel aux services, respect du scénario métier. Il ne teste pas l’état interne des objets.
- Test enrichi (scénario fonctionnel) : Le test vérifie aussi certaines valeurs retournées ou effets métier (ex. : décrémentation du stock), à condition d’utiliser des objets réels (pas de mocks).


Ci-dessous, nous avons privilégié un test enrichi, car les objets `Article` et `StockManagementService` utilisés ne sont pas des mocks, mais bien des instances réelles. Cela nous permet de vérifier non seulement la coordination des composants, mais également les effets métiers attendus, tels que la décrémentation effective du stock.

```java
@ExtendWith(MockitoExtension.class)
class OrderUseCaseTest {

    @Mock private OrderRepository orderRepository;
    @Mock private ArticleRepository articleRepository;
    @Spy private StockManagementService stockService;
    @InjectMocks private OrderUseCase orderUseCase;

    @Test
    void shouldCreateOrderSuccessfully() {
        Long articleId = 10L;
        Long clientId = 1L;
        int quantity = 2;

        MaterialState availableState = mock(MaterialState.class);
        when(availableState.isAvailable()).thenReturn(true);
        Article article = new Article("Cable", availableState, 10);
        article.setId(articleId);

        when(articleRepository.findById(articleId)).thenReturn(Optional.of(article));

        Order savedOrder = new Order(clientId, article, quantity);
        when(orderRepository.save(any())).thenReturn(savedOrder);

        Order result = orderUseCase.createOrder(new CreateOrderRequestModel(articleId, quantity, clientId));

        verify(stockService).checkAndDecrementStock(article, quantity);
        verify(orderRepository).save(any(Order.class));
        verify(articleRepository).save(article);

        assertNotNull(result);
        assertEquals(clientId, result.getClientId());
        assertEquals(1, result.getLines().size());
        assertEquals(quantity, result.getLines().get(0).getQuantity());
        assertEquals(articleId, result.getLines().get(0).getArticleId());
        assertEquals(8, article.getStock());
    }
}
```


#### Résumé tests
| Élément                | Testé sans Spring | Utilisation de mocks       | Séparation test métier / orchestration              |
|------------------------|-------------------|-----------------------------|------------------------------------------------------|
| `Order`                | ✅                | 🚫 (purement métier)        | Agrégat testé isolément                              |
| `StockManagementService` | ✅                | ✅                          | Domaine pur                                          |
| `OrderUseCase`         | ✅                | ✅ (mocks)                  | Orchestration testée sans base de données           |



## Conclusion
En conclusion, si l’architecture en couches classiques de type `@Controller` → `@Service` → `@Repository` peut suffire pour des applications simples, notamment de type CRUD, elle atteint rapidement ses limites dès que la complexité métier augmente.

C’est dans ces cas que le Domain-Driven Design prend tout son sens : il recentre le code sur le domaine métier, en plaçant la logique au cœur de l’application. Cette approche permet de mieux refléter les invariants métiers, de faciliter la maintenance du code, et de favoriser un langage commun entre développeurs et experts métier. Les bénéfices notables de cette approche :

- **Encapsulation du comportement (*behavior*) et des données au sein du modèle du domaine** (par exemple les entités `Order` et `Article`), ce qui permet de concevoir des objets métier riches, autonomes et cohérents.
- Cette modélisation comportementale permet une testabilité unitaire accrue, sans recours excessif aux mocks, car les objets du domaine peuvent être testés isolément de l'infrastructure.
- Pour coordonner des règles impliquant plusieurs entités ou agrégats, on introduit une couche dédiée via des *Domain Services*, tels que `StockManagementService`, qui encapsulent des règles métier sans appartenir à une entité spécifique.
- Enfin, les interactions avec les interfaces externes (persistance, APIs, etc.) sont gérées via des *Application Services*, qui orchestrent les cas d’usage sans embarquer de logique métier propre, respectant ainsi le principe de séparation des responsabilités.

Cette structuration permet de construire des systèmes plus maintenables, évolutifs et alignés sur la réalité métier. Elle facilite également l’adoption de bonnes pratiques de tests (unitaires, d’intégration, et fonctionnels) tout en assurant une séparation claire entre le *domaine*, *l’application*, et *l’infrastructure*.

## Sources
- [Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html) et [Transaction Script](https://martinfowler.com/eaaCatalog/transactionScript.html)
- [Comparison of Domain-Driven Design and Clean Architecture Concepts](https://khalilstemmler.com/articles/software-design-architecture/domain-driven-design-vs-clean-architecture/)
- [Entities, Aggregates, Domain services and Application services definitions](https://dddsample.sourceforge.net/characterization.html)
- [DDD, Hexagonal, Onion, Clean, CQRS, … How I put it all together](https://herbertograca.com/2017/11/16/explicit-architecture-01-ddd-hexagonal-onion-clean-cqrs-how-i-put-it-all-together/)

<br>

- *Implementing Domain-Driven Design* de Vaughn Vernon
- *Domain-Driven Design: Tackling Complexity in the Heart of Software* de Eric Evans