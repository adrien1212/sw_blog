+++
title = "Implementing the Strategy Pattern: A Real-World Example"
series = ["Design Patterns"]
date = "2024-11-15"
+++

The Strategy Pattern is a powerful design solution that transforms complex conditional logic into clean, maintainable, and extensible code. This article explores a real-world implementation where we'll tackle the challenge of managing country-specific business rules in a client management system.

## Real-world example
Imagine managing a client data service that needs to follow distinct regulatory requirements for each country before return the list of clients:
- logging access in France and saving in database
- notifying government agencies in the USA
- while requiring no special handling in Germany. 
  
How do you implement these varying behaviors without creating a maintenance nightmare?

## The bad code

```java
public class BadClientService {
    public List<Client> getClients(String countryCode) {
        List<Client> clients = clientRepo.findAll();
        
        // Regulation obliges us to log with national specificities
        // Bad practice: Long if-else chain mixing different concerns
        if (countryCode.equals("FR")) {
            clientLogRepository.save("Read list of client : " + clients.size());
        } else if (countryCode.equals("US")) {
            usaGovService.notifyClientAccessAsync(clients)
                .thenAccept(response -> log.info("Notification sent: {}", response));
        } else if (countryCode.equals("DE")) {
            // Nothing
        } 
        // Bad practice: What if we need to add more regions?
        
        return clients;
    }
}
```

This problematic implementation has several issues:
- **Violates Open-Closed Principle**: Need to modify existing code to add new countries
- **Code Duplication**: Country-specific logic needs to be repeated for each method
- **Poor Separation of Concerns**: Service class knows too much about each country's specific requirements
- **Hard to Reuse**: Country-specific logic is trapped inside the service class

## With Strategy Design Pattern
I'll help you design a flexible service layer architecture using the Strategy pattern combined with a factory to handle different country-specific behaviors. This will make it easy to add new countries without modifying existing code.

[![](https://mermaid.ink/img/pako:eNqtlN9rwjAQx_-VEhhMZv-BIsJwP17cGIovoy8hvdZAmyvJVVac_u1LTNV27ZiCeUnvcrn7XPJttkxgAixiIufGPEmeaV7EKrDj4AlmWCnS9SyXoGhJmhNkdbD1IW5MJlIR6JQLmE7P7ocSDX1oFGCM32zu59LQ3ht7m_7gHPktu1i1q75oUGL9d9Ers7sRPn9ZTMXzOWZL0BspIICe60TjpuHeJ99hOAjYbWFl-G35bcJX3BzRq7Z1KXWPqYv8jjcj_p_ld7EuSrPWNNviCP3KAiyPJNR1U9g5WrQZ0JHSFpAqC0SDYuU-CtrkLQG2CQbkcoHqc8x82kfhTuoKyXev94JSCkmm9dXV7u6CBeScJCqzlqXx3sEfLgynA8fgN_Tl7aJXfVV2b9IFDYoiVmzMCtAFl4l9jg79x4zWUEDMIvuZQMqrnGIWq50N5RXhslaCRaQrGDONVbZmUcpzY62qTGzW5jk7eUuuPhHPNiROQW_NA-im3Q-PArsG?type=png)](https://mermaid.live/edit#pako:eNqtlN9rwjAQx_-VEhhMZv-BIsJwP17cGIovoy8hvdZAmyvJVVac_u1LTNV27ZiCeUnvcrn7XPJttkxgAixiIufGPEmeaV7EKrDj4AlmWCnS9SyXoGhJmhNkdbD1IW5MJlIR6JQLmE7P7ocSDX1oFGCM32zu59LQ3ht7m_7gHPktu1i1q75oUGL9d9Ers7sRPn9ZTMXzOWZL0BspIICe60TjpuHeJ99hOAjYbWFl-G35bcJX3BzRq7Z1KXWPqYv8jjcj_p_ld7EuSrPWNNviCP3KAiyPJNR1U9g5WrQZ0JHSFpAqC0SDYuU-CtrkLQG2CQbkcoHqc8x82kfhTuoKyXev94JSCkmm9dXV7u6CBeScJCqzlqXx3sEfLgynA8fgN_Tl7aJXfVV2b9IFDYoiVmzMCtAFl4l9jg79x4zWUEDMIvuZQMqrnGIWq50N5RXhslaCRaQrGDONVbZmUcpzY62qTGzW5jk7eUuuPhHPNiROQW_NA-im3Q-PArsG)


### Define the strategies
The Strategy pattern is implemented by defining a simple contract `CountryClientStrategy` with a single method `postProcessClients(List<Client>)`, allowing each country to encapsulate its unique regulatory requirements.

```java
// Strategy interface for country-specific behavior
public interface CountryClientStrategy {
    void postProcessClients(List<Client> clients);
}
```

Each implementation will define the concrete code for the regulation.

```java
// Implementation for France
public class FrenchClientStrategy implements CountryClientStrategy {
    @Autowired
    private final ClientLogRepository clientLogRepository;
    
    @Override
    public void postProcessClients(List<Client> clients) {
        clientLogRepository.save("Read list of client : " + clients.size());
    }
}


// Implementation for USA
public class UsaClientStrategy implements CountryClientStrategy {
    @Autowired
    private final UsaGovService usaGovService;
    
    @Override
    public void postProcessClients(List<Client> clients) {
        usaGovService.notifyClientAccessAsync(clients)
            .thenAccept(response -> log.info("Notification sent: {}", response));
    }
}

// Implementation for country without regulation
public class NoClientStrategy implements CountryClientStrategy {
    @Override
    public void postProcessClients(List<Client> clients) {
        // No additional processing needed
    }
}
```

### Rewriting the service layer
The service layer is rewritten by removing the conditional logic and replacing it with a call to the right strategy class.

```java
public class ClientService {
    private final ClientRepository clientRepo;
    private final CountryClientStrategyFactory strategyFactory;
    
    public ClientService(ClientRepository clientRepo, CountryClientStrategyFactory strategyFactory) {
        this.clientRepo = clientRepo;
        this.strategyFactory = strategyFactory;
    }
    
    public List<Client> getClients(String countryCode) {
        List<Client> clients = clientRepo.findAll();
        
        // Select the right strategy to use (no if-elseif)
        CountryClientStrategy strategy = strategyFactory.getStrategy(countryCode);
        strategy.postProcessClients(clients); // call it
        return clients;
    }
}
```

### Factory for creating country-specific strategies
The Factory pattern plays a crucial role in our country-specific client processing system by elegantly solving the challenge of strategy instantiation and selection. Instead of cluttering our service layer with complex object creation logic or hard-coding strategy selection, the factory acts as a centralized "strategy registry". When a new country needs to be supported, we simply register its strategy implementation with the factory - no need to modify existing code or add new conditions.

```java
public class CountryClientStrategyFactory {
    private final Map<String, CountryClientStrategy> strategies;
    
    public CountryClientStrategyFactory() {
        strategies = new HashMap<>();
        // Register strategies by country code
        strategies.put("FR", new FrenchClientStrategy());
        strategies.put("US", new UsaClientStrategy());
        strategies.put("DE", new NoClientStrategy());
    }
    
    public CountryClientStrategy getStrategy(String countryCode) {
        return strategies.get(countryCode);
    }
}
```

Without a factory, we'd need to either manually instantiate strategies (leading to tight coupling) or create complex if-else chains to select the appropriate strategy (violating the Open-Closed Principle).

## Benefits
This architecture offers several benefits:
- **Open/Closed Principle**: You can add new country strategies without modifying existing code
- **Single Responsibility**: Each strategy handles only its country-specific logic

### Adding a new country regulation
If we open up to a new country, we'll be able to respond to regulation effortlessly. Simply create a new strategy class and add it to the factory.

```java
public class CanadianClientStrategy implements CountryClientStrategy {
    
    @Override
    public void postProcessClients(List<Client> clients) {
        // Do something
    }
}

// Then in the factory, add:
strategies.put("CA", new CanadianClientStrategy());
```