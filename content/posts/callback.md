+++
title = "Callback with Java"
series = ["Design Practices"]
date = "2024-09-20"
+++

The callback pattern is used to notify a single caller about the completion of an operation, often with a result.

## Synchronous Callback

```java
// Callback interface (remains the same)
interface Callback {
    void onComplete(String result);
}

// Class that performs an operation and invokes the callback synchronously
class SyncOperation {
    public void performOperation(Callback callback) {
        // Simulating a synchronous operation
        try {
            Thread.sleep(2000); // Simulate work
            callback.onComplete("Operation completed successfully");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

// Usage
public class SyncCallbackExample {
    public static void main(String[] args) {
        SyncOperation operation = new SyncOperation();
        
        System.out.println("Operation starting...");
        operation.performOperation(result -> System.out.println(result));
        System.out.println("Operation finished.");
    }
}
```


## Asynchronous Callback
 It's typically used for asynchronous operations. They don’t wait for one another before performing their operations. We can achieve it by creating a new Thread to perform the callback

```java
// Class that performs an operation and invokes the callback
class AsyncOperation {
    public void performOperation(Callback callback) {
        // Simulating an asynchronous operation
        new Thread(() -> {
            try {
                Thread.sleep(2000); // Simulate work
                callback.onComplete("Operation completed successfully");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
}

// Usage
public class AsyncCallbackExample {
    public static void main(String[] args) {
        AsyncOperation operation = new AsyncOperation();
        operation.performOperation(result -> System.out.println(result));
        
        System.out.println("Main thread continues...");
    }
}
```

## Callback with Java Consumer
The Consumer interface is a functional interface introduced in Java 8 as part of the java.util.function package. It represents an operation that accepts a single input argument and returns no result.

```java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
```

I'll refactor the previous callback example to use Java's Consumer interface. 

```java
class AsyncOperation {
    public void performOperation(Consumer<String> resultConsumer) {
        // Simulating an asynchronous operation
        new Thread(() -> {
            try {
                Thread.sleep(2000); // Simulate work
                resultConsumer.accept("Operation completed successfully"); // accept() method
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
}

public class ConsumerExample {
    public static void main(String[] args) {
        AsyncOperation operation = new AsyncOperation();
        
        Consumer<String> print = (result) -> System.out.println(result);
        operation.performOperation(print);
        
        System.out.println("Main thread continues...");
    }
}
```

## Related Design Patterns

* Command: Callbacks can be implemented as Command objects in scenarios where more flexibility or statefulness is required in the callback operation
* Observer: Callbacks can be seen as a more dynamic and lightweight form of the Observer pattern, with the ability to subscribe and unsubscribe callback functions dynamically