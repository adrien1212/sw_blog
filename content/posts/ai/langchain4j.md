+++
title = "Create RAG with LangChain4j (no spring required)"
series = ["AI"]
date = "2026-11-06"
+++

{{< notice tip >}} 
GitHub project : [https://github.com/adrien1212/rag-poc.git](https://github.com/adrien1212/rag-poc.git)
{{< /notice >}} 

LLMs are impressive for general knowledge, but ask one about your internal docs, your contracts or your product catalog and it'll confidently make things up. That's where **RAG (Retrieval-Augmented Generation)** comes in: instead of hoping the model knows the answer, you fetch the relevant pieces from your own documents and feed them as context. The model then answers based on *your* data, not its training set.

## What is LangChain4j

[LangChain4j](https://docs.langchain4j.dev/intro) is a Java library that makes working with LLMs less painful. Two things make it worth using:

- **Unified APIs**: every provider (OpenAI, Ollama, Mistral, etc) and every vector store has its own client, its own quirks. LangChain4j wraps them behind common interfaces like `ChatModel`, `EmbeddingModel`, `EmbeddingStore`. Want to swap Ollama for OpenAI later? Change one builder, done.
- **Ready-made building blocks**: document loaders, parsers, splitters, chat memory, tool calling, RAG — the usual stuff you'd end up writing yourself anyway.

These are exactly the pieces we need for our RAG.

## RAG Architecture Overview
### Ingestion
Everything starts with turning your raw documents into vectors that can be searched.

{{<mermaid>}}
flowchart LR
    A[("📁 Sources<br/>PDF, TXT, HTML, URL")] --> B

    subgraph Ingestion["Ingestion pipeline"]
        direction LR
        B["1. Load<br/><i>DocumentLoader</i>"] -- "raw file" --> C["2. Parse<br/><i>DocumentParser</i>"]
        C -- "Document" --> D["3. Split<br/><i>DocumentSplitter</i>"]
        D -- "TextSegment[]" --> E["4. Embed<br/><i>EmbeddingModel</i>"]
        E -- "Embedding[]" --> F[("5. Store<br/><i>EmbeddingStore</i>")]
    end
{{</mermaid>}}

1. **Load**: grab the raw content — from the filesystem, a URL, S3, wherever — via a `DocumentLoader`
2. **Parse**: extract text from whatever format you have (PDF, Word, HTML…) into a `Document` using a `DocumentParser`
3. **Split**: chop the `Document` into small pieces (`TextSegment`) with a `DocumentSplitter`. You want each chunk small enough to carry a single idea — that's what gives you precise search results
4. **Embed**: turn each chunk into a vector with an `EmbeddingModel`
5. **Store**: save those vectors (along with the original text) into an `EmbeddingStore` — could be in-memory, PgVector, Chroma, Qdrant, whatever fits your setup

### Retrieval
When a user asks a question, we embed it with the *same* model we used for ingestion (this is important — vectors from different models can't be compared). Then we do a similarity search to find the most relevant chunks and inject them into the prompt.

{{<mermaid>}}
 flowchart LR
     U(["👤 User"]) -- "question" --> A
     subgraph RA["RetrievalAugmentor"]
         direction LR
         subgraph CR["ContentRetriever"]
             A["1. Embed query<br/><i>EmbeddingModel</i>"] -- "Embedding" --> B[("2. Search<br/><i>EmbeddingStore</i>")]
         end
         B -- "top-K TextSegment[]" --> C["3. Inject segments<br/><i>ContentInjector</i>"]
     end
     M[("Chat history<br/><i>ChatMemory</i>")] -.-> D
     C -- "question + context" --> D["4. Generate<br/><i>ChatModel</i>"]
     D -- "answer" --> U
{{</mermaid>}}

1. **Embed the query**: same `EmbeddingModel` as ingestion — otherwise the vectors live in different spaces and the search is meaningless
2. **Search**: the `EmbeddingStore` finds the top-K closest segments (cosine similarity). You can set a `minScore` and filter on metadata. In LangChain4j, steps 1 & 2 are wrapped into `EmbeddingStoreContentRetriever`
3. **Augment the prompt**: the `RetrievalAugmentor` injects the retrieved segments into the user message, along with any conversation history from `ChatMemory`
4. **Generate**: the `ChatModel` (OpenAI, Ollama, Mistral…) produces an answer grounded in the retrieved context. The whole flow is driven by a simple Java interface via `AiServices`

## Implement the ingestion step

Let's build the ingestion pipeline piece by piece.

### Loading and parsing

Loading is about *where* a document comes from. LangChain4j has a `DocumentSource` interface for this, with three implementations :

- `ClassPathSource` from the classpath
- `FileSystemSource` from disk
- `UrlSource` over HTTP

Loading and parsing always go together: `DocumentLoader.load(...)` reads the bytes from a source, then passes them to a `DocumentParser` that knows how to extract text. We'll use `ApacheTikaDocumentParser` which auto-detects the format (PDF, DOCX, HTML, etc).

For this example, our docs live in `src/main/resources/docs`, so we load from the classpath

```java
Document document = DocumentLoader.load(
        ClassPathSource.from("docs/guide.pdf"),
        new ApacheTikaDocumentParser());
```

That gives us a single `Document`. To load a whole folder at once, LangChain4j has helper classes: `ClassPathDocumentLoader`, `FileSystemDocumentLoader` and `UrlDocumentLoader`.

```java
List<Document> documents = ClassPathDocumentLoader.loadDocuments(
        "docs",
        new ApacheTikaDocumentParser());

System.out.println(documents.size() + " documents loaded");
```
{{< notice tip >}} 
Use `loadDocumentsRecursively(...)` to include sub-folders, or pass a `PathMatcher` (e.g. `FileSystems.getDefault().getPathMatcher("glob:*.pdf")`) to filter by extension.
{{< /notice >}} 

### Split, Embed and Store

At this point we have a list of `Document` objects. Now we need to split them into chunks, embed each chunk into a vector, and store those vectors. Let's set up each component.

#### The Document Splitter

Why split at all? Two reasons: embedding models have a limited input size, and a small focused chunk gives a much more precise vector than a whole document. => Better vectors = better search results.

LangChain4j ships several splitters (`DocumentByParagraphSplitter`, `DocumentByLineSplitter`, `DocumentBySentenceSplitter`, etc) but the one you usually want is the **recursive** splitter. It tries the most natural boundary first — paragraphs — then falls back to lines, sentences, words. So you get chunks that actually contain coherent ideas instead of sentences cut in half ([source code](https://github.com/langchain4j/langchain4j/blob/main/langchain4j/src/main/java/dev/langchain4j/data/document/splitter/DocumentSplitters.java)):

```java
DocumentSplitter splitter = DocumentSplitters.recursive(
        500,  // max segment size (in characters)
        50);  // overlap between consecutive segments (in characters)
```

- **Segment size**: too small and you get precise matches but no context. Too large and you dilute the vector's meaning and eat up prompt tokens. A few hundred characters is a reasonable starting point — then tune for your documents
- **Overlap**: the end of one chunk is repeated at the start of the next, so information on chunk boundaries isn't lost. 10% of the segment size works well in practice

{{< notice tip >}}
Sizes are in characters by default. Pass a `TokenCountEstimator` as third argument to `recursive(...)` if you'd rather count in tokens.
{{< /notice >}}

#### The Embedding Model

I want to keep everything local and free, so I'm using [Ollama](https://ollama.com) for embeddings. The model is **`bge-m3`** — it's open-source, multilingual (handles French well, which matters for my docs), and supports inputs up to 8,192 tokens.

Pull it first:

```bash
ollama pull bge-m3
```

Then wire it up:

```java
EmbeddingModel embeddingModel = OllamaEmbeddingModel.builder()
        .baseUrl("http://localhost:11434")
        .modelName("bge-m3")
        .build();
```

{{< notice warning >}} 
You **must** use the same embedding model for ingestion and retrieval. Vectors from different models live in completely different spaces — comparing them gives garbage results.
{{< /notice >}} 

#### The Embedding Store

The embedding store is your vector database. It holds each vector alongside its `TextSegment` and can quickly find the closest vectors to a query.

LangChain4j supports [a bunch of them](https://docs.langchain4j.dev/integrations/embedding-stores) — PgVector, Chroma, Qdrant, Elasticsearch… For this example I'll stick with the [in-memory store](https://docs.langchain4j.dev/integrations/embedding-stores/in-memory) to keep things simple:

```java
EmbeddingStore<TextSegment> embeddingStore = new InMemoryEmbeddingStore<>();
```

#### Putting it all together

Now `EmbeddingStoreIngestor` wires everything up and runs the full pipeline (split, embed, store) in one call:

```java
EmbeddingStoreIngestor ingestor = EmbeddingStoreIngestor.builder()
        .documentSplitter(splitter)
        .embeddingModel(embeddingModel)
        .embeddingStore(embeddingStore)
        .build();

ingestor.ingest(documents);
```

For each document, it splits into segments, sends them to `bge-m3` for embedding, and stores the vectors. That's it for ingestion.

{{< notice warning >}}
Don't skip `.documentSplitter(...)`. Without it, the ingestor only splits if it finds a default splitter on the classpath (e.g. with `langchain4j-easy-rag`). Otherwise each document gets embedded as a single huge segment — bad search results and possibly exceeding the model's input limit.
{{< /notice >}} 

## Implement the retrieval step

Our documents are indexed. Now when a user asks a question, we need to find the relevant chunks, stuff them into the prompt, and let the LLM answer.

### The Chat Model

We're staying local with Ollama with `llama3.2` for generation

```bash
ollama pull llama3.2
```

```java
ChatModel chatModel = OllamaChatModel.builder()
        .baseUrl("http://localhost:11434")
        .modelName("llama3.2")
        .temperature(0.2)
        .build();
```

### The Content Retriever

The `ContentRetriever` is what actually finds relevant segments for a question. `EmbeddingStoreContentRetriever` handles the two-step process: embed the question, then search the store for the closest matches.

```java
ContentRetriever contentRetriever = EmbeddingStoreContentRetriever.builder()
        .embeddingStore(embeddingStore)
        .embeddingModel(embeddingModel)
        .maxResults(3)
        .minScore(0.6)
        .build();
```

- **`maxResults`**: how many segments end up in the prompt. Too few and you might miss information; too many and you add noise
- **`minScore`**: relevance goes from 0 to 1. This filters out segments that are technically "the closest" but still irrelevant. If nothing passes the threshold, no context is injected — which is actually what you want when the question is off-topic

### The Retrieval Augmentor

The `ContentRetriever` finds segments, but the `RetrievalAugmentor` orchestrates the whole flow: takes the user message, calls the retriever(s), and injects the results back into the message.

The default implementation (`DefaultRetrievalAugmentor`) runs five steps under the hood, based on the [docs](https://docs.langchain4j.dev/apidocs/dev/langchain4j/rag/DefaultRetrievalAugmentor.html):
1. **`QueryTransformer`** — optionally rewrites the question (e.g. making it self-contained using chat history). Default: passes it through unchanged
2. **`QueryRouter`** — picks which retriever(s) to query. Default: the one we provide
3. **`ContentRetriever`** — embeds the query and searches (our `EmbeddingStoreContentRetriever`)
4. **`ContentAggregator`** — merges and ranks results from all retrievers
5. **`ContentInjector`** — injects the segments into the user message

For a basic RAG, we just plug in our content retriever and keep the defaults five steps

```java
RetrievalAugmentor retrievalAugmentor = DefaultRetrievalAugmentor.builder()
        .contentRetriever(contentRetriever)
        .build();
```

{{< notice tip >}}
This is where you'd plug in more advanced techniques later: a `CompressingQueryTransformer` to rewrite follow-up questions, a `LanguageModelQueryRouter` to pick between several data sources, or a `ReRankingContentAggregator` to re-score results with a dedicated model.
{{< /notice >}}

### The Assistant interface

You don't call the chat model by hand. You just define a Java interface:

```java
interface Assistant {

    @SystemMessage("""
            You are an assistant who answers questions about our documentation.
            Answer only from the information provided. If it is not there, say you don't know.
            """)
    String chat(String question);
}
```

The `@SystemMessage` tells the model to stick to the provided context. That's what keeps hallucinations in check.

### Putting it all together

`AiServices` generates an implementation of our interface (a proxy) that wires the whole pipeline:

```java
Assistant assistant = AiServices.builder(Assistant.class)
        .chatModel(chatModel)
        .retrievalAugmentor(retrievalAugmentor)
        .build();

String answer = assistant.chat("How do I configure the embedding store?");
System.out.println(answer);
```

When you call `assistant.chat(...)`, here's what happens behind the scenes:

1. The question goes to the `RetrievalAugmentor`, which uses the `ContentRetriever` to find the top segments and injects them into the message
2. The system message + augmented user message are sent to the `ChatModel`
3. You get back the model's answer as a plain `String`

That's the whole thing. No framework magic, just Java interfaces and builders.

## Conclusion

We built a complete RAG in two phases:

- **Ingestion**: load documents, split them into chunks, embed them, store the vectors. `EmbeddingStoreIngestor` handles the whole chain
- **Retrieval**: for each question, find the relevant chunks, inject them into the prompt, let the LLM answer. `AiServices` hides the plumbing behind a simple interface

What I find nice about LangChain4j is that every component is an interface. Switching from Ollama to OpenAI, or from in-memory to PgVector, is swapping one implementation, the rest of the code doesn't change.

### Further reading
- [What is RAG — Spring AI](https://springdevpro.com/spring-ai/rag/what-is-rag/)
- [Spring AI vs LangChain4j](https://springdevpro.com/spring-ai/comparison/spring-ai-vs-langchain4j/)
- [LangChain4j documentation](https://docs.langchain4j.dev/intro)
