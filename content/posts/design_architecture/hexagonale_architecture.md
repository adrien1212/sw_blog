+++
title = "Ports and Adapters Pattern (Hexagonal Architecture)"
series = ["Design Layered Architecture"]
date = "2025-01-02"
+++

Becoming unmissable architecture year after year the Ports and Adapters celebrate 20 birthday. Written by Dr. Alistair Cockburn in 2005 in following [article](https://alistair.cockburn.us/hexagonal-architecture/)

## The KEY element
In the following section we will describe the architecture, but first of all you must remember the following sentence

![Hexagonal definition](../images/hexagonale_architecture/hexa_definition.png)

{{< notice warning >}}
**The domain must depend only on itself**
{{< /notice >}}

- The Business Logic (aka domain) doesn't depend on any layer; neither Presentation nor Persistance layers.
- The Business Logic doesn't depend on any library or framework; neither Hibernate nor Spring for Java programmers.

=> So, the idea of Hexagonal architecture, aka Ports and Adapters, is to focus on the Business logic, and have everything depend on that, and not the other way around.

## The Architecture
> Allow an application to equally be driven by users, programs, automated test or batch scripts, and to be developed and tested in isolation from its eventual run-time devices and databases. - Alistair Cockburn 2005, Ports and Adapters

![Hexagi architecture full](../images/hexagonale_architecture/hexa_architecture_full.png)

- A driven adapter does implement a driven port. A driven adapter is a software component outside the application that provides the specific implementation details required by a driven port; e.g. Database.  
  The business logic interacts with and makes calls to the driven port
- A driver adapter DOESN'T implement a driver port. A driver adapter is a software component outside the application that uses a driver port; e.g. REST Controller.  
  The implementation of a driver port is the business logic of the application itself.

### The Hexagon
Inside the hexagon we just have the things that are important for the business problem that the application is trying to solve.

The hexagon encapsulates the business logic, completely decoupled from any specific technology, framework, or real-world device. This ensures the application remains technology-agnostic.

Ports & Adapters pattern says nothing about the structure of the inside of the hexagon. You might organize it into layers, group it by feature components, or even allow it to devolve into spaghetti code ...
### The Port
Remember, *the idea of Hexagonal architecture, aka Ports and Adapters, is to focus on the Business logic, and have everything depend on that*. To achieve this, the Business Layer defines and uses interfaces (aka Port) to communicate to the Persistance Layer.

![Hexagonal Adapter](../images/hexagonale_architecture/hexa_port.png)

### The Adapter
Remember the introduction about [The Port]() subsection, *Business Layer defines and uses interfaces (aka Port) to communicate to the Persistance Layer*. By the way, the Persistance Layer will provide code (aka Adapter) that satisfies those interfaces allowing the Business Logic to talk to a MySQL, Postgres or whatever.

![Hexagonal Adapter](../images/hexagonale_architecture/hexa_adapter.png)

What an adapter does in the end is to convert an interface into another, so we could use the [Adapter Design Pattern](https://refactoring.guru/design-patterns/adapter) to implement it.

## Example and consequences 
Yet to abstract for you, I agree with you. The first time I read about Hexagonal Architecture I say "Ok and ... why it's helpful ?!"

We will design a a basic Article list system ... where we can just get all articles from database.

### Design the Hexagon and Driven Port
The first thing to design is the Business Logic (inside Hexagone). And remember :
- The Business Logic (aka domain) doesn't depend on any layer; neither Presentation nor Persistance layers.
- The Business Logic doesn't depend on any library or framework; neither Hibernate nor Spring for Java programmers.

![Hexagonal example](../images/hexagonale_architecture/hexa_example_hexa.png)

First, we define the domain entity (it's could be different from database entity)
```java
@Getter
@Setter
public class Article {
    private Long id;
    private String title;
    private String content;

}
```

Then, we define the Business Logic inside class named `Service` here. The following class has dependency to the `ArticleRepository` interface. But `ArticleService` is completely decoupled from data access concerns:
- It doesn't know if articles are stored in PostgreSQL, MongoDB, or even a text file
- It doesn't import any database-related classes
- It just works with a simple interface

```java
public interface ArticleRepository {
    List<Article> findAll();
}

public class ArticleService {
    private final ArticleRepository repository;  // Depends on interface, not implementation

    // Constructor Dependency Injection Principle
    public ArticleService(ArticleRepository repository) {
        this.repository = repository;
    }

    public List<Article> getAllArticles() {
        return repository.findAll();
    }
}
```

### Design Driven Adapter
The next step is to define `ArticleRepository` implementation(s). As mention above the domain entity could be different from database entity.
- `Article` classe doesn't contain any information about data storage
- `ArticleJPA` is the representation of data storage

![Hexagonal Adapter](../images/hexagonale_architecture/hexa_example_adapter.png)

```java
@Entity
@Table(name = "articles")
public class ArticleJPA {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String title;
    
    @Column(length = 1000)
    private String content;

    @Temporal(TemporalType.TIMESTAMP)
    @Column(nullable = false, updatable = false)
    @Generated(GenerationTime.INSERT)
    private Date createdAt;

    @Temporal(TemporalType.TIMESTAMP)
    @Column(nullable = false)
    @Generated(GenerationTime.ALWAYS)
    private Date updatedAt;
```

- `PostgresArticleRepository`: Real database implementation using JDBC
- `InMemoryArticleRepository`: Simple in-memory implementation using an ArrayList; useful for unit testing

{{< tabgroup >}}
{{< tab name="PostgresArticleRepository" >}}
```java
public class PostgresArticleRepository implements ArticleRepository {
    @PersistenceContext
    private final EntityManager entityManager;

    public PostgresArticleRepository(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    @Override
    public List<Article> getAllArticles() {
        return entityManager.createQuery("SELECT a FROM ArticleJPA a", ArticleJPA.class)
                .getResultList()
                .stream()
                .map(this::toDomain) // Use conversion method
                .collect(Collectors.toList());
    }

    private Article toDomain(ArticleJPA articleJPA) {
        Article article = new Article();
        article.setId(articleJPA.getId()); // Map JPA entity fields to domain object fields
        article.setTitle(articleJPA.getTitle());
        article.setContent(articleJPA.getContent());
        return article;
    }
}
```
{{< /tab >}}

{{< tab name="InMemoryArticleRepository" >}}
```java
public class InMemoryArticleRepository implements ArticleRepository {
    private final List<Article> articles = new ArrayList<>();

    public InMemoryArticleRepository() {
        // Add some sample data
        articles.add(new Article(1L, "First Article", "Content 1"));
        articles.add(new Article(2L, "Second Article", "Content 2"));
    }

    @Override
    public List<Article> findAll() {
        return new ArrayList<>(articles);  // Return a copy to prevent external modifications
    }
}
```
{{< /tab >}}
{{< /tabgroup >}}

The beauty of this design is that we could add more implementations (MongoDB, REST API, etc.) without touching any other code in our application. Each implementation is isolated in its own class and can be swapped in and out as needed.

### Design Hexagonal and Driver Port
The driver port represents the use cases that our application offers to the outside world.
First we define `ArticleUseCase` (Driver Port):
- The contract for what our application can do
- Is used by the outside world to interact with our application

Moreover, we separate our API contract from our domain model. 
- `Article` domain class isn't the same than `ArticleJPA` for Persistance Layer
- `Article` domain class isn't the same than `ArticleResponseDTO` for Presentation Layer

![Hexagonal Driver](../images/hexagonale_architecture/hexa_example_hexa_driver.png)

```java
@Getter
@Setter
public class ArticleResponseDTO {
    private Long id;
    private String title;
}
```

```java
public interface ArticleUseCase {
    List<ArticleResponseDTO> getAllArticles();
}

// Implement ArticleUseCase
// Return ArticleResponseDTO instead of domain object Article
public class ArticleService implements ArticleUseCase {
    private final ArticleRepository repository;

    @Override
    public List<ArticleResponseDTO> getAllArticles() {
        return repository.getAllArticles()
            .stream()
            .map(this::toDTO)
            .collect(Collectors.toList());
    }

    private ArticleResponseDTO toDTO(Article article) {
        return new ArticleResponseDTO(
            article.getId(),
            article.getTitle()
        );
    }
}
```

### Design Driver Adapter
And now, we can create the driver port (`ArticleController`) to access application functionality and it doesn't know about the concrete implementation (`ArticleService`)

![Hexagonal Driver](../images/hexagonale_architecture/hexa_example_hexa_controler.png)

```java
@RestController
@RequestMapping("/api/articles")
public class ArticleController {
    private final ArticleUseCase articleUseCase;

    public ArticleController(ArticleUseCase articleUseCase) {
        this.articleUseCase = articleUseCase;
    }

    @GetMapping
    public List<Article> retrieveAllArticles() {
        return articleUseCase.getAllArticles();
    }
}
```

Easy to add new ways to access our application (REST, GraphQL, CLI, etc.) by creating new driver adapters

## Several benefits
![Hexagonal full](../images/hexagonale_architecture/hexa_example_hexa_full.png)

This structure gives us several benefits:

- Clear separation between what our application offers (driver ports) and what it needs (driven ports)
- Business logic is isolated in the application layer

### Go deeper
If you're wondering where authentication should take place, I encourage you to read the article [The Minimal Service Layer Pattern](minimal_service_layer_pattern.md).  
According to it and this blog, authentication should be processed using an adapter (coupled to Spring, coupled to JWT)

## How to structure you application ?
- [Hexagonal Architecture An implementation guide - C2 Project Structure](https://jmgarridopaz.github.io/content/hexagonalarchitecture-ig/chapter2.html)
- [Hexagonal Architecture - C2 to C4](https://www.happycoders.eu/software-craftsmanship/hexagonal-architecture-java/)

## Hexagonal Architecture vs. Layered Architecture
According to the following [article]((https://www.happycoders.eu/software-craftsmanship/hexagonal-architecture/)), the layered architecture does not focus on the business logic but on the database:
![](https://www.happycoders.eu/wp-content/uploads/2023/01/hexagonal-architecture-vs-layered-architecture-simple.v2-800x317.png)

> While the hexagonal architecture added a port and two adapters with clear source code dependencies towards the core, the dependency chaos between the layers is growing: We now have transitive dependencies from the REST API to the data access layer, from the REST API to the third-party API, from the user interface to the data access layer, and from the user interface to the third-party API

![](https://www.happycoders.eu/wp-content/uploads/2023/01/hexagonal-architecture-vs-layered-architecture-with-source-code-dependencies-800x317.png)

See also [I can't really tell the difference between Hexagonal and Layered Architecture](https://softwareengineering.stackexchange.com/a/436195/385571)

##  Further reader
- [List of amazing articles](https://jmgarridopaz.github.io/content/articles.html)
- [List of amazing articles BIS](https://www.happycoders.eu/software-craftsmanship/hexagonal-architecture/)
- [Hexagonal Architecture Example: digging a Spring Boot implementation](https://beyondxscratch.com/2020/08/23/hexagonal-architecture-example-digging-a-spring-boot-implementation/)