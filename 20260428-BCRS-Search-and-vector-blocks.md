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

| Provider | Lexical Search | Vector Search | Native queued acknowledgement | Streamed task completion |
|----------|---|---|---|---|
| Meilisearch | ✓ | ✓ | ✓ | Experimental |
| Elasticsearch | ✓ | ✓ | — | — |
| Chroma | ✓ | ✓ | — | — |

Other providers with existing component implementations may be leveraged.

Meilisearch exposes a distinct queued acknowledgement through its asynchronous
[tasks API](https://www.meilisearch.com/docs/capabilities/indexing/tasks_and_batches/async_operations). Providers such
as Elasticsearch that expose only a final write result response immediately.

### Experimental streamed task completion

Each Dapr sidecar instance maintains one long-lived
[Meilisearch task-change stream](https://www.meilisearch.com/docs/reference/api/async-task-management/stream-tasks-changes)
for each initialized Meilisearch search component. Meilisearch's experimental `tasksStreamingRoute` setting must be
enabled, and the component credentials require `tasks.get` permission. The enqueue request and stream must target the
same logical Meilisearch instance and task-ID space. The stream is filtered to document addition or update tasks and
terminal statuses where supported; this includes user-provided vectors, which Meilisearch stores in a document's
[`_vectors` field](https://www.meilisearch.com/docs/capabilities/hybrid_search/how_to/search_with_user_provided_embeddings).
The component dispatches only matching task IDs to requests waiting for completion. Requests never open their own
stream.

The task stream is a notification channel rather than the source of truth. The component reconnects with backoff,
keeps pending waiters across reconnects, and reconciles their task IDs using the task status API after registration,
after a reconnect, and immediately before applying a wait-timeout action. A time-bounded cache of terminal changes
also covers the enqueue-to-registration race. While waiters exist, a stream liveness timeout forces a reconnect and
reconciliation if the connection becomes silent or half-open. These point-in-time reads are recovery checks, not a
polling loop. Waiters are removed on a terminal change, wait timeout, or request cancellation.

This behavior is selected through `IndexingOptionsAlpha1` on document indexing and vector upsert requests; it is not
advertised as a component feature. If Meilisearch reports that `tasksStreamingRoute` is disabled, a
wait-for-completion request returns `FAILED_PRECONDITION` before records are enqueued; return-on-acceptance requests
remain available.

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
  map<string, string> metadata = 3;
}

message SearchHit {
  SearchDocument document = 1;
  // Unnormalized provider-specific relevance score. Higher values indicate a
  // more relevant match.
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
  // Never returned by a successful indexing write call.
  INDEX_ACK_UNSPECIFIED = 0;
  // The provider accepted the request for asynchronous processing. This does
  // not indicate that any document has been indexed successfully.
  INDEX_ACK_QUEUED = 1;
  // The provider completed the write. failed_items contains every
  // item-specific failure for this request.
  INDEX_ACK_COMPLETED = 2;
}

enum IndexingMode {
  // Return at the earliest durable acknowledgement boundary offered by the
  // provider. This may return QUEUED without eventual diagnostics.
  INDEXING_MODE_UNSPECIFIED = 0;
  // Wait for a final provider result.
  INDEXING_MODE_WAIT_FOR_COMPLETION = 1;
  // Return after the provider durably accepts the write for background
  // processing. No eventual result is exposed by this API.
  INDEXING_MODE_RETURN_ON_ACCEPTANCE = 2;
}

enum IndexingWaitTimeoutAction {
  INDEXING_WAIT_TIMEOUT_ACTION_UNSPECIFIED = 0;
  // Return INDEX_ACK_QUEUED and allow a durably queued provider task to
  // continue. Invalid when the provider has no queued acknowledgement.
  INDEXING_WAIT_TIMEOUT_ACTION_CONTINUE_ASYNC = 1;
  // Return DEADLINE_EXCEEDED. This does not guarantee cancellation of
  // provider-side work.
  INDEXING_WAIT_TIMEOUT_ACTION_FAIL_REQUEST = 2;
}

message IndexingOptionsAlpha1 {
  IndexingMode mode = 1;
  // Required positive limit for waiting on provider completion. Valid only with
  // INDEXING_MODE_WAIT_FOR_COMPLETION and must be shorter than the remaining
  // RPC context deadline when one is set.
  google.protobuf.Duration wait_timeout = 2;
  // Required with INDEXING_MODE_WAIT_FOR_COMPLETION.
  IndexingWaitTimeoutAction on_wait_timeout = 3;
}

// FailedItem is shared by document indexing and vector upserts.
message FailedItem {
  string id = 1;
  // error.code uses a canonical google.rpc.Code and must not be OK.
  // Provider-specific codes may be included in google.rpc.ErrorInfo details.
  google.rpc.Status error = 2;
}

message CreateIndexRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  // Component-specific index settings.
  map<string, string> metadata = 3;
}

message GetIndexRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  map<string, string> metadata = 3;
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
  map<string, string> metadata = 2;
}

message ListIndexesResponseAlpha1 {
  repeated string indexes = 1;
}

message DeleteIndexRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  map<string, string> metadata = 3;
}

message IndexDocumentsRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  repeated SearchDocument documents = 3;
  map<string, string> metadata = 4;
  IndexingOptionsAlpha1 options = 5;
}

message IndexDocumentsResponseAlpha1 {
  // Item-specific failures known at the acknowledgement boundary.
  repeated FailedItem failed_items = 1;
  // Always set to QUEUED or COMPLETED on a successful RPC.
  IndexAck ack = 2;
}

message GetDocumentsRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  repeated string ids = 3;
  bool include_content = 4;
  map<string, string> metadata = 5;
}

message GetDocumentsResponseAlpha1 {
  repeated SearchDocument documents = 1;
}

message DeleteDocumentsRequestAlpha1 {
  string store_name = 1;
  string index = 2;
  repeated string ids = 3;
  map<string, string> metadata = 4;
}

message SearchRequestAlpha1 {
  string store_name = 1;
  string index = 2;

  oneof query {
    string text = 3;
    google.protobuf.Struct native = 4;
  }

  google.protobuf.Struct filter = 5;
  // Maximum number of hits to return in this page.
  uint32 top_k = 6;
  // Opaque token returned by the previous SearchResponseAlpha1. Empty for the
  // first page.
  string continuation_token = 7;
  repeated string return_fields = 8;
  bool include_content = 9;
  repeated string search_fields = 10;
  repeated SortClause sort = 11;
  repeated string highlight_fields = 12;

  map<string, string> metadata = 13;
}

enum TotalHitsRelation {
  TOTAL_HITS_RELATION_UNSPECIFIED = 0;
  TOTAL_HITS_RELATION_EXACT = 1;
  TOTAL_HITS_RELATION_LOWER_BOUND = 2;
  TOTAL_HITS_RELATION_ESTIMATE = 3;
}

message SearchResponseAlpha1 {
  repeated SearchHit hits = 1;
  // Best-effort total. Omitted when the provider cannot supply one.
  optional uint64 total_hits = 2;
  // Opaque token for the next page. Empty when there are no more results.
  string continuation_token = 3;
  // Describes the accuracy of total_hits. UNSPECIFIED when total_hits is
  // omitted.
  TotalHitsRelation total_hits_relation = 4;
}
```

#### Search score semantics

`SearchHit.score` is an unnormalized provider-specific relevance score where higher values indicate a better match.
Components translate any distance-oriented native representation to preserve this higher-is-better contract. Scores
are comparable only among hits from the same query, provider, and index configuration. When an explicit sort is
requested, result order follows that sort and does not necessarily follow score order.

#### Search pagination semantics

The first search request omits `continuation_token`. When more results are available, the response returns an opaque
token that the caller supplies unchanged in the next request. An empty response token means that the provider
definitively reports no further results. Callers must not parse, construct, or modify tokens and must tolerate a
non-empty token leading to a final empty page when provider exhaustion can only be detected by reading the next page.

The component translates the token to the provider's strongest resumable pagination primitive, such as
`search_after`, a point-in-time handle, a provider-managed cursor, or an encoded offset for providers that expose only
offset pagination.

A continuation token is bound to the component, store, index, query, filter, sort, page size, projection, highlighting,
and component-declared result-affecting metadata. Transport metadata such as trace or request IDs is not bound. A
subsequent request repeats those fields unchanged and sets the token from the previous response. A malformed or
mismatched token returns `INVALID_ARGUMENT`. An expired provider cursor returns `FAILED_PRECONDITION` with
`google.rpc.ErrorInfo.reason` set to `SEARCH_CONTINUATION_EXPIRED`; callers restart pagination from the first page.

Pagination requires deterministic ordering. Components append the document ID as a stable tie-breaker when the
requested sort or provider relevance order is not unique. A provider-native query must not embed its own offset,
cursor, page size, or conflicting sort; such a request returns `INVALID_ARGUMENT`.

`total_hits` is best-effort and may be omitted. `total_hits_relation` distinguishes an exact total, a lower bound, and
an estimate. Providers may return it only on the first page, and concurrent writes can change a total reported on
later pages.

#### Write failure and acknowledgement semantics

`failed_items` is reserved for failures that the component can attribute to individual input IDs. Authentication,
transport, missing-index, malformed-request, and other request-wide failures are returned as the non-OK status of the
RPC rather than duplicated for every item. Components map native provider errors to canonical `google.rpc.Code`
values. Messages are developer-facing, must not be used as machine-readable values, and must not expose document
content or other sensitive provider details.

`IndexDocumentsAlpha1` and `UpsertVectorsAlpha1` are keyed upserts. Every input must have a non-empty,
caller-supplied ID, and IDs must be unique within a request. The RPC returns `INVALID_ARGUMENT` before invoking the
provider when an ID is empty or duplicated. Retrying the same request is therefore idempotent, including after an
indeterminate timeout.

`INDEXING_MODE_UNSPECIFIED` and `INDEXING_MODE_RETURN_ON_ACCEPTANCE` return at the earliest durable acknowledgement
boundary offered by the provider. A provider with a native asynchronous queue returns `INDEX_ACK_QUEUED` after
accepting the request for background processing. The response's `failed_items` contains only item failures known at
that boundary, and an empty list does not indicate that every document will eventually be indexed. The API
intentionally provides no operation ID or later diagnostics; callers receiving `INDEX_ACK_QUEUED` accept that eventual
provider failures are not observable through Dapr. Callers that require consistent final-result semantics across
providers must explicitly select `INDEXING_MODE_WAIT_FOR_COMPLETION`.

If the provider exposes only a final result, or completes the write before returning, the component returns
`INDEX_ACK_COMPLETED` with final `failed_items`. Return-on-acceptance is therefore not guaranteed to be non-blocking
across providers. Components must not create a process-local background queue to manufacture an earlier
acknowledgement.

`INDEXING_MODE_WAIT_FOR_COMPLETION` explicitly waits for provider processing to finish. For Meilisearch, the component
enqueues the document or vector write, registers the returned task ID with the sidecar's task-change dispatcher, and
waits for a terminal change from the shared stream:

- `succeeded` returns `INDEX_ACK_COMPLETED`; the IDs in `failed_items` failed and every requested ID not listed
  succeeded.
- `failed` returns a non-OK RPC status mapped from the task error.
- `canceled` returns a non-OK RPC status with canonical code `ABORTED`; `CANCELLED` remains reserved for cancellation
  of the Dapr RPC itself.

A provider failure that cannot be attributed to individual documents is returned as the non-OK status of the RPC.
Meilisearch task completion is atomic: a succeeded task has no eventual `failed_items`, while a failed task fails the
whole RPC. For Meilisearch, `failed_items` can therefore contain only failures identified before the task is enqueued.

When a provider reports a batch-level failure but cannot establish which items were applied, the non-OK RPC status
includes `google.rpc.ErrorInfo` with reason `INDEXING_OUTCOME_UNKNOWN`. A non-OK response does not by itself guarantee
that no write occurred; callers can safely retry the keyed upsert.

`wait_timeout` and `on_wait_timeout` are experimental and required with an explicit
`INDEXING_MODE_WAIT_FOR_COMPLETION`. The duration must be positive. If the RPC context has a deadline, the remaining
deadline must be longer than `wait_timeout` so the selected timeout action can be returned to the caller. Missing
fields, an insufficient context deadline, or either field used with another mode returns `INVALID_ARGUMENT` before
records are enqueued. Immediately before the timeout action, the component performs one task-status reconciliation
so a missed stream event cannot produce a false timeout.

If provider work is still not terminal when `wait_timeout` expires:

- `INDEXING_WAIT_TIMEOUT_ACTION_CONTINUE_ASYNC` returns `INDEX_ACK_QUEUED` and the provider task continues. This action
  requires a native durable queued acknowledgement; providers without one return `INVALID_ARGUMENT` before invoking
  the provider.
- `INDEXING_WAIT_TIMEOUT_ACTION_FAIL_REQUEST` returns `DEADLINE_EXCEEDED`. Provider-side work is not guaranteed to be
  canceled, so its final outcome may be unknown to the caller.

Client cancellation or an RPC context ending unexpectedly still takes precedence over the wait option. Components
must remove the request waiter promptly; provider-side work may continue.

The protobuf options message is the portable wire contract. SDKs may expose it using language-idiomatic functional
options; for example, a Go SDK can provide `WithReturnOnIndexAcceptance()` and
`WithWaitForIndexingCompletion(timeout, onTimeout)`.

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
  map<string, string> metadata = 3;
}

message GetCollectionRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  map<string, string> metadata = 3;
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
  map<string, string> metadata = 2;
}

message ListCollectionsResponseAlpha1 {
  repeated string collections = 1;
}

message DeleteCollectionRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  map<string, string> metadata = 3;
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
  SparseVector sparse_values = 4;
  map<string, NamedVector> named_vectors = 5;
  map<string, string> metadata = 6;
}

message VectorMatch {
  VectorRecord record = 1;
  // Unnormalized value of the effective metric for dense-only queries. Higher
  // is better for COSINE and DOT_PRODUCT; lower is better for EUCLIDEAN. For
  // hybrid queries, this is an unnormalized provider-specific fused relevance
  // score where higher values indicate better matches.
  double score = 2;
}

message UpsertVectorsRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  repeated VectorRecord records = 3;
  map<string, string> metadata = 4;
  IndexingOptionsAlpha1 options = 5;
}

message UpsertVectorsResponseAlpha1 {
  // Item-specific failures known at the acknowledgement boundary.
  repeated FailedItem failed_items = 1;
  // Always set to QUEUED or COMPLETED on a successful RPC.
  IndexAck ack = 2;
}

message DeleteVectorsRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  repeated string ids = 3;
  map<string, string> metadata = 4;
}

message GetVectorsRequestAlpha1 {
  string store_name = 1;
  string collection = 2;
  repeated string ids = 3;
  bool include_values = 4;
  map<string, string> metadata = 5;
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
  SparseVector sparse_query = 10;
  optional float alpha = 11;
  optional string vector_name = 12;
  // Inclusive cutoff for dense-only queries. A match is retained when its
  // score is greater than or equal to this value for COSINE and DOT_PRODUCT,
  // or less than or equal to this value for EUCLIDEAN. The value is not
  // normalized. score_threshold and sparse_query must not be set together.
  optional double score_threshold = 13;
  map<string, string> metadata = 14;
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
  map<string, string> metadata = 4;
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

In summary, higher scores are better for cosine similarity and dot product, while lower scores are better for
Euclidean distance. Hybrid fused relevance scores use higher-is-better semantics.

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
// Wait up to five seconds for a terminal task change. If the wait expires,
// return QUEUED and allow the provider task to continue.
indexing, err := client.IndexDocumentsAlpha1(ctx, &client.IndexDocumentsRequestAlpha1{
    StoreName: "meili-products",
    Index:     "products",
    Documents: []*client.SearchDocument{
        {Id: "headphones-123", Content: document},
    },
    Options: &client.IndexingOptionsAlpha1{
        Mode: client.IndexingMode_INDEXING_MODE_WAIT_FOR_COMPLETION,
        WaitTimeout: durationpb.New(5 * time.Second),
        OnWaitTimeout: client.IndexingWaitTimeoutAction_INDEXING_WAIT_TIMEOUT_ACTION_CONTINUE_ASYNC,
    },
})

// Fire-and-forget indexing. SDKs may expose this request option as a
// language-idiomatic functional option such as WithReturnOnIndexAcceptance().
queued, err := client.IndexDocumentsAlpha1(ctx, &client.IndexDocumentsRequestAlpha1{
    StoreName: "meili-products",
    Index:     "products",
    Documents: []*client.SearchDocument{
        {Id: "product-123", Content: anotherDocument},
    },
    Options: &client.IndexingOptionsAlpha1{
        Mode: client.IndexingMode_INDEXING_MODE_RETURN_ON_ACCEPTANCE,
    },
})

// Search
firstPage, err := client.SearchAlpha1(ctx, &client.SearchRequestAlpha1{
    StoreName: "meili-products",
    Index:         "products",
    Query:         &client.SearchRequestAlpha1Text{Text: "wireless headphones"},
    SearchFields:  []string{"title", "description"},
    TopK:          10,
})

// Request the next page by repeating the same search and passing the opaque
// token unchanged.
if err == nil && firstPage.ContinuationToken != "" {
    nextPage, err := client.SearchAlpha1(ctx, &client.SearchRequestAlpha1{
        StoreName:         "meili-products",
        Index:             "products",
        Query:             &client.SearchRequestAlpha1Text{Text: "wireless headphones"},
        SearchFields:      []string{"title", "description"},
        TopK:              10,
        ContinuationToken: firstPage.ContinuationToken,
    })
}

// Vector upserts use the same acknowledgement and wait options.
vectorIndexing, err := client.UpsertVectorsAlpha1(ctx, &client.UpsertVectorsRequestAlpha1{
    StoreName:  "meili-docs",
    Collection: "manuals",
    Records: []*client.VectorRecord{
        {Id: "manual-123", Values: dense, Payload: payload},
    },
    Options: &client.IndexingOptionsAlpha1{
        Mode:          client.IndexingMode_INDEXING_MODE_WAIT_FOR_COMPLETION,
        WaitTimeout:   durationpb.New(5 * time.Second),
        OnWaitTimeout: client.IndexingWaitTimeoutAction_INDEXING_WAIT_TIMEOUT_ACTION_FAIL_REQUEST,
    },
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