+++
title = "Create RAG with Spring AI - conversation"
series = ["Spring AI"]
date = "2026-11-25"
+++

{{< notice tip >}} 
GitHub project (conversation branch) : [https://github.com/adrien1212/RAG-springai-poc/tree/conversation](https://github.com/adrien1212/RAG-springai-poc/tree/conversation)
{{< /notice >}} 



Large language models are, by design, stateless: each request is processed independently, and the model retains no memory of prior exchanges. 

> Large language models (LLMs) are stateless, meaning they do not retain information about previous interactions. This can be a limitation when you want to maintain context or state across multiple interactions. To address this, Spring AI provides chat memory features that allow you to store and retrieve information across multiple interactions with the LLM. [^1]

Consequently, any multi-turn interaction requires an external mechanism that persists previous messages and supplies them to the model alongside each new prompt. In this article, we refer to this accumulated history of exchanges as a *conversation*, and we examine how Spring AI's chat memory abstractions can be used to implement it.

## Memory Storage

By default, Spring AI uses `InMemoryChatMemoryRepository`, but in this tutorial we will use `JdbcChatMemoryRepository` instead.

> JdbcChatMemoryRepository is a built-in implementation that uses JDBC to store messages in a relational database. It supports multiple databases out-of-the-box and is suitable for applications that require persistent storage of chat memory.

## Conversations table

In this blog series, the database schema is managed by Liquibase, so we have to disable Spring AI's automatic schema initialization and create the chat memory table ourselves (see the [configuration properties](https://docs.spring.io/spring-ai/reference/api/chat-memory.html#_configuration_properties)).

```properties
# Schema is owned by Liquibase, same rule as pgvector
spring.ai.chat.memory.repository.jdbc.initialize-schema=never
```

```sql
CREATE TABLE conversations (
    id          UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id    VARCHAR(255)  NOT NULL,
    title       VARCHAR(255)  NOT NULL,
    created_at  TIMESTAMPTZ   NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ   NOT NULL DEFAULT now()
);
CREATE INDEX idx_conversations_owner ON conversations (owner_id, updated_at DESC);

CREATE TABLE spring_ai_chat_memory (
    conversation_id  VARCHAR(36)  NOT NULL,
    content          TEXT         NOT NULL,
    type             VARCHAR(10)  NOT NULL CHECK (type IN ('USER', 'ASSISTANT', 'SYSTEM', 'TOOL')),
    "timestamp"      TIMESTAMP    NOT NULL,
    sequence_id      BIGINT       NOT NULL
);
CREATE INDEX spring_ai_chat_memory_conversation_id_timestamp_idx
    ON spring_ai_chat_memory (conversation_id, "timestamp");
CREATE INDEX spring_ai_chat_memory_conversation_id_sequence_id_idx
    ON spring_ai_chat_memory (conversation_id, sequence_id);
```

The `spring_ai_chat_memory` table matches the `schema-postgresql.sql` file shipped with Spring AI 2.0.1. Since version 2.0, messages are ordered by `sequence_id`, which the repository fills in, rather than by `timestamp`. If you manage the schema yourself, this column is required.

The two tables are linked by `conversation_id = conversations.id::text`, without a foreign key, following the same pattern as `documents` and `vector_store`. Deleting a conversation therefore requires clearing both tables in application code.

## Add the memory advisor to ChatClient

```java
@Bean
ChatMemory chatMemory(ChatMemoryRepository chatMemoryRepository) {
    return MessageWindowChatMemory.builder()
            .chatMemoryRepository(chatMemoryRepository)   // JdbcChatMemoryRepository (Postgres)
            .maxMessages(10)
            .build();
}

@Bean
ChatClient defaultChatClient(ChatClient.Builder builder, VectorStore vectorStore, ChatMemory chatMemory) {
    return builder
        .defaultAdvisors(
            MessageChatMemoryAdvisor.builder(chatMemory).build(),   // 1st: load history, save the raw question
            QuestionAnswerAdvisor.builder(vectorStore)...build()    // 2nd: unchanged
        )
        .build();
}
```

At this stage, the retention mechanism is in place: every exchange is persisted and replayed to the model. However, the advisor can only do so if each request is associated with a conversation identifier. The client must therefore create a conversation explicitly before sending its first question, and then reference that identifier in every subsequent request.

## Request flow

The interaction is split into two endpoints:

1. `POST /conversations` creates a row in the `conversations` table and returns its identifier.
2. `POST /chat` receives the `conversationId` and the `question` in the request body. The identifier is forwarded to `MessageChatMemoryAdvisor`, which loads the previous messages before the call and stores the new exchange afterwards.

{{<mermaid>}}
sequenceDiagram
    autonumber
    actor Client
    participant API as Spring Boot API
    participant DB as PostgreSQL
    participant Memory as MessageChatMemoryAdvisor
    participant QA as QuestionAnswerAdvisor
    participant LLM

    Note over Client,DB: 1. Open a new conversation
    Client->>API: POST /conversations
    API->>DB: INSERT INTO conversations
    DB-->>API: id
    API-->>Client: 201 Created (conversationId)

    Note over Client,LLM: 2. Ask a question within that conversation
    Client->>API: POST /chat (conversationId, question)
    API->>Memory: prompt + CONVERSATION_ID
    Memory->>DB: SELECT last 10 messages FROM spring_ai_chat_memory
    DB-->>Memory: history
    Memory->>QA: history + question
    QA->>DB: similarity search on vector_store
    DB-->>QA: relevant chunks
    QA->>LLM: history + context + question
    LLM-->>QA: answer
    QA-->>Memory: answer
    Memory->>DB: INSERT USER and ASSISTANT messages
    Memory-->>API: answer
    API-->>Client: 200 OK (answer)
{{</mermaid>}}

### Create a conversation

```java
public record ConversationResponse(UUID id, String title) {}

@PostMapping("/conversations")
@ResponseStatus(HttpStatus.CREATED)
ConversationResponse create() {
    // No authentication yet: each conversation gets a random owner identifier
    Conversation conversation = conversationRepository.save(
            new Conversation(UUID.randomUUID().toString(), "New conversation"));
    return new ConversationResponse(conversation.getId(), conversation.getTitle());
}
```

### Chat within a conversation

```java
public record ChatRequest(UUID conversationId, String question) {}

@PostMapping("/chat")
String chat(@RequestBody ChatRequest request) {
    return chatClient.prompt()
            .user(request.question())
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, request.conversationId().toString()))
            .call()
            .content();
}
```

The `ChatMemory.CONVERSATION_ID` parameter is the only link between the HTTP request and the stored history: two requests sharing the same identifier share the same context, while a new identifier starts from an empty history. Authentication is deliberately left out of this article; the `owner_id` column is nevertheless kept in the schema so that conversations can later be scoped to the authenticated user.

## Example

Here is what the database contains after one short exchange.

First, `POST /conversations` adds a row to our `conversations` table. Its `id` is the identifier the client sends with every following question.

![A new row in the conversations table](../images/spring_ai_conversation/conversation.png)

Then, each call to `POST /chat` with that identifier adds two rows to `spring_ai_chat_memory`: the user's question (`USER`) and the model's answer (`ASSISTANT`). Both rows have the conversation's `id` as their `conversation_id`, and `sequence_id` keeps them in order. On the next question, `MessageChatMemoryAdvisor` reads these rows back and sends them to the model as history.

![Messages stored in the spring_ai_chat_memory table](../images/spring_ai_conversation/spring_ai_chat_memory.png)

## Conclusion

Because LLMs are stateless, conversation support has to be built around the model. With Spring AI, this takes very little code: a `JdbcChatMemoryRepository` persists messages in PostgreSQL, a `MessageWindowChatMemory` limits how many of them are replayed, and a `MessageChatMemoryAdvisor` loads and saves the history on every call. The only thing the client has to do is create a conversation and send its identifier with each question.

Our RAG application can now handle follow-up questions. The next step is to add authentication, so that each user only sees and continues their own conversations.

[^1]: https://docs.spring.io/spring-ai/reference/api/chat-memory.html#_memory_in_chat_model