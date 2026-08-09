# Search and Vector Building Blocks

Author: Mike Nguyen - @mikeee (hey@mike.ee)

## Overview

This is a proposal surrounding the implementation of two new specialised state stores without bringing into
scope the deep integrations planned on implementation of these building blocks. Whilst the two building 
blocks share almost the same envelope, the method of requesting the records can vary significantly and should
be kept distinct shapes.

## Background

The current implementations of state stores do not accomodate the necessity to access data not only by 
key-value but an extended querying layer. New 'specialised' building blocks facilitate rapid development.

There is a specific need from AI workloads to not only query by key but also by 'relevant' records returned
through lexical/structured matching or geometric proximity for example.

### Scope boundary

This proposal is limited to storing, retrieving, and querying index-ready documents and vectors. It does not
include data-source connectors or application-level data transformation such as extraction, OCR, chunking,
embedding generation, or ingestion-pipeline orchestration. Callers must perform those steps before invoking these
building blocks; provider-internal indexing remains an implementation detail of the component.

[OmniVec](https://github.com/AzureCosmosDB/OmniVec) illustrates why this boundary is important: turning arbitrary
data sources into searchable vectors requires a purpose-built pipeline spanning source integration, content
processing, model execution, and destination management. Absorbing those concerns here would turn the building
blocks into a kitchen-sink search and vector platform rather than a focused, provider-neutral data access API.

## Providers in scope

| Provider | Lexical Search | Vector Search |
|----------|---|---|
| Meilisearch | ✓ | ✓ |
| Elasticsearch | ✓ | ✓ |
| Chroma | ✓ | ✓ |

Other providers with existing component implementations may be leveraged.

## HTTP API / Protos

The HTTP API mirrors the protobuf methods below. Path parameters provide `store_name` and, where applicable,
`index` or `collection`; the remaining request fields are supplied in the body.

### HTTP API

#### Search endpoints

| Method | Endpoint | Operation |
|--------|----------|-----------|
| `POST` | `/v1.0-alpha1/search/{storeName}/indexes/{index}` | Create an index |
| `GET` | `/v1.0-alpha1/search/{storeName}/indexes/{index}` | Get an index |
| `GET` | `/v1.0-alpha1/search/{storeName}/indexes` | List indexes |
| `DELETE` | `/v1.0-alpha1/search/{storeName}/indexes/{index}` | Delete an index |
| `POST` | `/v1.0-alpha1/search/{storeName}/indexes/{index}/documents` | Index documents |
| `POST` | `/v1.0-alpha1/search/{storeName}/indexes/{index}/documents/get` | Get documents by ID |
| `DELETE` | `/v1.0-alpha1/search/{storeName}/indexes/{index}/documents` | Delete documents |
| `POST` | `/v1.0-alpha1/search/{storeName}/indexes/{index}/query` | Search an index |

#### Vector endpoints

| Method | Endpoint | Operation |
|--------|----------|-----------|
| `POST` | `/v1.0-alpha1/vector/{storeName}/collections/{collection}` | Create a collection |
| `GET` | `/v1.0-alpha1/vector/{storeName}/collections/{collection}` | Get a collection |
| `GET` | `/v1.0-alpha1/vector/{storeName}/collections` | List collections |
| `DELETE` | `/v1.0-alpha1/vector/{storeName}/collections/{collection}` | Delete a collection |
| `POST` | `/v1.0-alpha1/vector/{storeName}/collections/{collection}/upsert` | Upsert vectors |
| `POST` | `/v1.0-alpha1/vector/{storeName}/collections/{collection}/get` | Get vectors |
| `DELETE` | `/v1.0-alpha1/vector/{storeName}/collections/{collection}/vectors` | Delete vectors |
| `POST` | `/v1.0-alpha1/vector/{storeName}/collections/{collection}/query` | Query vectors |
| `POST` | `/v1.0-alpha1/vector/{storeName}/collections/{collection}/batch-query` | Batch query vectors |

The document and vector get endpoints use `POST` because they are batch reads whose repeated IDs and retrieval
options are supplied in the request body. A `GET` request body is not consistently supported by HTTP clients,
proxies, or caches, while encoding large ID lists in a query string imposes practical URL-length limits. These
operations remain read-only.

### Search

```protobuf
rpc CreateIndexAlpha1(CreateIndexRequestAlpha1) returns (google.protobuf.Empty) {}
rpc GetIndexAlpha1(GetIndexRequestAlpha1) returns (GetIndexResponseAlpha1) {}
rpc ListIndexesAlpha1(ListIndexesRequestAlpha1) returns (ListIndexesResponseAlpha1) {}
rpc DeleteIndexAlpha1(DeleteIndexRequestAlpha1) returns (google.protobuf.Empty) {}
rpc IndexDocumentsAlpha1(IndexDocumentsRequestAlpha1) returns (IndexDocumentsResponseAlpha1) {}
rpc GetDocumentsAlpha1(GetDocumentsRequestAlpha1) returns (GetDocumentsResponseAlpha1) {}
rpc DeleteDocumentsAlpha1(DeleteDocumentsRequestAlpha1) returns (google.protobuf.Empty) {}
rpc SearchAlpha1(SearchRequestAlpha1) returns (SearchResponseAlpha1) {}

message SearchDocument {
  string id = 1;
  bytes content = 2;
  map<string, string> metadata = 10;
}

message SearchHit {
  SearchDocument document = 1;
  double score = 2;
  map<string, string> highlights = 3;
}

enum SortOrder {
  SORT_ORDER_UNSPECIFIED = 0;
  SORT_ORDER_ASC = 1;
  SORT_ORDER_DESC = 2;
}

message SortClause {
  string field = 1;
  SortOrder order = 2;
}

enum IndexAck {
  INDEX_ACK_UNSPECIFIED = 0;
  INDEX_ACK_QUEUED = 1;
  INDEX_ACK_DURABLE = 2;
}

message CreateIndexRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  // Component-specific index settings.
  map<string, string> metadata = 10;
}

message GetIndexRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  map<string, string> metadata = 10;
}

message GetIndexResponseAlpha1 {
  string index = 1;
  // Approximate document count. Providers that cannot supply this value
  // efficiently may return 0.
  uint64 document_count = 2;
  // Component-specific index properties.
  map<string, string> properties = 3;
}

message ListIndexesRequestAlpha1 {
  string store_name = 1;
  map<string, string> metadata = 10;
}

message ListIndexesResponseAlpha1 {
  repeated string indexes = 1;
}

message DeleteIndexRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  map<string, string> metadata = 10;
}

message IndexDocumentsRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  repeated SearchDocument documents = 3;
  map<string, string> metadata = 10;
}

message IndexDocumentsResponseAlpha1 {
  repeated string failed_ids = 1;
  IndexAck ack = 2;
}

message GetDocumentsRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  repeated string ids = 3;
  bool include_content = 4;
  map<string, string> metadata = 10;
}

message GetDocumentsResponseAlpha1 {
  repeated SearchDocument documents = 1;
}

message DeleteDocumentsRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  repeated string ids = 3;
  map<string, string> metadata = 10;
}

message SearchRequestAlpha1 {
  string store_name = 1;
  string index = 2;

  oneof query {
    string text = 3;
    google.protobuf.Struct native = 4;
  }

  google.protobuf.Struct filter = 5;
  uint32 top_k = 6;
  uint32 offset = 7;
  repeated string return_fields = 8;
  bool include_content = 9;
  // metadata = 10
  repeated string search_fields = 11;
  repeated SortClause sort = 12;
  repeated string highlight_fields = 13;

  map<string, string> metadata = 10;
}

message SearchResponseAlpha1 {
  repeated SearchHit hits = 1;
  uint64 total_hits = 2;
  string continuation_token = 3;
}
```

`UpdateIndexAlpha1` is intentionally omitted. Providers differ in which index settings can be changed after creation,
and some changes require rebuilding the index. A future update operation should define patch semantics, distinguish
mutable and immutable settings, and return an error rather than silently recreating an index or ignoring unsupported
changes.

### Vector

```protobuf
rpc CreateCollectionAlpha1(CreateCollectionRequestAlpha1) returns (google.protobuf.Empty) {}
rpc GetCollectionAlpha1(GetCollectionRequestAlpha1) returns (GetCollectionResponseAlpha1) {}
rpc ListCollectionsAlpha1(ListCollectionsRequestAlpha1) returns (ListCollectionsResponseAlpha1) {}
rpc DeleteCollectionAlpha1(DeleteCollectionRequestAlpha1) returns (google.protobuf.Empty) {}
rpc UpsertVectorsAlpha1(UpsertVectorsRequestAlpha1) returns (UpsertVectorsResponseAlpha1) {}
rpc DeleteVectorsAlpha1(DeleteVectorsRequestAlpha1) returns (google.protobuf.Empty) {}
rpc GetVectorsAlpha1(GetVectorsRequestAlpha1) returns (GetVectorsResponseAlpha1) {}
rpc QueryVectorsAlpha1(QueryVectorsRequestAlpha1) returns (QueryVectorsResponseAlpha1) {}
rpc BatchQueryVectorsAlpha1(BatchQueryVectorsRequestAlpha1) returns (BatchQueryVectorsResponseAlpha1) {}

enum DistanceMetric {
  // Use the metric configured for the collection.
  DISTANCE_METRIC_UNSPECIFIED = 0;
  // Cosine similarity. Higher scores indicate closer matches.
  DISTANCE_METRIC_COSINE = 1;
  // Dot-product similarity. Higher scores indicate closer matches.
  DISTANCE_METRIC_DOT_PRODUCT = 2;
  // Euclidean distance. Lower scores indicate closer matches.
  DISTANCE_METRIC_EUCLIDEAN = 3;
}

message CreateCollectionRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  // Component-specific collection settings, such as dimensions, distance
  // metric, named vectors, or index parameters.
  map<string, string> metadata = 10;
}

message GetCollectionRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  map<string, string> metadata = 10;
}

message GetCollectionResponseAlpha1 {
  string collection = 1;
  // Approximate number of records. Providers that cannot supply this value
  // efficiently may return 0.
  uint64 record_count = 2;
  // Component-specific collection properties.
  map<string, string> properties = 3;
}

message ListCollectionsRequestAlpha1 {
  string store_name = 1;
  map<string, string> metadata = 10;
}

message ListCollectionsResponseAlpha1 {
  repeated string collections = 1;
}

message DeleteCollectionRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  map<string, string> metadata = 10;
}

message SparseVector {
  repeated uint32 indices = 1;
  repeated float values = 2;
}

message NamedVector {
  repeated float values = 1;
  SparseVector sparse_values = 2;
}

message VectorRecord {
  string id = 1;
  repeated float values = 2;
  bytes payload = 3;
  SparseVector sparse_values = 5;
  map<string, NamedVector> named_vectors = 6;
  map<string, string> metadata = 10;
}

message VectorMatch {
  VectorRecord record = 1;
  // Unnormalized value of the effective metric for dense-only queries. For
  // hybrid queries, this is an unnormalized provider-specific fused relevance
  // score, where higher values indicate better matches.
  double score = 2;
}

message UpsertVectorsRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  repeated VectorRecord records = 3;
  map<string, string> metadata = 10;
}

message UpsertVectorsResponseAlpha1 {
  repeated string failed_ids = 1;
}

message DeleteVectorsRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  repeated string ids = 3;
  map<string, string> metadata = 10;
}

message GetVectorsRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  repeated string ids = 3;
  bool include_values = 4;
  map<string, string> metadata = 10;
}

message GetVectorsResponseAlpha1 {
  repeated VectorRecord records = 1;
}

message QueryVectorsRequestAlpha1 {
  string store_name = 1;
  string collection = 2;

  oneof query {
    VectorRecord vector = 3;
    string by_id = 4;
  }

  uint32 top_k = 5;
  google.protobuf.Struct filter = 6;
  bool include_values = 7;
  bool include_payload = 8;
  // Determines how score and score_threshold are interpreted. UNSPECIFIED
  // uses the metric configured for the collection.
  DistanceMetric metric = 9;
  // metadata = 10
  SparseVector sparse_query = 11;
  optional float alpha = 12;
  optional string vector_name = 13;
  // Inclusive cutoff for dense-only queries. A match is retained when its
  // score is greater than or equal to this value for COSINE and DOT_PRODUCT,
  // or less than or equal to this value for EUCLIDEAN. The value is not
  // normalized. score_threshold and sparse_query must not be set together.
  optional double score_threshold = 14;
  map<string, string> metadata = 10;
}

message QueryVectorsResponseAlpha1 {
  repeated VectorMatch matches = 1;
  // Effective metric used for dense scores. This is set to a concrete value
  // even when the request uses DISTANCE_METRIC_UNSPECIFIED.
  DistanceMetric metric = 2;
}

message BatchQueryVectorsRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  repeated QueryVectorsRequestAlpha1 queries = 3;
  map<string, string> metadata = 10;
}

message BatchQueryVectorsResponseAlpha1 {
  repeated QueryVectorsResponseAlpha1 results = 1;
}
```

`UpdateCollectionAlpha1` is intentionally omitted. Providers differ in which collection settings can be changed after
creation, and changes to settings such as vector dimensions or distance metric generally require rebuilding the
collection. A future update operation should define patch semantics, distinguish mutable and immutable settings, and
return an error rather than silently recreating a collection or ignoring unsupported changes.

#### Vector score semantics

For dense-only queries, `VectorMatch.score` and `score_threshold` use the metric's unnormalized value rather than
a provider-specific ranking score or a value normalized onto a common range. Components translate their provider's
native score representation to the following contract:

| Metric | Score | Better match | Inclusive threshold |
|--------|-------|--------------|---------------------|
| Cosine | `dot(a, b) / (norm(a) * norm(b))`, in `[-1, 1]` | Higher | `score >= score_threshold` |
| Dot product | `dot(a, b)`, unbounded | Higher | `score >= score_threshold` |
| Euclidean | `sqrt(sum((a_i - b_i)^2))`, in `[0, +inf)` | Lower | `score <= score_threshold` |

Scores are only comparable when they use the same metric and embedding model. If `metric` is
`DISTANCE_METRIC_UNSPECIFIED`, the collection's configured metric determines these semantics and the response's
`metric` field reports that effective metric.

`score_threshold` is not accepted with `sparse_query`: hybrid fusion produces a provider-specific score that cannot
be interpreted using `DistanceMetric` without defining a cross-provider fusion contract. A hybrid
`VectorMatch.score` is an unnormalized fused relevance score where higher values indicate a better match, but its
magnitude is provider-specific.

## Consumption Examples (Go)

```go
// Search
hits, err := client.SearchAlpha1(ctx, &client.SearchRequestAlpha1{
    StoreName: "meili-products",
    Index:         "products",
    Query:         &client.SearchRequestAlpha1Text{Text: "wireless headphones"},
    SearchFields:  []string{"title", "description"},
    TopK:          10,
})

// Dense vector query
matches, err := client.QueryVectorsAlpha1(ctx, &client.QueryVectorsRequestAlpha1{
    StoreName: "meili-docs",
    Collection:    "manuals",
    Query: &client.QueryVectorsRequestAlpha1_Vector{
        Vector: &client.VectorRecord{Values: dense},
    },
    Metric:         client.DistanceMetric_DISTANCE_METRIC_COSINE,
    TopK:           5,
    IncludePayload: true,
    ScoreThreshold: proto.Float64(0.6),
})

// Hybrid dense and sparse vector query
hybridMatches, err := client.QueryVectorsAlpha1(ctx, &client.QueryVectorsRequestAlpha1{
    StoreName: "meili-docs",
    Collection:    "manuals",
    Query: &client.QueryVectorsRequestAlpha1_Vector{
        Vector: &client.VectorRecord{Values: dense},
    },
    SparseQuery:    &client.SparseVector{Indices: sparseIdx, Values: sparseVal},
    Alpha:          proto.Float32(0.7),
    TopK:           5,
    IncludePayload: true,
})

// Batch query
batch, err := client.BatchQueryVectorsAlpha1(ctx, &client.BatchQueryVectorsRequestAlpha1{
    StoreName: "meili-docs",
    Collection:    "manuals",
    Queries: []*client.QueryVectorsRequestAlpha1{
        {Query: &client.QueryVectorsRequestAlpha1_Vector{Vector: &client.VectorRecord{Values: q1}}, TopK: 5},
        {Query: &client.QueryVectorsRequestAlpha1_Vector{Vector: &client.VectorRecord{Values: q2}}, TopK: 5},
    },
})
```