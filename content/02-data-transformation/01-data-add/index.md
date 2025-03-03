## Adding Data with `data` and `append` 

### Import Notebook into Dynatrace

[Notebook](https://github.com/popecruzdt/dt-k8s-otel-o11y-logs/blob/code-spaces/dt-k8s-otel-o11y-logs_dt_notebook.json)

### Including Raw Data with `data`
The `data` command provides an easy way to add any data you care about into a Notebook, Dashboard, Workflow, or any other component that uses DQL. This is most useful when pulling in data from other sources (i.e., Excel, CSV, etc.) or consolidating data from another query into a DQL statement. 

Syntax:
```
data record(key=value1),
record(key=value2,key2=value3), ...
```

For example, let's say your customer has a product name on all their key applications, placed as tag metadata on those entities in Dynatrace. This value, `DT_RELEASE_PRODUCT`, does not change often; the customer only adds new apps every 6 months. Therefore, if we use a query to return those `DT_RELEASE_PRODUCT` values, we can then build them as a new `data` array for subsequent queries without having to have to run a DQL `fetch`.

Extract the tags as a string for all of the process group instances (PGIs) in the environment, and parse out the `DT_RELEASE_PRODUCT` values.
```
fetch dt.entity.process_group_instance
| fieldsAdd tags
| fieldsAdd tagsToString = toString(tags)
| parse tagsToString, """DATA 'DT_RELEASE_PRODUCT:' LD:productName '\",' DATA EOF"""
| filter isNotNull(productName)
| summarize count=count(), by:{productName}
```

**⏩ Try it out**: Now, see if you can take the column `productName` and transform it into a new `data` array with the key `productName` and the value of these product names. Hint: you will need to use `concat` and export the data as a CSV. 

```
fetch dt.entity.process_group_instance
| fieldsAdd tags
| fieldsAdd tagsToString = toString(tags)
| parse tagsToString, """DATA 'DT_RELEASE_PRODUCT:' LD:productName '\",' DATA EOF"""
| filter isNotNull(productName)
| summarize count=count(), by:{productName}
| fieldsAdd outputRecord = concat("record(productName=\"..."))
```
![Data for Export](../../../assets/images/data_for_export.png)

The ability to add data on demand - particularly for non-DQL data sources and for quasi-static, long-running DQL queries - is a powerful way to add data into your DQL statements.

### Raw JSON Data Types with `data`
The `data` [command](https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-query-language/commands/data-source-commands#data) allows you to also pull in raw JSON content. 

The following is an example for nested JSON:
```
data json:"""[
  {
    "dataRecord": {
      "name": "jwoo1",
      "depositAmount": 112,
      "bankAccountId": 1234,
      "properties":
      {
        "type": "savings",
        "interestRate": 0.0045,
        "relatedAccounts":
        {
          "savings": "N/A",
          "checking": "2341",
          "mmf": "1493"
        }
      }
    }
  },
  {
    "dataRecord": {
      "name": "jdoe2",
      "depositAmount": 343,
      "bankAccountId": 1120,
      "properties":
      {
        "type": "checking",
        "interestRate": 0.00325,
        "relatedAccounts":
        {
          "savings": "3001",
          "checking": "N/A",
          "mmf": "8843"
        }
      }
    }
  },
  {
    "dataRecord": {
      "name": "jdoe3",
      "depositAmount": 8433,
      "bankAccountId": 1555
    }
  },
  {
    "dataRecord": {
      "name": "batman4",
      "depositAmount": 8433413,
      "bankAccountId": 1000,
      "properties":
      {
        "type": "savings",
        "interestRate": 0.0055,
        "relatedAccounts":
        {
          "savings": "N/A",
          "checking": "3499",
          "mmf": "2224"
        }
      }
    }
  }
]"""
```

**⏩ Try it out**: For that example, copy the `data` record into your Notebook and determine the median deposit amount (`depositAmount`) from the data. 

Nested JSON like the example in the `data` record can be more easily parsed if flatted accordingly. This can be accomplished by using `fieldsFlatten`: [Documentation](https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-query-language/commands/structuring-commands#fieldsFlatten). For JSON fields, you can provide a `depth` argument that then automatically extracts nested JSON fields and places those keys as their own variables in the record.

For nested, lengthy JSON records, the `fieldsKeep` command proves useful: [Documentation](https://docs.dynatrace.com/docs/discover-dynatrace/references/dynatrace-query-language/commands/selection-and-modification-commands#fieldsKeep). The `fieldsKeep` command retains only the records that match either an exact key or a key with wildcards. An example wildcard pattern for the above would be: `fieldsKeep "dataRecord.name*"`

In the case of that `fieldsKeep` pattern, only the `name` nested JSON keys and their respective values would be kept. Wildcards can be applied either as prefixes or suffixes (i.e., before and after the matching pattern).

**⏩ Try it out**: Using `fieldsFlatten`, `depth`, and `fieldsKeep`, count how many `relatedAccounts.savings` keys are defined.

### Adding Data Sets With `append`
The `append` command ([Documentation]()) is one of three ways to add data to an existing DQL data set. This command behaves similarly to a **SQL UNION ALL** operation between two data sets `A` (i.e., "left") and `B` (i.e., "right"). A diagram showing that is below, highlighting the fact that the two sets have no excluded intersection and that the two sets may have duplicate values. However, the keys will still remain unique.

![append Set Logic](../../../assets/images/append_set_diagram.png)

Example syntax for a DQL `fetch`:
```
fetch logs
| fieldsAdd keyValue="test"
| limit 10
| append
[
   data record(keyValue="test2", content="Hello World")
]
```
In that example case, log records will have an additional key added to the records, `keyValue`, and its value will be fixed as `test`. With the `append` command, the `keyValue` key will have the value `test2` added as a record. Further, the `content` key from the previous log DQL statement will have the value `Hello World`. Therefore, the total record count from the query will be 11, 10 from the logs and 1 from the `append`, with the keys `keyValue` and `content` having non-null entries on all records. 

A potentially useful scenario for `append` is when you have multiple metric queries that you want to "glue" together, regardless of whether you have a common value between `A` ("left") and `B` ("right"). For instance, if you wanted to show the timeseries for host idle CPU over the last 2 hours and in the same window 7 days ago, you could accomplish that with `append`, as seen below. 

Another useful scenario is to combine metrics and logs on the same entity, namely, a host or a PGI.

Example:
```
timeseries cpuIdle=avg(dt.host.cpu.idle), by: { host.name }, filter: { matchesValue(host.name, "ip-172-31-23-111.ec2.internal") }
| fieldsAdd cpuIdleAvg = arrayAvg(cpuIdle)
| append
[
  timeseries cpuIdle7dShift = avg(dt.host.cpu.idle), by: { host.name }, filter: { matchesValue(host.name, "ip-172-31-23-111.ec2.internal") }, shift: -168h
  | fieldsAdd cpuIdleAvg7dShift = arrayAvg(cpuIdle7dShift)
]
```
![append for host idle CPU](../../../assets/images/host_idle_cpu_append.png)

**⏩ Try it out**: Let's say you're investigating a JVM memory leak on host `i-0d8d16e6f7c82fd48`. To do so, we would like to get the heap details for that host combined with the server's logs. Using `append`, stitch together the heap metric (hint: `dt.runtime.jvm.memory_pool.used`) with the `WARN` and `ERROR` logs for that host. Make a timeseries for the log count (hint: `| makeTimeseries count=count()`), and plot both timeseries on the same time selection (e.g., **Last 2 hours**).  
