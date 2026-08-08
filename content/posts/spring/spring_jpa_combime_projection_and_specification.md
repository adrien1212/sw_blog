+++
title = "Spring Data JPA: Combine Projection and Specification"
series = ["Spring JPA"]
date = "2026-08-08"
+++

{{< notice tip >}} 
- GitHub project : [https://github.com/Adrien-Courses/Spring-jpa-specification-projection](https://github.com/Adrien-Courses/Spring-jpa-specification-projection)
{{< /notice >}} 

For a long time, combining a **Specification** (dynamic filtering) with a **Projection** (selecting only the columns you need) was considered impossible in Spring Data JPA. The usual answer on [StackOverflow](https://stackoverflow.com/questions/41692391/how-to-use-projections-and-specifications-with-spring-data-jpa) was:

> Starting from Spring Data 3.0, mixing projections and specifications is supported.
> [... BUT ...]
> the generated SQL still queries all the columns

This is no longer true. In this article we review projections, specifications, and how to combine them while keeping a narrow SQL query.

## Projection

- https://thorben-janssen.com/projections-with-jpa-and-hibernate/
- https://docs.spring.io/spring-data/jpa/reference/repositories/projections.html

A projection is an interface (or a record) declaring only the accessors you need. Spring Data reads its methods and selects only the matching columns.

```java
public interface ClientProjection {
    String getFirstName();
    String getLastName();
}

@Test
void projectionClient() {
    Collection<ClientProjection> result = clientRepository.findByLastName("Martin");
}
```

```
Hibernate: 
    select
        c1_0.first_name,
        c1_0.last_name 
    from
        client c1_0 
    where
        c1_0.last_name=?
```

Only the two projected columns are selected.

## Specification

- https://docs.spring.io/spring-data/jpa/reference/jpa/specifications.html

A `Specification` builds the `where` clause programmatically, which is handy when filters are optional.

```java
public class ClientSpecification {

    public static Specification<Client> filter(String firstName, String city, Boolean active) {
        return (root, query, cb) -> {
            List<Predicate> predicates = new ArrayList<>();
            if (firstName != null) predicates.add(cb.equal(root.get("firstName"), firstName));
            if (city != null) predicates.add(cb.equal(root.get("city"), city));
            if (active != null) predicates.add(cb.equal(root.get("active"), active));
            return cb.and(predicates.toArray(new Predicate[0]));
        };
    }
}

@Test
void specificationClient() {
    Specification<Client> specification = ClientSpecification.filter("Alice", null, null);

    List<Client> results = clientRepository.findAll(specification);
}
```

```
Hibernate: 
    select
        c1_0.id,
        c1_0.active,
        c1_0.city,
        c1_0.country,
        c1_0.created_at,
        c1_0.email,
        c1_0.first_name,
        c1_0.last_name 
    from
        client c1_0 
    where
        c1_0.first_name=?
```

`findAll(specification)` returns full entities, so every column is selected.

## Combine Projection with Specification

Since Spring Data JPA 3.x, `JpaSpecificationExecutor` exposes a fluent variant introduced with the concept of [Query by Example](https://docs.spring.io/spring-data/commons/reference/query-by-example.html)

```java
<S extends T, R> R findBy(Specification<T> spec, Function<FluentQuery.FetchableFluentQuery<S>, R> queryFunction);
```

```java
@Test
void combineProjectionAndSpecification() {
    Specification<Client> specification = ClientSpecification.filter("Alice", null, null);

    List<ClientProjection> result = clientRepository.findBy(specification, q -> q
            .as(ClientProjection.class)       // projection result class
            .all()
    );
}
```

```
Hibernate: 
    select
        c1_0.first_name,     // Only the projected fields are queried
        c1_0.last_name 
    from
        client c1_0 
    where
        c1_0.first_name=?
```

The dynamic filter and the narrow `select` are applied in the same query: the old "the generated SQL still queries all the columns" answer no longer holds.

## Combine Projection with Specification with a JOIN

### Nested interface

```java
public interface OrderWithClientView {
    String getReference();
    OrderStatus getStatus();
    BigDecimal getTotalAmount();
    ClientProjection getClient();
}

@Test
void nestedInterfaceProjection() {
    List<OrderWithClientView> rows = orderRepository.findBy(parisOver100(),
            q -> q.as(OrderWithClientView.class).all());
}
```

```
Hibernate: 
    select
        o1_0.reference,
        o1_0.status,
        o1_0.total_amount,
        c1_0.id,
        c1_0.active,
        c1_0.city,
        c1_0.country,
        c1_0.created_at,
        c1_0.email,
        c1_0.first_name,
        c1_0.last_name 
    from
        orders o1_0 
    join
        client c1_0 
            on c1_0.id=o1_0.client_id 
    where
        c1_0.city=? 
        and o1_0.total_amount>?
```

The nested `ClientProjection` is not pushed down into the `select`: the associated `Client` is materialized as an entity first, then wrapped in the projection. All client columns are fetched.

### Flat record

```java
public record OrderWithClientRow(
    String reference,
    OrderStatus status,
    BigDecimal totalAmount,
    String clientFirstName,
    String clientLastName) {
}

@Test
void flatRecordProjection() {
    List<OrderWithClientRow> rows = orderRepository.findBy(parisOver100(),
            q -> q.as(OrderWithClientRow.class).all());
}
```

```
Hibernate: 
    select
        o1_0.reference,
        o1_0.status,
        o1_0.total_amount,
        c1_0.first_name,
        c1_0.last_name 
    from
        orders o1_0 
    join
        client c1_0 
            on c1_0.id=o1_0.client_id 
    where
        c1_0.city=? 
        and o1_0.total_amount>?
```

With a flat DTO, each component maps to a single column path (`clientFirstName` → `client.firstName`), so the `select` stays minimal.

## Summary

| Approach | Nested shape | Narrow SQL | Works with Specification |
| --- | --- | --- | --- |
| Nested interface + `findBy` | ✅ | ❌ all 8 client columns | ✅ |
| Flat record + `findBy` | ❌ | ✅ exactly 5 columns | ✅ |

Specifications and projections do combine, and `findBy(spec, q -> q.as(...).all())` is the API to use. Prefer a flat record when the width of the SQL query matters; keep a nested interface only when the shape of the returned object is more important than the number of columns fetched.
