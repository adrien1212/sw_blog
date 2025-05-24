+++
title = "How Database multitenancy works with Java?"
series = ["Design Practices"]
date = "2025-05-24"
+++

Dans cet article nous allons voir réaliser une application multi-tenant avec une base de données par tenant.

## La difficulté
Pour chaque requête HTTP, il faut savoir quel tenant fait la requête, et par la suite quelle base de données utiliser pour accéder/sauvegarder ses données.

## Stocker le tenantId
Dans une application web, chaque requête utilisateur (HTTP) est traitée par un thread distinct (modèle thread-per-request).

Dans un contexte multi-tenant, il faut garder l’information du tenant courant pour toute la durée du traitement de la requête. Et cette information **doit être disponible partout dans le code**. Une première solution serait de passer l'information du tenant en paramètre de chaque méthode, mais ceci devient vite compliqué. Une alternative est d'utiliser un `ThreadLocal`.

Un `ThreadLocal` permet de stocker une valeur spécifique à un thread (ici : l’id du tenant). 
- Chaque requête HTTP (traitée par un thread distinct) dispose de son propre contexte tenant.
- La valeur stockée dans le `ThreadLocal` sera accessible partout dans le code (services, DAO, interceptors…), tant que l’on reste dans le même thread – donc pendant toute la durée de la requête.

Regardons en détail le fonctionnement. 

### TenantContext et ThreadLocal
Tout d'abord la classe `TenantContext` contient le `ThreadLocal`
```java
public class TenantContext {
    private static final ThreadLocal<String> CURRENT_TENANT = new ThreadLocal<>();

    public static void setTenant(String tenantId) {
        CURRENT_TENANT.set(tenantId);
    }

    public static String getTenant() {
        return CURRENT_TENANT.get();
    }

    public static void clear() {
        CURRENT_TENANT.remove();
    }
}
```

Dès le début de la requête (dans un filtre ou un interceptor), on extrait l’identifiant du tenant (via un header, un token JWT, un sous-domaine, etc.) et on le stocke dans le `ThreadLocal`
- ici nous déclarons un filtre (qui sera exécuté à chaque requête http)
- dans ce filtre nous récupérons l'identifiant du tenant 
- pour l'associer au `ThreadLocal` 

```java
public class TenantFilter implements Filter {
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        try {
            String tenantId = // extract from header, cookie, etc.
            TenantContext.setTenant(tenantId);
            chain.doFilter(request, response);
        } finally {
            TenantContext.clear();
        }
    }
}
```

À partir de là, n’importe quel composant peut récupérer l’id du tenant courant simplement avec `String tenantId = TenantContext.getTenant();`

## Connexion à la base de données
Pour persister les données dans la bonne base de données (celle associée au tenant), il faut que chaque transaction utilise une connexion JDBC pointant vers cette base spécifique. Hibernate propose une gestion native du multitenant qui repose sur deux interfaces fondamentales à implémenter :
- `CurrentTenantIdentifierResolver` : pour déterminer dynamiquement l’identifiant du tenant courant.
- `MultiTenantConnectionProvider` : pour fournir la bonne connexion JDBC/DataSource selon l’identifiant du tenant.

### Cycle de vie
Cycle de vie typique d’une requête

1. Au début d’une requête (HTTP, ou autre), tu extrais l’ID tenant (header, JWT, argument…)
2. Le stocker dans le ThreadLocal (TenantContext.setTenant("xxx"))
3. Quand on souhaite accéder à la base données, en arrière plan on fait un `sessionFactory.openSession()`, qui :
   1. Appelle CurrentTenantIdentifierResolver (qui lit le ThreadLocal)
   2. Utilise MultiTenantConnectionProvider pour obtenir la bonne DataSource/Connection

### CurrentTenantIdentifierResolver
[CurrentTenantIdentifierResolver](https://docs.jboss.org/hibernate/orm/6.2/javadocs/org/hibernate/context/spi/CurrentTenantIdentifierResolver.html) définie la méthode `resolveCurrentTenantIdentifier()`, chargée de retourner l’identifiant du tenant courant. On peut l’implémenter de la manière suivante :

```java
public class MyTenantIdentifierResolver implements CurrentTenantIdentifierResolver {
    @Override
    public String resolveCurrentTenantIdentifier() {
        String tenant = TenantContext.getTenant(); // Via the ThreadLocal, le tenant est accessible partout
        return tenant != null ? tenant : "default_tenant";
    }
}
```

### MultiTenantConnectionProvider
L’interface `MultiTenantConnectionProvider` permet de fournir dynamiquement une connexion JDBC à la bonne base de données, en fonction de l’identifiant du tenant passé en paramètre. Exemple minimal :

```java
public class MyMultiTenantConnectionProvider implements MultiTenantConnectionProvider {
    private Map<String, DataSource> tenantDataSources;

    // Constructeur où on initialise la map des DataSources par tenant
    public MyMultiTenantConnectionProvider(Map<String, DataSource> tenantDataSources) {
        this.tenantDataSources = tenantDataSources;
    }

    @Override
    public Connection getConnection(String tenantIdentifier) throws SQLException {
        DataSource ds = tenantDataSources.get(tenantIdentifier);
        if(ds == null) {
            throw new SQLException("Unknown tenant: " + tenantIdentifier);
        }
        return ds.getConnection();
    }

    // autres méthodes à implémenter (deleguer au DataSource principal ou throw)
}
```

{{<mermaid>}}
sequenceDiagram
    participant Client
    participant AppServer
    participant ThreadLocal
    participant Hibernate
    participant DBs

    Client->>AppServer: Requête HTTP (avec ID tenant)
    AppServer->>ThreadLocal: Stocke le tenantId
    AppServer->>Hibernate: openSession()
    Hibernate->>ThreadLocal: Résout le tenant courant
    Hibernate->>AppServer: Appelle MultiTenantConnectionProvider.getConnection(tenantId)
    AppServer->>DBs: Ouvre connexion vers la base du tenant
    DBs-->>AppServer: Connexion JDBC
{{</mermaid>}}

## Conclusion
Pour réaliser une application multi-tenant :
- nous devons être en mesure de connaître le *tenantId* durant toute la durée de la requête. Pour ce faire nous stockons l'information dans un `ThreadLocal`
- pour accéder à la bonne base de données, nous déléguons le travail à Hibernate qui grâce au `ThreadLocal` est en mesure de retourner une connexion vers la base de données du tenant