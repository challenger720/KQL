
# Microsoft Sentinel & KQL Cheatsheet

A comprehensive reference guide for Kusto Query Language (KQL) operators, Microsoft Sentinel data lake integration, and query best practices.

---

## 1. Core KQL Operators

### The table reference ###
The simplest queries you can enter and run are the table references
```
SecurityEvent  
SecurityAlert
```

### The `where` Operator

The `where` operator filters a table to the subset of rows that satisfy a specified predicate.

```kusto
// Filter by time range (e.g., older than 1 day)
SecurityEvent | where TimeGenerated > ago(1d)

// Combine multiple conditions with 'and'
SecurityEvent | where TimeGenerated > ago(1h) and EventID == "4624"

// Multiple filter lines (implicit 'and')
SecurityEvent 
| where TimeGenerated > ago(1h)
| where EventID == 4624
| where AccountType =~ "user"

// Filter using the 'in' operator for multiple values
SecurityEvent | where EventID in (4624, 4625)

```

### The `search` Operator

The `search` operator provides a multi-table and multi-column search experience. *Note: When compared to the `where` operator, `search` is less efficient.*

```kusto
// Search for "err" across all tables
search "err"

// Search for "err" in specific tables and wildcard table patterns
search in (SecurityEvent, SecurityAlert, A*) "err"

```

### The `let` Statement

`let` statements allow for declaring and reusing variables, defining dynamic tables or lists, and improving query modularity and reuse.

```kusto
// Declaring scalar variables
let timeOffset = 7d;
let discardEventId = 4688;
SecurityEvent 
| where TimeGenerated > ago(timeOffset*2) and TimeGenerated < ago(timeOffset)
| where EventID != discardEventId

// Declaring dynamic tables or lists using datatable
let suspiciousAccounts = datatable(account: string) [
    @"\administrator",
    @"NT AUTHORITY\SYSTEM"
];
SecurityEvent | where Account in (suspiciousAccounts)

// Improving modularity by storing summarized results in a variable
let LowActivityAccounts = 
    SecurityEvent 
    | summarize cnt = count() by Account 
    | where cnt < 1000;
LowActivityAccounts | where Account contains "SQL"

```

### The `extend` Operator

The `extend` operator creates calculated columns and appends the new columns to the result set.

```kusto
SecurityEvent 
| where ProcessName != "" and Process != ""
| extend StartDir = substring(ProcessName, 0, string_size(ProcessName) - string_size(Process))

```

### The `order by` Operator

Sorts the rows of the input table by one or more columns in ascending or descending order.

```kusto
SecurityEvent 
| where ProcessName != "" and Process != ""
| extend StartDir = substring(ProcessName, 0, string_size(ProcessName) - string_size(Process))
| order by StartDir desc, Process asc

```

### The `project` Operators

Manage and reshape columns in your query output using the project family:

| Operator | Description |
| --- | --- |
| `project` | Select the columns to include, rename or drop, and insert new computed columns. |
| `project-away` | Select what columns from the input to exclude from the output. |
| `project-keep` | Select what columns from the input to keep in the output. |
| `project-rename` | Select the columns to rename in the resulting output. |
| `project-reorder` | Set the column order in the resulting output. |

*Example combining operations:*

```kusto
SecurityEvent 
| where ProcessName != "" and Process != ""
| extend StartDir = substring(ProcessName, 0, string_size(ProcessName) - string_size(Process))
| order by StartDir desc, Process asc
| project-away ProcessName

```

---

## 2. Visualizations and Time Series

The `render` operator generates visual representations of query results.

```kusto
// Render a bar chart summarizing event count by account
SecurityEvent 
| summarize count() by Account 
| render barchart

// Create a time series using bin() and render a timechart
SecurityEvent 
| summarize count() by bin(TimeGenerated, 1d) 
| render timechart

```

### Supported Visualization Types

* `areachart`
* `barchart`
* `columnchart`
* `piechart`
* `scatterchart`
* `timechart`

---

## 3. Microsoft Sentinel Data Lake

The Microsoft Sentinel data lake allows you to store and analyze high-volume, low-fidelity logs (such as firewall or DNS data, asset inventories, and historical records) for up to 12 years. Because storage and compute are decoupled, you can query the same copy of data using multiple tools without moving or duplicating it.

### KQL Interactive Queries vs. KQL Jobs

* **KQL Interactive Queries:** Run interactive KQL queries directly on the data lake to investigate and respond using historical data, enrich investigations with high-volume logs, and correlate asset and log data.
* **KQL Jobs:** Run queries against the data lake tier and promote results to the analytics tier to enable incident investigation, log correlation, scheduled recurring enrichment, and automated insights for threat hunting or compliance.

---

## 4. Query Considerations and Limitations

* **Single Workspace Scope:** Queries run against a single workspace; ensure you select the correct workspace before running your query.
* **Billing & Pricing:** Executing KQL queries on the data lake incurs charges based on query billing meters.
* **Data Retention:** Review data ingestion and table retention policies before setting query time ranges to verify data availability.
* **Performance:** Queries against the data lake have lower performance than queries on the analytics tier, so they are best recommended when exploring historical data or working with data lake-only tables.
