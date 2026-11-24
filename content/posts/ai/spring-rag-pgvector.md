+++
title = "Create RAG with Spring AI - vector database"
series = ["Spring AI"]
date = "2026-11-15"
+++

{{< notice tip >}} 
GitHub project (add-pgstore branch) : [https://github.com/adrien1212/RAG-springai-poc/tree/add-pgstore](https://github.com/adrien1212/RAG-springai-poc/tree/add-pgstore)
{{< /notice >}} 

In the previous article, we created a RAG with in-memory storage. In this post, we look at the persistence mechanism for a RAG system.


## What we persist
We need to persist:
- **document metadata**: filename, sha256, status, embedding model, etc.
- **vectors**: the chunk content, its metadata and the embedding vector
- optionally, **the file itself** in S3/MinIO storage

### Document table

A few columns worth explaining:
- **`content_hash`** (sha256): used to detect duplicates. If we have already indexed this exact file, we skip it
- **`status`** (`PENDING`, `INDEXING`, `INDEXED` / `FAILED`): tracks where the document is in the ingestion pipeline, useful for retries and monitoring
- **`embedding_model`**: we store which model produced the vectors, because if we switch models later the existing embeddings are no longer compatible and need to be regenerated

```sql
CREATE TABLE documents (
    id               UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    filename         VARCHAR(512)  NOT NULL,
    mime_type        VARCHAR(255),
    size_bytes       BIGINT        NOT NULL,
    content_hash     VARCHAR(64)      NOT NULL,
    storage_path     VARCHAR(1024) NOT NULL,
    status           VARCHAR(20)   NOT NULL DEFAULT 'PENDING',
    error_message    TEXT,
    chunk_count      INTEGER,
    embedding_model  VARCHAR(100),
    created_at       TIMESTAMPTZ   NOT NULL DEFAULT now(),
    indexed_at       TIMESTAMPTZ,

    CONSTRAINT uq_documents_hash UNIQUE (content_hash),
    CONSTRAINT ck_documents_status
        CHECK (status IN ('PENDING', 'INDEXING', 'INDEXED', 'FAILED'))
);
```

### Vectors table

A vector database (or vector DB) stores data in a vector format to allow similarity searches. Each piece of text is converted by the embedding model into a dense vector (1024D in this case). When a query is made, the user's question is embedded and the most similar vectors are returned. 

Traditional search engines usually search based on keywords, but assume that the user and data will be in the same language. Vector DBs make the search more meaningful by looking at the similarity between vectors. The naive solution would be to search through all vectors (O(n)); however, this doesn’t scale well. An algorithm called HNSW (Hierarchical Navigable Small World) can perform Approximate Nearest Neighbour searches in O(log n) time. 

Spring AI abstracts this away with a VectorStore abstraction (see [PgVector Docs](https://docs.spring.io/spring-ai/reference/api/vectordbs/pgvector.html)). When we call `vectorStore.add(chunks)`, Spring will embed the chunks and persist them to this table. In the previous article this was stored in memory. Now it will be stored in the table below.

```sql
CREATE TABLE vector_store (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content    TEXT,
    metadata   JSONB,
    embedding  VECTOR(1024)
);

CREATE INDEX idx_vector_store_embedding
    ON vector_store USING hnsw (embedding vector_cosine_ops);

CREATE INDEX idx_vector_store_metadata
    ON vector_store USING gin (metadata jsonb_path_ops);
```

#### Go deeper 
{{< notice tip >}} 
- https://springdevpro.com/spring-ai/source-code/vector-store/
{{< /notice >}} 


The `VectorStore` [interface](https://docs.spring.io/spring-ai/docs/current/api/org/springframework/ai/vectorstore/VectorStore.html) defines the following methods:

```java
public interface VectorStore extends DocumentWriter, VectorStoreRetriever {
    void add(List<Document> documents);
    Optional<Boolean> delete(List<String> idList);
    List<Document> similaritySearch(String query); // From VectorStoreRetriever
    List<Document> similaritySearch(SearchRequest request); // From VectorStoreRetriever
}
```

and has several provider implementations (following the Strategy / Adapter design pattern):

{{<mermaid>}}
classDiagram
    direction BT
    class VectorStore {
        +add(List~Document~)
        +delete(List~String~)
        +similaritySearch(String) List~Document~
        +similaritySearch(SearchRequest) List~Document~
    }

    PgVectorStore ..|> VectorStore
    ElasticsearchVectorStore ..|> VectorStore
    OpenSearchVectorStore ..|> VectorStore
    MilvusVectorStore ..|> VectorStore
    PineconeVectorStore ..|> VectorStore
    WeaviateVectorStore ..|> VectorStore
    ChromaVectorStore ..|> VectorStore
    QdrantVectorStore ..|> VectorStore
{{</mermaid>}}


You can find the available properties in the [documentation](https://docs.spring.io/spring-ai/reference/api/vectordbs/pgvector.html):

```properties
# https://docs.spring.io/spring-ai/reference/api/vectordbs/pgvector.html
spring.ai.vectorstore.pgvector.initialize-schema=false
spring.ai.vectorstore.pgvector.schema-name=public
spring.ai.vectorstore.pgvector.dimensions=1024
#spring.ai.vectorstore.pgvector.index-type=HNSW  done by 003-vector-store.sql
spring.ai.vectorstore.pgvector.distance-type=COSINE_DISTANCE
```

## Storing and embedding a document

{{<mermaid>}}
sequenceDiagram
    autonumber
    actor Client
    participant Controller
    participant StorageService
    participant DocumentRepository
    participant VectorStore
    participant EmbeddingModel

    Client->>Controller: POST multipart file
    Controller->>StorageService: storeAndEmbed(file)
    activate StorageService
    Note over StorageService: Transaction begins

    StorageService->>StorageService: sha256(file bytes)
    StorageService->>DocumentRepository: findByContentHash(sha256)
    DocumentRepository-->>StorageService: Optional SourceDocument

    alt Document already exists
        StorageService-->>Controller: return (skip)
    else New document
        StorageService->>DocumentRepository: save(new SourceDocument)
        DocumentRepository-->>StorageService: sourceDocument
        StorageService->>StorageService: markIndexing()

        StorageService->>StorageService: read file and split into chunks
        StorageService->>StorageService: put document_id in chunk metadata

        StorageService->>VectorStore: add(chunks)
        VectorStore->>EmbeddingModel: embed(chunks)
        EmbeddingModel-->>VectorStore: vectors
        VectorStore-->>StorageService: stored in the PgVector by VectorStore

        StorageService->>StorageService: markIndexed(chunkCount, embeddingModel)
        StorageService->>DocumentRepository: save(sourceDocument)

        opt Exception thrown
            Note over StorageService: caught and swallowed (empty catch)
        end

        StorageService-->>Controller: return
    end

    Note over StorageService: Transaction commits
    deactivate StorageService
    Controller-->>Client: 200 OK
{{</mermaid>}}

1. Check if the document already exists in the database by comparing the sha256
2. Store the document data in the `documents` table
3. Parse (`TikaDocumentReader`), split (`TokenTextSplitter`) and embed/store (`vectorStore.add(chunks)`) into PgVector
    - for each chunk, we add `document_id` to the metadata, so when we want to remove a document, we can easily find all related vectors

```java
@Transactional
public void storeAndEmbed(MultipartFile multipartFile) {
    // 1. Does the document exist?
    String sha256 = null;
    try {
        sha256 = sha256(multipartFile.getBytes());
    } catch (IOException e) {
        throw new RuntimeException(e);
    }

    Optional<SourceDocument> existing = documentRepository.findByContentHash(sha256);
    if(existing.isPresent())
        return;

    // 2. Store the document data in the table `documents`
    SourceDocument sourceDocument = documentRepository.save(new SourceDocument(multipartFile.getOriginalFilename(), multipartFile.getContentType(), multipartFile.getSize(), sha256, "path to the S3/Minio"));
    sourceDocument.markIndexing();

    // 3. Parse, Split, Embed and Store
    try {
        DocumentTransformer splitter = TokenTextSplitter.builder().withChunkSize(400).withMinChunkSizeChars(200).build();
        List<Document> chunks = splitter.apply(new TikaDocumentReader(multipartFile.getResource()).read());
        chunks.forEach(c -> {
            c.getMetadata().put("document_id", sourceDocument.getId());
        });
        vectorStore.add(chunks);
        sourceDocument.markIndexed(chunks.size(), embeddingModel);
        documentRepository.save(sourceDocument);
    } catch (Exception e) {

    }
    return;
}
```

| Column      | Example value                                                                  |
|-------------|--------------------------------------------------------------------------------|
| `id`        | `bec61226-bc42-446a-9f06-d92ffc99016f`                                         |
| `content`   | "Dispositions prises pour la sécurité. Secouristes : le nombre de participants étant limité à 500 coureurs…" |
| `metadata`  | `{"source": "Dispositions prises pour la sécurité.pdf", "chunk_index": 0, "document_id": "3612c0d9-…"}` |
| `embedding` | `[-0.0142, -0.0120, -0.0584, 0.0171, -0.0287, …]`                              |

## Conclusion

By moving from an in-memory store to PgVector, our RAG now keeps its data across restarts. The `documents` table tracks each file (hash, status, embedding model), which lets us skip duplicates, monitor ingestion and know when re-embedding is needed. The `vector_store` table holds the chunks and their embeddings, indexed with HNSW for fast similarity search.

Thanks to Spring AI's `VectorStore` abstraction, the code that adds and searches vectors stays the same: switching from in-memory to PgVector is mostly a matter of adding a dependency and a few properties. The same code could run on Elasticsearch, Qdrant or any other supported provider.
