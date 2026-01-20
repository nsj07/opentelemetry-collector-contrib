# Google Cloud Spanner Receiver - Gaps and Roadmap Report

## 1. Feature Gaps

### Missing Metrics
The receiver currently collects data from several `SPANNER_SYS` tables, but several introspection tables are not yet covered. Adding these would provide a more complete view of Spanner performance.

*   **Oldest Active Queries (`SPANNER_SYS.OLDEST_ACTIVE_QUERIES`)**: Critical for identifying currently running queries that may be blocked or taking too long.
*   **Active Partitioned DMLs (`SPANNER_SYS.ACTIVE_PARTITIONED_DMLS`)**: Necessary for monitoring long-running batch DML operations.
*   **Table Operations Stats (`SPANNER_SYS.TABLE_OPERATIONS_STATS_1HOUR`)**: Provides insights into table and index usage (inserts, updates, deletes, reads).
*   **Column Operations Stats (`SPANNER_SYS.COLUMN_OPERATIONS_STATS_1HOUR`)**: Provides granular usage data at the column level.
*   **Hot Split Statistics (`SPANNER_SYS.HOT_SPLIT_STATS_1MINUTE`)**: Essential for diagnosing "hotspotting" issues where specific splits are overloaded.

### Missing Signal: Traces
The receiver is currently **Metrics-only**.
*   **Gap:** There is no support for exporting **Traces**.
*   **Opportunity:** While Spanner is a managed service, the receiver could generate **Spans** for "Slow Queries" or "Long Running Transactions".
    *   **Source:** `SPANNER_SYS.OLDEST_ACTIVE_QUERIES` contains `START_TIMESTAMP` and query text. This can be converted into Spans representing active, long-running queries.
    *   **Benefit:** This would allow users to see slow queries in their Trace backend (e.g., Jaeger, Tempo) alongside their application traces, providing immediate context for database latency.

## 2. Documentation & Compliance Gaps

*   **`metadata.yaml` vs `metrics.yaml`**: The component uses a custom `internal/metadataconfig/metrics.yaml` to define metrics dynamically. However, the standard `metadata.yaml` file (used by `mdatagen` for documentation and helper code generation) is empty of metric definitions.
    *   **Impact:** Automatic documentation generation is incomplete. The metrics are not listed in the official collector registry in the standard format.
*   **README:** While the README is decent, it lacks a detailed table of all exported metrics (Name, Type, Unit, Description) because it's not being auto-generated from `metadata.yaml`.

## 3. Testing Gaps

*   **No Integration Tests:** There are no `integration_test.go` files or tests that run against a real Google Cloud Spanner instance or the Spanner Emulator.
    *   **Risk:** The receiver is tested only with mocks. Changes to the `SPANNER_SYS` schema or actual behavior of the Spanner Go client might break the receiver without being caught in CI.

---

# Proposed Roadmap

This roadmap is prioritized to first ensure stability and compliance, then expand feature coverage.

### Phase 1: Foundation & Compliance
1.  **Standardize Metadata**:
    *   Migrate metric definitions from `internal/metadataconfig/metrics.yaml` to the standard `metadata.yaml` format.
    *   Use `mdatagen` to generate the metric definitions and documentation. This ensures the component follows the repository standards and has up-to-date documentation.
2.  **Add Integration Tests**:
    *   Implement an integration test suite using the **Google Cloud Spanner Emulator**.
    *   Verify that the receiver can successfully connect to the emulator, query `SPANNER_SYS` tables, and produce metrics.

### Phase 2: Metric Expansion
1.  **Add `OLDEST_ACTIVE_QUERIES` Metrics**: Implement collection for oldest active queries (count, duration).
2.  **Add `TABLE_OPERATIONS_STATS`**: Add metrics for table/index usage.
3.  **Add `HOT_SPLIT_STATS`**: Add metrics for detecting hot splits.

### Phase 3: Trace Support (New Feature)
1.  **Implement Trace Receiver Capability**:
    *   Update `factory.go` to support `CreateTracesReceiver`.
2.  **"Slow Query" Tracing**:
    *   Create a logic to convert rows from `SPANNER_SYS.OLDEST_ACTIVE_QUERIES` (and potentially `query_stats_top_minute` for historical aggregates) into **Spans**.
    *   Attributes should include `db.statement`, `db.user`, `spanner.session_id`, and duration.
