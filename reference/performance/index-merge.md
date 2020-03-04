---
title: Access Tables with the Index Merge Method
category: reference
---

# Access Tables with the Index Merge Method

When the TiDB version is upgraded to 4.0, it allows to access tables with the `IndexMerge` method. With the new method, optimizer can use multiple indexes per table, and merge the results returned by the indexes. In some scenarios, the method makes the query more efficient by avoiding full table scan.

This document introduces the applicable scenarios, the actual use case and the method to enable the `IndexMerge`.

## Applicable scenarios

For each table involved in the SQL query, the TiDB optimizer during the physical optimization used to choose one from the following three access methods based on the cost estimation:

- `TableScan`: Scan the table data, with `_tidb_rowid` as the key.
- `IndexScan`: Scan the index data, with the value of the index column as the key.
- `IndexLookUp`: Get the `_tidb_rowid` set from the index, with the value of the index column as the key. Then retrieve the corresponding data rows of the tables.

These methods allow to use one index per table only. In some cases, the execution produced is not optimal. For example:

{{< copyable "sql" >}}

```sql
create table t(a int, b int, c int, unique key(a), unique key(b));
explain select * from t where a = 1 or b = 1;
```

The filter conditions in the query expression are connected by `OR`. Within the limitations of one index per table only, `a = 1` cannot be pushed down to the query `a`; neither can `b = 1` be pushed down to the query `b`. To ensure the correct result, the execution plans of `TableScan` are generated after the query:

```
+---------------------+----------+-----------+------------------------------------------------------------+
| id                  | count    | task      | operator info                                              |
+---------------------+----------+-----------+------------------------------------------------------------+
| TableReader_7       | 8000.00  | root      | data:Selection_6                                           |
| └─Selection_6       | 8000.00  | cop[tikv] | or(eq(test.t.a, 1), eq(test.t.b, 1))                       |
|   └─TableScan_5     | 10000.00 | cop[tikv] | table:t, range:[-inf,+inf], keep order:false, stats:pseudo |
+---------------------+----------+-----------+------------------------------------------------------------+
```

Full table scans are inefficient in the cases of high data volume of `t`. The query only returns two rows. To handle such scenarios, TiDB introduces `IndexMerge` to access tables.

## Actual use cases

`IndexMerge` allows the optimizer to use multiple indexes per table, and merge the results returned by the indexes before further operation.Take the query above as an example, the execution plans generated will be:

```
+--------------------+-------+-----------+---------------------------------------------------------------+
| id                 | count | task      | operator info                                                 |
+--------------------+-------+-----------+---------------------------------------------------------------+
| IndexMerge_11      | 2.00  | root      |                                                               |
| ├─IndexScan_8      | 1.00  | cop[tikv] | table:t, index:a, range:[1,1], keep order:false, stats:pseudo |
| ├─IndexScan_9      | 1.00  | cop[tikv] | table:t, index:b, range:[1,1], keep order:false, stats:pseudo |
| └─TableScan_10     | 2.00  | cop[tikv] | table:t, keep order:false, stats:pseudo                       |
+--------------------+-------+-----------+---------------------------------------------------------------+
```

The structure of the `IndexMerge` execution plan is similar to that of the `IndexLookUp` since both can be divided into index scans and full table scans. However, index scans of the `IndexMerge` may include multiple `IndexScan`. When the primary key index of the table is an integer, index scans may include one `TableScan`. For example:

{{< copyable "sql" >}}

```sql
create table t(a int primary key, b int, c int, unique key(b));
```

```
Query OK, 0 rows affected (0.01 sec)
```

{{< copyable "sql" >}}

```sql
explain select * from t where a = 1 or b = 1;
```

```
+--------------------+-------+-----------+---------------------------------------------------------------+
| id                 | count | task      | operator info                                                 |
+--------------------+-------+-----------+---------------------------------------------------------------+
| IndexMerge_11      | 2.00  | root      |                                                               |
| ├─TableScan_8      | 1.00  | cop[tikv] | table:t, range:[1,1], keep order:false, stats:pseudo          |
| ├─IndexScan_9      | 1.00  | cop[tikv] | table:t, index:b, range:[1,1], keep order:false, stats:pseudo |
| └─TableScan_10     | 2.00  | cop[tikv] | table:t, keep order:false, stats:pseudo                       |
+--------------------+-------+-----------+---------------------------------------------------------------+
4 rows in set (0.01 sec)
```

Note that `IndexMerge` is used only when optimizer use multiple indexes per table. If the condition in the query expression is `a = 1 and b = 1`, the optimizer uses the index `a` or the index `b`, instead of the `IndexMerge`, to access tables.

## Enable the `IndexMerge`

`IndexMerge` is disabled by default. Enable the `IndexMerge` in one of two ways:

- Set the `tidb_enable_index_merge` system variable to `1`;
- Use the SQL Hint [`USE_INDEX_MERGE`](/reference/performance/optimizer-hints.md#use_index_merget1_name-idx1_name--idx2_name-) in the query.

    > **Note:**
    >
    > The SQL Hint has higher precedence than the system variable.