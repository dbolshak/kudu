### Awesome KUDU API
This my first public post here, but I feel that it could be interested for wide audience in data engineering field.
Recently we’ve started using [apache kudu](https://kudu.apache.org) as a storage engine. And I want to express my feelings regarding programming API.
In few words, Kudu has amazing and user-friendly API. There are bindings for different languages:
- C++
- Java
- Python.

There is also a scala wrapper on java client for spark integration purposes.
I will describe my experience programming with Kudu based on Java client and spark extensions.
 
But before diving into nuts and bolts a couple of notes about Kudu from official documentation:
> Apache Kudu is a columnar storage manager developed for the Hadoop platform. Kudu shares the common technical properties of Hadoop ecosystem applications: It runs on commodity hardware, is horizontally scalable, and supports highly available operation.

The full doc is available [here](https://www.cloudera.com/documentation/kudu/latest/PDF/cloudera-kudu.pdf)
I will try to leave out overviewing of operation side of Kudu as much as possible and instead of that concentrate on API.

Few examples provided on [official site](https://kudu.apache.org/docs/developing.html#_working_examples).
And yes, there are just really few examples. Honestly it’s enough to get started. But please remember, Kudu is an open source project, you can find its repository on [GitHub](https://github.com/apache/kudu). Thanks to Kudu's developers for their unit tests, they really really like tests, and several times I used unit tests as another source of examples. Fully agree with Uncle Bob that tests are the best documentation.

In next few sections I will provide a simplified my source code snippets (sometimes they are partially covered in official documentation).

The following snippet shows the simplest way how to ingest data using spark to Kudu (it should be adopted for production usage for your use-case)
```
object ParquetToKuduJob {
  def main(args: Array[String]): Unit = {

    val spark = SparkSession.builder().getOrCreate()
    try {
      val inputPath: String = ???
      val kuduMasters: String = ???
      val keyFields: scala.collection.immutable.List = ???
      val partitionColumns: java.util.List = ???
      val numberOfPartitions: Int = ???

      val df = spark.read.parquet(inputPath)

      val kuduContext = new KuduContext(kuduMasters, spark.sparkContext)
      val kuduTable = args("kuduTable")

      kuduContext.createTable(kuduTable, df.schema, keyFields,
        new CreateTableOptions()
          .setNumReplicas(3)
          .addHashPartitions(partitionColumns, numberOfPartitions))

      kuduContext.insertIgnoreRows(df, kuduTable)
    } finally {
      spark.stop()
    }
  }
}
```
Just to note:
`partitionColumns` should be subset of `keyFields`.
all attributes provided in `keyFields` must be not empty.

In example above only hash partitioning used, but Kudu also provides range partition. I did not include it in the first snippet for two reasons:
* Kudu does not allow to create a lot of partitions at creating time. Maximum value is defined like `max_create_tablets_per_ts x number of live tservers`. By default `max_create_tablets_per_ts` is 20, but of course you could override it in master configuration.
* Range partition starts to shine while you ingest your new data, it’s just increaditable feature for time series applications, or in cases where you need to keep data with fixed window of history (say for last 12 months).

So looks like it’s a time to show example with range partition. Unfortunately there is no ready to go solution just with KuduContext, but it’s not a big deal to use KuduClient. The simplest example looks like
```
val spark = SparkSession.builder().getOrCreate()

val kuduMasters: String = ???
val kuduContext = new KuduContext(kuduMasters, spark.sparkContext)

val kuduTableName: String = ???

val schemaAsString: String = ???
val schema: org.apache.kudu.Schema = parseInputSchema(schemaAsString)
val rangeColumn: String = ???

def bound(i: Int) = {
  val bound = schema.newPartialRow()
  bound.addInt(rangeColumn, i)
  bound
}

val rangeColumns: java.util.List = List(rangeColumn).asJava
val partitionColumns: java.util.List = ???
val numberOfPartitions: Int = ???

client.createTable(kuduTableName, schema, new CreateTableOptions()
  .setNumReplicas(3)
  .setRangePartitionColumns(rangeColumns)
  .addRangePartition(bound(Int.MinValue), bound(0))
  .addHashPartitions(partitionColumns, numberOfPartitions))

val lowerBoundRange = 0.to(10000, step = 100)
//for production use cases you don't need to use Int.MaxValue
val upperBoundRange = lowerBoundRange.tail :+ Int.MaxValue

//usually you call something similar later, when you need to create new range partition
client.alterTable(kuduTableName, (lowerBoundRange zip upperBoundRange).foldLeft(new AlterTableOptions) { case (opt, (lower, upper)) =>
  opt.addRangePartition(bound(lower), bound(upper))
})
```
It’s not full, but gives enough clue how it could be applied.
Very important notes here are:
- ranges can not be overlapped
- you must call setRangePartitionColumns on client while creating table if you partition by ranges using not full key
- you must call addRangePartition on client while creating table with initial range, otherwise Kudu will think that you use unbounded range (and because of ranges could not be overlapped you won’t be able to add more ranges later, because any new range will be overlapped with default unbounded range)
- carefully planing your partition schema could enable so called `partitioning pruning` option. 

In example above there is no something like `keyFields` (as in example before), this information now is provided in `val schema: org.apache.kudu.Schema`.

Final example will cover how to use client to perform narrow scans
```
client
  .newScannerBuilder(kuduTable)
  .addPredicate(KuduPredicate.newComparisonPredicate(
    columnSchema,
    ComparisonOp.EQUAL,
    valueWhichWeAreLookingFor
  ))
  .setProjectedColumnNames(projectedColumns)
  .build()
  .nextRows()
```
Just on comment here, as it's already said, Kudu is columnar storage, providing projected columns (using `setProjectedColumnNames`) can improve your reading throughput.
Really user friendly API.

Also we performed a lot of performance and operation tests, compression and encodings research.
But all of these is another subject and if the article is useful I will follow with my findings.
Kudu must go on!
