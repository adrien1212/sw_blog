+++
title = "Spring MCP Server and Client"
series = ["MCP, Spring"]
date = "2025-08-16"
+++

{{< notice Tip >}} 
- GitHub project : [https://github.com/adrien1212/spring-mcp-demo](https://github.com/adrien1212/spring-mcp-demo)
{{< /notice >}} 

## Introduction
Model Context Protocol (MCP) is a standard way for AI apps and agents to talk to external capabilities—databases, APIs, file systems, SaaS apps—without custom wiring each time.
Instead of baking ad-hoc HTTP calls into your prompts, MCP gives you a consistent handshake to discover what capabilities are available, for exemple "Create a new Client"

With MCP, you don’t manually hit REST endpoints. You express the intent in natural language, and the MCP client selects the right tool (and underlying API call) to execute it. For example:
```
curl -X POST http://localhost:8080/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Create client John Doe born 1990-01-01"}'
```

### What are “tools”?
In MCP, a tool is a named, typed capability exposed to the model. It’s the unit of action.
- Shape: a tool has a name, a description, a JSON schema for inputs, and a well-defined output.
- Behavior: tools do things—fetch records, write a file, send an email, create an invoice.
- Contract: the client advertises tools to the model; the model decides if/when to call them; the client executes the call and returns results back to the model.

```yml
{
  "name": "create_client",
  "description": "Create a client",
  "input_schema": {
    "type": "object",
    "properties": {
      "firstName": {"type": "string"},
      "lastName": {"type": "string"},
      "birthday": {"type": "string", "format": "date-time"}
    },
    "required": ["firstName", "lastName", "birthday"]
  }
}
```

### What are "server" and "client" ?
MCP follows a clear client–server split:
- MCP server: publishes typed tools and executes actions (e.g., persist a new client in the database).
- MCP client: runs alongside the model, discovers tools from one or more servers, negotiates capabilities, supplies context to the model, and—when the model requests it—invokes the right tool with the right arguments. You can use an off-the-shelf client like Claude Desktop, but in our case we’ll build our own client that connects to Ollama and talks to our MCP server.

{{<mermaid>}}
flowchart LR
  UI[User / App UI]
  Client[MCP Client]
  LLM[LLM]
  Server[MCP Server(s)]
  Systems[Your systems (DBs, APIs, files)]

  UI --> Client
  Client <-- "talks to" --> LLM
  Client <-- "tool requests / responses" --> Server
  Server --> Systems
{{</mermaid>}}

### Steps to complete the project
1. Start an Ollama instance and pull your model.
2. Implement the MCP server and expose.
3. Build the MCP client and connect it to both Ollama and the MCP server.

## Start Ollama instance
From `https://ollama.com/download` download and install. Then open a terminal and download a model `ollama pull mistral`. 

Ollama will expose an endpoint `http://localhost:11434/`

## Create MCP Server
{{< notice info >}} 
- Documentation : https://docs.spring.io/spring-ai/reference/api/mcp/mcp-overview.html
{{< /notice >}}

### Dependencies
*The Spring AI MCP (Model Context Protocol) Server Boot Starter provides auto-configuration for setting up an MCP server in Spring Boot applications. It enables seamless integration of MCP server capabilities with Spring Boot’s auto-configuration system.*

 ```xml
 <!-- Spring MVC-based SSE transport implementation -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-mcp-server-webmvc-spring-boot-starter</artifactId>
</dependency>
 ```

 ### Defining Custom Tools
We provide the following tools:
- **List clients** : returns all clients.
- **Create client** : adds a new client with firstName, lastName, and birthDate.
- **Get article by title** : retrieves a single article by its title.
- **Create command** : creates a new command with clientId, articleId, and quantity.

```java
@Component
public class ShopTools {
    @Tool(name = "list_clients", description = "Return all clients")
    public java.util.List<Client> listClients() {
        // Call Service/Repository
    }

    @Tool(
        name = "create_client",
        description = "Create a client with firstName, lastName, and birthDate (yyyy-MM-dd)"
    )
    public Client createClient(
        @ToolParam(description = "First name") String firstName,
        @ToolParam(description = "Last name") String lastName,
        @ToolParam(description = "Birth date in yyyy-MM-dd") String birthDate
    ) {
        // Call Service/Repository
    }

    @Tool(
    name = "get_client_by_name",
    description = "Return a single client by firstName and lastName (case-insensitive). Throws if not found."
    )
    public Client getClientByName(
        @ToolParam(description = "First name") String firstName,
        @ToolParam(description = "Last name") String lastName
    ) {
        // Call Service/Repository
    }

    @Tool(name = "get_article_by_title", description = "Retrieve a single article by its title (case-insensitive)")
    public Article getArticleByTitle(@ToolParam(description = "Article title") String title) {
        // Call Service/Repository
    }

    @Tool(
        name = "create_command",
        description = "Create a command with clientId, articleId, and quantity (>=1)"
    )
    public Command createCommand(
        @ToolParam(description = "Client ID") Long clientId,
        @ToolParam(description = "Article ID") Long articleId,
        @ToolParam(description = "Quantity (>=1)") Integer quantity
    ) {
        // Call Service/Repository
    }
}
```

**Note**: I tried using a single DTO (`CommandRequestModel`) for `create_command`, but the tool parameter binding didn’t pick it up. For now, we’ll stick to individual parameters.

### Exposing tools
The `mcp-server` application will use `ToolCallbackProvider` to register `@Tool` methods.

```java
@Configuration
public class McpConfig {
  @Bean
  public ToolCallbackProvider createToolCallbackProvider(
      ShopTools shopTools) {
    return MethodToolCallbackProvider.builder()
        .toolObjects(shopTools)
        .build();
  }
}
```

### Lancement
Once we start the application, we should see the log indicating how many tools were registered in the MCP server.
```
o.s.a.a.m.s.MpcServerAutoConfiguration   : Registered tools: 5 notification: true
```


## Create MCP Client
The MCP client is the entry point your app or users interact with. Under the hood, it forwards prompts to the local Ollama instance and invokes tools on the MCP server to carry out actions.
We’ll expose a REST endpoint at `http://localhost:8080/chat`, where clients can send requests like “Create client …”. The MCP client will choose the right tool and execute it.


### Dependencies
```xml
<dependency> <!-- To create http://localhost:8080/chat endpoint -->
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-mcp-client-spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
</dependency>
```

### Application.properties
```properties
server.port=8080

# Ollama Configuration
# Default Ollama base URL. Ensure Ollama is running on this port.
spring.ai.ollama.base-url=http://localhost:11434

# MCP Client Configuration
# Define a connection to your MCP server. 'myMcpServer' is an arbitrary name for this connection.
spring.ai.mcp.client.sse.connections.myMcpServer.url=http://localhost:8081
# The specific endpoint for sending messages to the MCP server.
spring.ai.mcp.client.sse.connections.myMcpServer.sse-message-endpoint=/mcp/messages
```

### MCP Client Configuration
```java
@Configuration
public class McpClientConfig {

  public static final OllamaModel OLLAMA_MODEL = OllamaModel.MISTRAL;

  @Bean
  public ChatClient createChatClient(ToolCallbackProvider tools) {

    var ollamaApi = new OllamaApi();
    var chatModel =
        OllamaChatModel.builder()
            .ollamaApi(ollamaApi)
            .defaultOptions(
                OllamaOptions.builder()
                    .model(OLLAMA_MODEL)
                    .temperature(0.4)
                    .build())
            .build();

    var chatClientBuilder = ChatClient.builder(chatModel);

    return chatClientBuilder.defaultTools(tools).build();
  }
}
```

- Sets the default Ollama model: `OLLAMA_MODEL = OllamaModel.MISTRAL` (i.e., use Mistral via your local Ollama).
- Builds an Ollama-backed chat model: Creates OllamaApi → OllamaChatModel with default options (model + temperature=0.4). Ollama must be running on localhost.
- Registers tool calling: The injected `ToolCallbackProvider` tools collects your `@Tool` methods.
- Exposes a ChatClient bean: returns a ready-to-use `ChatClient` you can inject into controllers/services (e.g., your /chat endpoint).

### Chat endpoint
```java
@RestController
public class McpClientController {

  @Autowired ChatClient chatClient;

  @PostMapping("/chat")
  public String tool(@RequestBody ToolRequest message) {
    System.out.println("Received message: " + message.message);

    PromptTemplate promptTemplate = new PromptTemplate(message.message);
    var response = chatClient.prompt(promptTemplate.create()).call().content();
    System.out.println("Response " + response);
    return response.toString();
  }
}

public class ToolRequest {

  public String message;
}
```

## Query example
### Simple exemple
Know we can perform following action
```
curl -X POST http://localhost:8080/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Create client John Doe born 1990-01-01"}'

```

### Complex example
**Note**: For this example, we switch the Ollama model to `OllamaModel.LLAMA3_1`. Make sure it’s installed locally by running `ollama pull llama3.1`.
```
curl -X POST -H 'Content-Type: application/json' -d \
'{"message":"create new command with client Adrien CAUBEL and article Pelle"}' \
http://localhost:8080/chat
```

With tool-calling enabled, the assistant will automatically chain the steps:
1. `get_client_by_name` to get the clientID
2. `get_article_by_title` to get the articleID
3. `create_command` to process the command with the right clientID and articleID
