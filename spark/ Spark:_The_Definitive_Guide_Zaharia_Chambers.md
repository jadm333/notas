# Spark: The Definitive Guide: Big Data Processing Made Simple
**Matei Zaharia, Bill Chambers**



## Intro
Launching python REPL.
```bash
./bin/pyspark
```
Launching scala REPL.
```bash
./bin/spark-shell
```
Launching Saprk SQL REPL.
```bash
./bin/spark-sql
```

```scala
// Se crea un data frame distribuido
val myRange = spark.range(1000).toDF("number")

```
**Partitions** are collection of rows & this are distributed in many executors
Types of transformation of **partitions**:
1. *narrow dependencies* : each input partition will contribute to only one output partition. *ONE TO ONE*
2. *wide dependency* : input partitions contributing to many output partitions ONE TO MANY

A una Data frame:
**TRANSFORMATION**
```scala
// Transformacion (mutation)
val divisBy2 = myRange.where("number % 2 = 0")
```

LAZY EVALUATION UNTIL AN ACTION OCCUR
**ACTION**
Hay 3 diferentes:
1. Para ver datos en la consola
2. Collect data to native objects in the respective language
3. To write to output to data sources


```scala
// Transformacion (mutation)
divisBy2.count()
```

Example
Schema: estructura y tipo de datos
Auto schema y specify schema( esta mejor en ambientes de produccion)

```scala
// TRANSFORMATION
val flightData2015 = spark
  .read
  .option("inferSchema", "true")
  .option("header", "true")
  .csv("/data/flight-data/csv/2015-summary.csv")

// ACTION
flightData2015.take(3)
```

Set number of partitions:
```scala
spark.conf.set("spark.sql.shuffle.partitions", "5")

```
En mi logica el numero de particiones tiene que ser el numero de ejecutores un ejecutor es un procesador

#### Data frames and sql
Primero creas un tabla/view temporal y luego la llamas y transformas
```scala
flightData2015.createOrReplaceTempView("flight_data_2015")

val sqlWay = spark.sql("""
SELECT DEST_COUNTRY_NAME, count(1)
FROM flight_data_2015
GROUP BY DEST_COUNTRY_NAME
""")

val dataFrameWay = flightData2015
  .groupBy('DEST_COUNTRY_NAME)
  .count()

sqlWay.explain
dataFrameWay.explain
```

## Sparks toolsets

### Spark submit

Para correr un script de spark
```bash
./bin/spark-submit 
  --class org.apache.spark.examples.SparkPi
  --master local 
  ./examples/jars/spark-examples_2.11-2.2.0.jar 10
```

## Api Overview
### Schemas
```scala
import org.apache.spark.sql.types.{StructField, StructType, StringType, LongType}
import org.apache.spark.sql.types.Metadata

val myManualSchema = StructType(Array(
  StructField("DEST_COUNTRY_NAME", StringType, true),
  StructField("ORIGIN_COUNTRY_NAME", StringType, true),
  StructField("count", LongType, false,
    Metadata.fromJson("{\"hello\":\"world\"}"))
))

val df = spark.read.format("json").schema(myManualSchema)
  .load("/data/flight-data/json/2015-summary.json")
```

### Columns

```scala
import org.apache.spark.sql.functions.{col, column}
col("someColumnName")
column("someColumnName")
```

#### Explicit column references
```scala

df.col("count")
// Need to import spark.implicits._ to use $
$"myColumn"
'myColumn
```

### Expressions
```scala
import org.apache.spark.sql.functions.expr
expr("(((someCol + 5) * 200) - 6) < otherCol")
```
### Rows
```scala
import org.apache.spark.sql.Row
val myRow = Row("Hello", null, 1, false)


myRow(0) // type Any
myRow(0).asInstanceOf[String] // String
myRow.getString(0) // String
myRow.getInt(2) // Int
```

### DataFrame Transformations

- We can add rows or columns
- We can remove rows or columns
- We can transform a row into a column (or vice versa)
- We can change the order of rows based on the values in columns

#### Creating DataFrames

```scala
val df = spark.read.format("json")
  .load("/data/flight-data/json/2015-summary.json")
df.createOrReplaceTempView("dfTable")
import org.apache.spark.sql.Row
import org.apache.spark.sql.types.{StructField, StructType, StringType, LongType}

val myManualSchema = new StructType(Array(
  new StructField("some", StringType, true),
  new StructField("col", StringType, true),
  new StructField("names", LongType, false)))
val myRows = Seq(Row("Hello", null, 1L))
val myRDD = spark.sparkContext.parallelize(myRows)
val myDf = spark.createDataFrame(myRDD, myManualSchema)
myDf.show()
```
#### select and selectExpr

```scala
df.select("DEST_COUNTRY_NAME").show(2)
df.select("DEST_COUNTRY_NAME", "ORIGIN_COUNTRY_NAME").show(2)


import org.apache.spark.sql.functions.{expr, col, column}
df.select(
    df.col("DEST_COUNTRY_NAME"),
    col("DEST_COUNTRY_NAME"),
    column("DEST_COUNTRY_NAME"),
    'DEST_COUNTRY_NAME,
    $"DEST_COUNTRY_NAME",
    expr("DEST_COUNTRY_NAME"))
  .show(2)
```

|DEST_COUNTRY_NAME|DEST_COUNTRY_NAME|DEST_COUNTRY_NAME|DEST_COUNTRY_NAME|DEST_COUNTRY_NAME|DEST_COUNTRY_NAME|
|-----------------|-----------------|-----------------|-----------------|-----------------|-----------------|
|    United States|    United States|    United States|    United States|    United States|    United States|
|    United States|    United States|    United States|    United States|    United States|    United States|

Error juntar cols y strings
```
df.select(col("DEST_COUNTRY_NAME"), "DEST_COUNTRY_NAME")
```

Expresions are mini sql- statements
```scala
df.select(expr("DEST_COUNTRY_NAME AS destination")).show(2)

df.select(expr("DEST_COUNTRY_NAME as destination").alias("DEST_COUNTRY_NAME"))
  .show(2)
  
df.selectExpr("DEST_COUNTRY_NAME as newColumnName", "DEST_COUNTRY_NAME").show(2)

```

Ahorrar el `select(expr())` con `selectExpr()`

Agregar columnas
```scala
df.selectExpr(
    "*", // include all original columns
    "(DEST_COUNTRY_NAME = ORIGIN_COUNTRY_NAME) as withinCountry")
  .show(2)
```

#### Literals
Son lo mismo:
```scala
import org.apache.spark.sql.functions.lit
df.select(expr("*"), lit(1).as("One")).show(2)

val sqlWay = spark.sql("""
SELECT *, 1 as One FROM flight_data_2015 LIMIT 2
""").show()
```

#### Agregar columna

```scala
import org.apache.spark.sql.functions.lit
df.select(expr("*"), lit(1).as("One")).show(2)

val sqlWay = spark.sql("""
SELECT *, 1 as One FROM flight_data_2015 LIMIT 2
""").show()

df.withColumn("numberOne", lit(1)).show(2)

df.withColumn("withinCountry", expr("ORIGIN_COUNTRY_NAME == DEST_COUNTRY_NAME"))
  .show(2)
```

#### Renombrar columna

```scala
df.withColumnRenamed("DEST_COUNTRY_NAME", "dest").columns
```

#### Reserved Characters and Keywords
```scala
import org.apache.spark.sql.functions.expr

val dfWithLongColName = df.withColumn(
  "This Long Column-Name",
  expr("ORIGIN_COUNTRY_NAME"))
```

#### Case Sensitive

```sql
set spark.sql.caseSensitive true
```
#### Quitar columnas

```scala
dfWithLongColName.drop("ORIGIN_COUNTRY_NAME", "DEST_COUNTRY_NAME")
```

#### Changing a Column’s Type (cast)

```scala
df.withColumn("count2", col("count").cast("long"))
```
#### Filtrar columnas

```scala
df.filter(col("count") < 2).show(2)
df.where("count < 2").show(2)
```

Multiple filters spark interprets it as AND

```scala
df.where(col("count") < 2).where(col("ORIGIN_COUNTRY_NAME") =!= "Croatia")
  .show(2)
```
Es equivalente a:

```sql
SELECT * FROM dfTable 
WHERE count < 2 AND ORIGIN_COUNTRY_NAME != "Croatia" 
LIMIT 2
```

#### Unique rows

```scala
df.select("ORIGIN_COUNTRY_NAME", "DEST_COUNTRY_NAME").distinct().count()
```

```sql
SELECT COUNT(DISTINCT(ORIGIN_COUNTRY_NAME, DEST_COUNTRY_NAME)) FROM dfTable
```
#### Random Samples

```scala
val seed = 5
val withReplacement = false
val fraction = 0.5
df.sample(withReplacement, fraction, seed).count()
```

#### Random Splits

```scala
val seed = 96845
val dataFrames = df.randomSplit(Array(0.25, 0.75), seed)
dataFrames(0).count() > dataFrames(1).count() // False
```

#### Concatenating and Appending Rows (Union)

```scala
import org.apache.spark.sql.Row
val schema = df.schema
val newRows = Seq(
  Row("New Country", "Other Country", 5L),
  Row("New Country 2", "Other Country 3", 1L)
)
val parallelizedRows = spark.sparkContext.parallelize(newRows)
val newDF = spark.createDataFrame(parallelizedRows, schema)
df.union(newDF)
  .where("count = 1")

```

#### Sorting Rows

`sort()` = `orderBy()`

```scala
df.sort("count").show(5)
df.orderBy("count", "DEST_COUNTRY_NAME").show(5)
df.orderBy(col("count"), col("DEST_COUNTRY_NAME")).show(5)

import org.apache.spark.sql.functions.{desc, asc}
df.orderBy(expr("count desc")).show(2)
df.orderBy(desc("count"), asc("DEST_COUNTRY_NAME")).show(2)
```

```sql
SELECT * FROM dfTable ORDER BY count DESC, DEST_COUNTRY_NAME ASC LIMIT 2
```

#### limit

```scala
df.limit(5).show()
```

```sql
SELECT * FROM dfTable LIMIT 6
```

#### `coalesce` and `repartition`

```scala
df.rdd.getNumPartitions // 1
df.repartition(5)
```

If you know that you’re going to be filtering by a certain column often, it can be worth repartitioning based on that column:

```scala
df.repartition(col("DEST_COUNTRY_NAME"))
df.repartition(5, col("DEST_COUNTRY_NAME"))
```

#### ¿`coalesce`?


#### Collecting Rows to the Driver

```scala
val collectDF = df.limit(10)
collectDF.take(5) // take works with an Integer count
collectDF.show() // this prints it out nicely
collectDF.show(5, false)
collectDF.collect()

// to iterator
collectDF.toLocalIterator()
```

## Working with Different Types of Data

### API docs links

#### DataFrame (Dataset) Methods
[Spark 2.3.0 ScalaDoc](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset)
[Spark 2.3.0 ScalaDoc DataFrameStatFunctions](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.DataFrameStatFunctions)
[Spark 2.3.0 ScalaDoc DataFrameNaFunctions](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.DataFrameNaFunctions)

#### Column Methods

[Spark 2.3.0 ScalaDoc](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Column)

#### SQL functions `org.apache.spark.sql.functions`
[Spark 2.3.0 ScalaDoc](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.functions$)

```scala
val df = spark.read.format("csv")
  .option("header", "true")
  .option("inferSchema", "true")
  .load("/data/retail-data/by-day/2010-12-01.csv")
df.printSchema()
df.createOrReplaceTempView("dfTable")
```

### Lit

Se crea una columna literal (i.e. el mismo valor para todos los registros)

```scala
import org.apache.spark.sql.functions.lit
df.select(lit(5), lit("five"), lit(5.0))
```
```sql
SELECT 5, "five", 5.0
```

### Booleans

```scala
import org.apache.spark.sql.functions.col
df.where(col("InvoiceNo").equalTo(536365))
  .select("InvoiceNo", "Description")
  .show(5, false)
  
df.where(col("InvoiceNo") === 536365)
  .select("InvoiceNo", "Description")
  .show(5, false)
  
df.where("InvoiceNo = 536365")
  .show(5, false)
  
df.where("InvoiceNo <> 536365")
  .show(5, false)
```
OR's

```scala
al priceFilter = col("UnitPrice") > 600
val descripFilter = col("Description").contains("POSTAGE")
df.where(col("StockCode").isin("DOT")).where(priceFilter.or(descripFilter))
  .show()
```

```sql
SELECT * 
FROM dfTable 
WHERE StockCode in ("DOT") AND(UnitPrice > 600 OR
    instr(Description, "POSTAGE") >= 1)
```

Boolean Column
```scala
val DOTCodeFilter = col("StockCode") === "DOT"
val priceFilter = col("UnitPrice") > 600
val descripFilter = col("Description").contains("POSTAGE")
df.withColumn("isExpensive", DOTCodeFilter.and(priceFilter.or(descripFilter)))
  .where("isExpensive")
  .select("unitPrice", "isExpensive").show(5)
```

```sql
SELECT UnitPrice, (StockCode = 'DOT' AND
  (UnitPrice > 600 OR instr(Description, "POSTAGE") >= 1)) as isExpensive
FROM dfTable
WHERE (StockCode = 'DOT' AND
       (UnitPrice > 600 OR instr(Description, "POSTAGE") >= 1))
```

### Numbers columns

```scala
import org.apache.spark.sql.functions.{expr, pow}
val fabricatedQuantity = pow(col("Quantity") * col("UnitPrice"), 2) + 5
df.select(expr("CustomerId"), fabricatedQuantity.alias("realQuantity")).show(2)

df.selectExpr(
  "CustomerId",
  "(POWER((Quantity * UnitPrice), 2.0) + 5) as realQuantity").show(2)
```

```sql
SELECT customerId, (POWER((Quantity * UnitPrice), 2.0) + 5) as realQuantity
FROM dfTable
```

Rounding

```scala
import org.apache.spark.sql.functions.{round, bround}
df.select(round(col("UnitPrice"), 1).alias("rounded"), col("UnitPrice")).show(5)

import org.apache.spark.sql.functions.lit
df.select(round(lit("2.5")), bround(lit("2.5"))).show(2)
```
Pearson correlation coefficient for two columns

```scala
import org.apache.spark.sql.functions.{corr}
df.stat.corr("Quantity", "UnitPrice")
df.select(corr("Quantity", "UnitPrice")).show()
```
```sql
SELECT corr(Quantity, UnitPrice) FROM dfTable
```

Summary STATS
```scala
df.describe().show()
//With Agg

import org.apache.spark.sql.functions.{count, mean, stddev_pop, min, max}
```

StatFunctions Package df.stat

```scala
val colName = "UnitPrice"
val quantileProbs = Array(0.5)
val relError = 0.05
df.stat.approxQuantile("UnitPrice", quantileProbs, relError) // 2.51

// cross-tabulation
df.stat.crosstab("StockCode", "Quantity").show() //Too long caution

//frequent item
df.stat.freqItems(Seq("StockCode", "Quantity")).show()

```

Add increasing ID

```scala
import org.apache.spark.sql.functions.monotonically_increasing_id
df.select(monotonically_increasing_id()).show(2)
```

### Strings columns

Capitalize every word in string
```scala
import org.apache.spark.sql.functions.{initcap}
df.select(initcap(col("Description"))).show(2, false)
```
```sql
SELECT initcap(Description) FROM dfTable
```
Similar with upper and lower

```scala
import org.apache.spark.sql.functions.{lower, upper}
df.select(col("Description"),
  lower(col("Description")),
  upper(lower(col("Description")))).show(2)
```
```sql
SELECT Description, lower(Description), Upper(lower(Description)) FROM dfTable
```

Triming spaces

```scala
import org.apache.spark.sql.functions.{lit, ltrim, rtrim, rpad, lpad, trim}
df.select(
    ltrim(lit("    HELLO    ")).as("ltrim"),
    rtrim(lit("    HELLO    ")).as("rtrim"),
    trim(lit("    HELLO    ")).as("trim"),
    lpad(lit("HELLO"), 3, " ").as("lp"),
    rpad(lit("HELLO"), 10, " ").as("rp")).show(2)
```
```sql
SELECT
  ltrim('    HELLLOOOO  '),
  rtrim('    HELLLOOOO  '),
  trim('    HELLLOOOO  '),
  lpad('HELLOOOO  ', 3, ' '),
  rpad('HELLOOOO  ', 10, ' ')
FROM dfTable
```
#### Regular Expressions regex

```scala
import org.apache.spark.sql.functions.regexp_replace
val simpleColors = Seq("black", "white", "red", "green", "blue")
val regexString = simpleColors.map(_.toUpperCase).mkString("|")
// the | signifies `OR` in regular expression syntax
df.select(
  regexp_replace(col("Description"), regexString, "COLOR").alias("color_clean"),
  col("Description")).show(2)

```
```sql
SELECT
  regexp_replace(Description, 'BLACK|WHITE|RED|GREEN|BLUE', 'COLOR') as
  color_clean, Description
FROM dfTable
```

Replace charecters
```scala
import org.apache.spark.sql.functions.translate
df.select(translate(col("Description"), "LEET", "1337"), col("Description"))
  .show(2)

```
```sql
SELECT translate(Description, 'LEET', '1337'), Description FROM dfTable
```

Substring

```scala
import org.apache.spark.sql.functions.regexp_extract
val regexString = simpleColors.map(_.toUpperCase).mkString("(", "|", ")")
// the | signifies OR in regular expression syntax
df.select(
     regexp_extract(col("Description"), regexString, 1).alias("color_clean"),
     col("Description")).show(2)
```

```sql
SELECT regexp_extract(Description, '(BLACK|WHITE|RED|GREEN|BLUE)', 1),
  Description
FROM dfTable
```

Check if string match

```scala
val containsBlack = col("Description").contains("BLACK")
val containsWhite = col("DESCRIPTION").contains("WHITE")
df.withColumn("hasSimpleColor", containsBlack.or(containsWhite))
  .where("hasSimpleColor")
  .select("Description").show(3, false)
```

```sql
SELECT Description FROM dfTable
WHERE instr(Description, 'BLACK') >= 1 OR instr(Description, 'WHITE') >= 1
```

programmatically generate columns

```scala
val simpleColors = Seq("black", "white", "red", "green", "blue")
val selectedColumns = simpleColors.map(color => {
   col("Description").contains(color.toUpperCase).alias(s"is_$color")
}):+expr("*") // could also append this value
df.select(selectedColumns:_*).where(col("is_white").or(col("is_red")))
  .select("Description").show(3, false)
```

### Dates and Timestamps

```scala
import org.apache.spark.sql.functions.{current_date, current_timestamp}
val dateDF = spark.range(10)
  .withColumn("today", current_date())
  .withColumn("now", current_timestamp())
dateDF.createOrReplaceTempView("dateTable")
```

Add and substract days

```scala
import org.apache.spark.sql.functions.{date_add, date_sub}
dateDF.select(date_sub(col("today"), 5), date_add(col("today"), 5)).show(1)
```
```sql
SELECT date_sub(today, 5), date_add(today, 5) FROM dateTable
```

Diff days and months

```scala
import org.apache.spark.sql.functions.{datediff, months_between, to_date}
dateDF.withColumn("week_ago", date_sub(col("today"), 7))
  .select(datediff(col("week_ago"), col("today"))).show(1)
dateDF.select(
    to_date(lit("2016-01-01")).alias("start"),
    to_date(lit("2017-05-22")).alias("end"))
  .select(months_between(col("start"), col("end"))).show(1)
```

```sql
SELECT to_date('2016-01-01'), months_between('2016-01-01', '2017-01-01'),
datediff('2016-01-01', '2017-01-01')
FROM dateTable
```

Convert string to date if can't parse it return null

```scala
import org.apache.spark.sql.functions.to_date
val dateFormat = "yyyy-dd-MM"
val cleanDateDF = spark.range(1).select(
    to_date(lit("2017-12-11"), dateFormat).alias("date"),
    to_date(lit("2017-20-12"), dateFormat).alias("date2"))
cleanDateDF.createOrReplaceTempView("dateTable2")
```

```sql
SELECT to_date(date, 'yyyy-dd-MM'), to_date(date2, 'yyyy-dd-MM'), to_date(date)
FROM dateTable2
```

Convert to timestamp

```scala
import org.apache.spark.sql.functions.to_timestamp
cleanDateDF
  .select(to_timestamp(col("date"), dateFormat))
  .show()
```
```sql
SELECT to_timestamp(date, 'yyyy-dd-MM'), to_timestamp(date2, 'yyyy-dd-MM')
FROM dateTable2
```

Casting in SQL

```sql
SELECT cast(to_date("2017-01-01", "yyyy-dd-MM") as timestamp)
```

Filtering dates

```scala
cleanDateDF.filter(col("date2") > lit("2017-12-12")).show()
cleanDateDF.filter(col("date2") > "'2017-12-12'").show()
```


> **WARNING**
> Implicit type casting is an easy way to shoot yourself in the foot, especially when dealing with null values or dates in different timezones or formats. We recommend that you parse them explicitly instead of relying on implicit conversions.


### NULLS

**`coalesce`**: select the first non-null value from a set of columns

```scala
import org.apache.spark.sql.functions.coalesce
df.select(coalesce(col("Description"), col("CustomerId"))).show()
```
**`ifnull`**: allows you to select the second value if the first is null, and defaults to the first.

**`nullIf`**: returns null if the two values are equal or else returns the second if they are not

**`nvl`**: returns the second value if the first is null, but defaults to the first

**`nvl2`**: returns the second value if the first is not null; otherwise, it will return the last specified value

```sql
SELECT
  ifnull(null, 'return_value'),
  nullif('value', 'value'),
  nvl(null, 'return_value'),
  nvl2('not_null', 'return_value', "else_value")
FROM dfTable LIMIT 1
```
> We can use these in select expressions on DataFrames, as well.

**`drop`**: Removes rows that contain nulls. The default is to drop any row in which any value is null

```scala
df.na.drop()
df.na.drop("any") //same as above

df.na.drop("all") //drops the row only if all values are null or NaN for that row

df.na.drop("all", Seq("StockCode", "InvoiceNo")) // apply  to certain sets of columns by passing in an array of columns
```

```sql
SELECT * FROM dfTable WHERE Description IS NOT NULL
```


**`fill`**: Fill one or more columns with a set of values. This can be done by specifying a map — that is a particular value and a set of columns

```scala
df.na.fill("All Null values become this string")
df.na.fill(5:Integer) //Fill only columns of integer type
df.na.fill(5, Seq("StockCode", "InvoiceNo")) // specify columns

val fillColValues = Map("StockCode" -> 5, "Description" -> "No Value")
df.na.fill(fillColValues)
```

**`replace`**: Replece match

```scala
df.na.replace("Description", Map("" -> "UNKNOWN"))
```

##### Ordering NULLS

`asc_nulls_first`, `desc_nulls_first`, `asc_nulls_last`, or `desc_nulls_last`

### Complex Types

There are three kinds of complex types: structs, arrays, and maps

#### Structs
 Structs are Datafrema within DataFrames
 
 ```scala
df.selectExpr("(Description, InvoiceNo) as complex", "*")
df.selectExpr("struct(Description, InvoiceNo) as complex", "*")

import org.apache.spark.sql.functions.struct
val complexDF = df.select(struct("Description", "InvoiceNo").alias("complex"))
complexDF.createOrReplaceTempView("complexDF")
```
Query them

 ```scala
complexDF.select("complex.Description")
complexDF.select(col("complex").getField("Description"))
```

Alternative brings up all the columns to the top-level:
 ```scala
complexDF.select("complex.*")
```

 ```sql
SELECT complex.* FROM complexDF
```
#### Arrays

**`split`**: Split string into Array

 ```scala
import org.apache.spark.sql.functions.split
df.select(split(col("Description"), " ")).show(2)
```

 ```sql
SELECT split(Description, ' ') FROM dfTable
```

|split(Description,  )|
|---------------------|
| [WHITE, HANGING, ...] |
| [WHITE, METAL, LA...] |

Query Values

 ```scala
df.select(split(col("Description"), " ").alias("array_col"))
  .selectExpr("array_col[0]").show(2)
```
 ```sql
SELECT split(Description, ' ')[0] FROM dfTable
```

|array_col[0]|
|------------|
|       WHITE|
|       WHITE|

**`size`**: Array Length

 ```scala
import org.apache.spark.sql.functions.size
df.select(size(split(col("Description"), " "))).show(2) // shows 5 and 3
```

**`array_contains`**:

 ```scala
import org.apache.spark.sql.functions.array_contains
df.select(array_contains(split(col("Description"), " "), "WHITE")).show(2)
```

 ```sql
SELECT array_contains(split(Description, ' '), 'WHITE') FROM dfTable
```

|array_contains(split(Description,  ), WHITE)|
|--------------------------------------------|
|                                        true|
|                                        true|


**`explode`**: The explode function takes a column that consists of arrays and creates one row (with the rest of the values duplicated) per value in the array.

 ```scala
import org.apache.spark.sql.functions.{split, explode}

df.withColumn("splitted", split(col("Description"), " "))
  .withColumn("exploded", explode(col("splitted")))
  .select("Description", "InvoiceNo", "exploded").show(2)
```

 ```sql
SELECT Description, InvoiceNo, exploded
FROM (SELECT *, split(Description, " ") as splitted FROM dfTable)
LATERAL VIEW explode(splitted) as exploded
```

|         Description|InvoiceNo|exploded|
|--------------------|---------|--------|
|WHITE HANGING HEA...|   536365|   WHITE|
|WHITE HANGING HEA...|   536365| HANGING|


#### Maps
Maps are created by using the map function and key-value pairs of columns. You then can select them just like you might select from an array, They are the similar as scala `Map` data structure:

```scala
// in Scala
import org.apache.spark.sql.functions.map
df.select(map(col("Description"), col("InvoiceNo")).alias("complex_map")).show(2)
```

 ```sql
SELECT map(Description, InvoiceNo) as complex_map FROM dfTable
WHERE Description IS NOT NULL
```
|         complex_map|
|--------------------|
|Map(WHITE HANGING...|
|Map(WHITE METAL L...|

Query Values, A missing key returns null:

 ```scala
// in Scala
df.select(map(col("Description"), col("InvoiceNo")).alias("complex_map"))
  .selectExpr("complex_map['WHITE METAL LANTERN']").show(2)
```
|complex_map[WHITE METAL LANTERN]|
|--------------------------------|
|                            null|
|                          536365|

explode map

 ```scala
df.select(map(col("Description"), col("InvoiceNo")).alias("complex_map"))
  .selectExpr("explode(complex_map)").show(2)
```

|                 key| value|
|--------------------|------|
|WHITE HANGING HEA...|536365|
| WHITE METAL LANTERN|536365|


### JSON

 ```scala
val jsonDF = spark.range(1).selectExpr("""
  '{"myJSONKey" : {"myJSONValue" : [1, 2, 3]}}' as jsonString""")
```

**`get_json_object`**: inline query a JSON object, be it a dictionary or array. You can use json_tuple if this object has only one level of nesting

 ```scala
import org.apache.spark.sql.functions.{get_json_object, json_tuple}
jsonDF.select(
    get_json_object(col("jsonString"), "$.myJSONKey.myJSONValue[1]") as "column",
    json_tuple(col("jsonString"), "myJSONKey")).show(2)
    
//The same
jsonDF.selectExpr(
  "json_tuple(jsonString, '$.myJSONKey.myJSONValue[1]') as column").show(2)

```

|column|                  c0|
|------|--------------------|
|     2|{"myJSONValue":[1...]|


**`to_json`**:

 ```scala
import org.apache.spark.sql.functions.to_json
df.selectExpr("(InvoiceNo, Description) as myStruct")
  .select(to_json(col("myStruct")))
```
**`from_json`**:

 ```scala
import org.apache.spark.sql.functions.from_json
import org.apache.spark.sql.types._
val parseSchema = new StructType(Array(
  new StructField("InvoiceNo",StringType,true),
  new StructField("Description",StringType,true)))
df.selectExpr("(InvoiceNo, Description) as myStruct")
  .select(to_json(col("myStruct")).alias("newJSON"))
  .select(from_json(col("newJSON"), parseSchema), col("newJSON")).show(2)
```

|jsontostructs(newJSON)|             newJSON|
|----------------------|--------------------|
|  [536365,WHITE HAN...]|{"InvoiceNo":"536...|
|  [536365,WHITE MET...]|{"InvoiceNo":"536...|


### User-Defined Functions (UDFs)

Example, power3

 ```scala
val udfExampleDF = spark.range(5).toDF("num")
def power3(number:Double):Double = number * number * number
power3(2.0)
```

Next you need to pass this function to all workers

> In scala/Java there will be little performance penalty, python is faster because spark start a python process on each worker
 
>Starting this Python process is expensive, but the real cost is in serializing the data to Python. This is costly for two reasons: it is an expensive computation, but also, after the data enters Python, Spark cannot manage the memory of the worker. This means that you could potentially cause a worker to fail if it becomes resource constrained (because both the JVM and Python are competing for memory on the same machine). We recommend that you write your UDFs in Scala or Java — the small amount of time it should take you to write the function in Scala will always yield significant speed ups, and on top of that, you can still use the function from Python!

Register the function:
 ```scala
import org.apache.spark.sql.functions.udf
val power3udf = udf(power3(_:Double):Double)

udfExampleDF.select(power3udf(col("num"))).show()
```
|power3(num)|
|-----------|
|          0|
|          1|

Register to use it in spark sql, this way you can use the function defined in scala on python and vice versa

 ```scala
spark.udf.register("power3", power3(_:Double):Double)
udfExampleDF.selectExpr("power3(num)").show(2)
```
After register them

 ```sql
SELECT power3(12), power3py(12) -- doesn't work because of return type
```

## Aggregations

- A “group by” allows you to specify one or more keys as well as one or more aggregation functions to transform the value columns.
- A “window” gives you the ability to specify one or more keys as well as one or more aggregation functions to transform the value columns. However, the rows input to the function are somehow related to the current row.
- A “grouping set,” which you can use to aggregate at multiple different levels. Grouping sets are available as a primitive in SQL and via rollups and cubes in DataFrames.
- A “rollup” makes it possible for you to specify one or more keys as well as one or more aggregation functions to transform the value columns, which will be summarized hierarchically.
- A “cube” allows you to specify one or more keys as well as one or more aggregation functions to transform the value columns, which will be summarized across all combinations of columns.

Data for this chapter:
 ```scala
val df = spark.read.format("csv")
  .option("header", "true")
  .option("inferSchema", "true")
  .load("/data/retail-data/all/*.csv")
  .coalesce(5)
df.cache()
df.createOrReplaceTempView("dfTable")
```

|InvoiceNo|StockCode|         Description|Quantity|   InvoiceDate|UnitPrice|Cu...|
|---------|---------|--------------------|--------|--------------|---------|-----|
|   536365|   85123A|WHITE HANGING...    |       6|12/1/2010 8:26|     2.55|  ...|
|   536365|    71053|WHITE METAL...      |       6|12/1/2010 8:26|     3.39|  ...|
|  ...    |  ...    |  ...    |  ...    |  ...    |  ...    |  ...    |
|   536367|    21755|LOVE BUILDING BLO...|       3|12/1/2010 8:34|     5.95|  ...|
|   536367|    21777|RECIPE BOX WITH M...|       4|12/1/2010 8:34|     7.95|  ...|

Basic aggregations:
 ```scala
df.count() == 541909
```

### Aggregation Functions

Scala reference
[Spark 2.3.1 ScalaDoc](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.functions$)

**`count`**: The difference of the above function is that this is a transformation instead of an action
 ```scala
import org.apache.spark.sql.functions.count
df.select(count("StockCode")).show() // 541909
```
 ```sql
SELECT COUNT(*) FROM dfTable
```
> Spark will count null values (including rows containing all nulls). However, when counting an individual column, Spark will not count the null values.


**`countDistinct`**
 ```scala
import org.apache.spark.sql.functions.countDistinct
df.select(countDistinct("StockCode")).show() // 4070
```
 ```sql
SELECT COUNT(DISTINCT *) FROM DFTABLE
```

**`countDistinct`**: If exact number is not neccesary, useful in LARGE datasets, estimation error parameter needed.
 ```scala
import org.apache.spark.sql.functions.approx_count_distinct
df.select(approx_count_distinct("StockCode", 0.1)).show() // 3364
```
 ```sql
SELECT approx_count_distinct(StockCode, 0.1) FROM DFTABLE
```

**`first and last`**: based on the rows order in the DataFrame, not on the values
 ```scala
import org.apache.spark.sql.functions.{first, last}
df.select(first("StockCode"), last("StockCode")).show()
```

 ```sql
SELECT first(StockCode), last(StockCode) FROM dfTable
```


**`min and max`**:
 ```scala
import org.apache.spark.sql.functions.{min, max}
df.select(min("Quantity"), max("Quantity")).show()
```
 ```scala
SELECT min(Quantity), max(Quantity) FROM dfTable
```

**`sum`**:
 ```scala
import org.apache.spark.sql.functions.sum
df.select(sum("Quantity")).show() // 5176450
```
 ```sql
SELECT sum(Quantity) FROM dfTable
```

**`sumDistinct`**:
 ```scala
import org.apache.spark.sql.functions.sumDistinct
df.select(sumDistinct("Quantity")).show() // 29310
```
????
 ```sql
SELECT SUM(Quantity) FROM dfTable -- 29310 -- ???
```

**`avg`**: average
 ```scala
import org.apache.spark.sql.functions.{sum, count, avg, expr}

df.select(
    count("Quantity").alias("total_transactions"),
    sum("Quantity").alias("total_purchases"),
    avg("Quantity").alias("avg_purchases"),
    expr("mean(Quantity)").alias("mean_purchases"))
  .selectExpr(
    "total_purchases/total_transactions",
    "avg_purchases",
    "mean_purchases").show()
```

**`Variance and Standard Deviation`**: average
 ```scala
import org.apache.spark.sql.functions.{var_pop, stddev_pop}
import org.apache.spark.sql.functions.{var_samp, stddev_samp}
df.select(var_pop("Quantity"), var_samp("Quantity"),
  stddev_pop("Quantity"), stddev_samp("Quantity")).show()
```
 ```sql
SELECT var_pop(Quantity), var_samp(Quantity),
  stddev_pop(Quantity), stddev_samp(Quantity)
FROM dfTable
```

**`skewness and kurtosis`**:
 ```scala
import org.apache.spark.sql.functions.{skewness, kurtosis}
df.select(skewness("Quantity"), kurtosis("Quantity")).show()
```
 ```sql
SELECT skewness(Quantity), kurtosis(Quantity) FROM dfTable
```
**`Covariance and Correlation`**:

 ```scala
import org.apache.spark.sql.functions.{corr, covar_pop, covar_samp}
df.select(corr("InvoiceNo", "Quantity"), covar_samp("InvoiceNo", "Quantity"),
    covar_pop("InvoiceNo", "Quantity")).show()
```
 ```sql
SELECT corr(InvoiceNo, Quantity), covar_samp(InvoiceNo, Quantity),
  covar_pop(InvoiceNo, Quantity)
FROM dfTable
```
**`Aggregating to Complex Types`**: you can also perform them on complex types. For example, we can collect a list of values present in a given column or only the unique values by collecting to a set.

 ```scala
import org.apache.spark.sql.functions.{collect_set, collect_list}
df.agg(collect_set("Country"), collect_list("Country")).show()
```
### Grouping

Simple grouping:
 ```scala
df.groupBy("InvoiceNo", "CustomerId").count().show()
```
 ```sql
SELECT count(*) FROM dfTable GROUP BY InvoiceNo, CustomerId
```
#### Grouping with Expressions

 ```scala
import org.apache.spark.sql.functions.count

df.groupBy("InvoiceNo").agg(
  count("Quantity").alias("quan"),
  expr("count(Quantity)")).show() /// This ar the same
```
#### Grouping with Maps

 ```scala
df.groupBy("InvoiceNo").agg("Quantity"->"avg", "Quantity"->"stddev_pop").show()
```
 ```sql
SELECT avg(Quantity), stddev_pop(Quantity), InvoiceNo 
FROM dfTable
GROUP BY InvoiceNo
```
### Window Functions

Aggregations by either computing some aggregation on a specific “window” of data. A group-by takes data, and every row can go only into one grouping. A window function calculates a return value for every input row of a table based on a group of rows, called a frame. Each row can fall into one or more frames. A common use case is to take a look at a rolling average of some value for which each row represents one day. If you were to do this, each row would end up in seven different frames.


 ```scala
import org.apache.spark.sql.functions.{col, to_date}
val dfWithDate = df.withColumn("date", to_date(col("InvoiceDate"),
  "MM/d/yyyy H:mm"))
dfWithDate.createOrReplaceTempView("dfWithDate")

import org.apache.spark.sql.expressions.Window
import org.apache.spark.sql.functions.col
val windowSpec = Window
  .partitionBy("CustomerId", "date")
  .orderBy(col("Quantity").desc)
  .rowsBetween(Window.unboundedPreceding, Window.currentRow)
```
Agg func over window

 ```scala
import org.apache.spark.sql.functions.max
val maxPurchaseQuantity = max(col("Quantity")).over(windowSpec)
```
Ranks:

 ```scala
import org.apache.spark.sql.functions.{dense_rank, rank}
val purchaseDenseRank = dense_rank().over(windowSpec)
val purchaseRank = rank().over(windowSpec)
```
The last two return columns, to view calculated windows:

 ```scala
import org.apache.spark.sql.functions.col

dfWithDate.where("CustomerId IS NOT NULL").orderBy("CustomerId")
  .select(
    col("CustomerId"),
    col("date"),
    col("Quantity"),
    purchaseRank.alias("quantityRank"),
    purchaseDenseRank.alias("quantityDenseRank"),
    maxPurchaseQuantity.alias("maxPurchaseQuantity")).show()
```
 ```sql
SELECT CustomerId, date, Quantity,
  rank(Quantity) OVER (PARTITION BY CustomerId, date
                       ORDER BY Quantity DESC NULLS LAST
                       ROWS BETWEEN
                         UNBOUNDED PRECEDING AND
                         CURRENT ROW) as rank,

  dense_rank(Quantity) OVER (PARTITION BY CustomerId, date
                             ORDER BY Quantity DESC NULLS LAST
                             ROWS BETWEEN
                               UNBOUNDED PRECEDING AND
                               CURRENT ROW) as dRank,

  max(Quantity) OVER (PARTITION BY CustomerId, date
                      ORDER BY Quantity DESC NULLS LAST
                      ROWS BETWEEN
                        UNBOUNDED PRECEDING AND
                        CURRENT ROW) as maxPurchase
FROM dfWithDate WHERE CustomerId IS NOT NULL ORDER BY CustomerId
```
|CustomerId|      date|Quantity|quantityRank|quantityDenseRank|maxP...Quantity|
|----------|----------|--------|------------|-----------------|---------------|
|     12346|2011-01-18|   74215|           1|                1|          74215|
|     12346|2011-01-18|  -74215|           2|                2|          74215|
|     12347|2010-12-07|      36|           1|                1|             36|
|     12347|2010-12-07|      30|           2|                2|             36|
| ...|...|...|...|...|...|
|     12347|2010-12-07|      12|           4|                4|             36|
|     12347|2010-12-07|       6|          17|                5|             36|
|     12347|2010-12-07|       6|          17|                5|             36|


### Grouping Sets

Data prep
 ```scala
val dfNoNull = dfWithDate.drop()
dfNoNull.createOrReplaceTempView("dfNoNull")
```
Using SQL:

 ```sql
SELECT CustomerId, stockCode, sum(Quantity) 
FROM dfNoNull
GROUP BY customerId, stockCode
ORDER BY CustomerId DESC, stockCode DESC
```
|CustomerId|stockCode|sum(Quantity)|
|----------|---------|-------------|
|     18287|    85173|           48|
|     18287|   85040A|           48|
|     18287|   85039B|          120|
| ...| ...| ...|
|     18287|    23269|           36|

Doing the same of above but with grouping set
 ```sql
SELECT CustomerId, stockCode, sum(Quantity) FROM dfNoNull
GROUP BY customerId, stockCode GROUPING SETS((customerId, stockCode))
ORDER BY CustomerId DESC, stockCode DESC
```
> Grouping sets depend on null values for aggregation levels. If you do not filter-out null values, you will get incorrect results. This applies to cubes, rollups, and grouping sets.

regardless of customer or stock code:
 ```sql
SELECT CustomerId, stockCode, sum(Quantity) FROM dfNoNull
GROUP BY customerId, stockCode GROUPING SETS((customerId, stockCode),())
ORDER BY CustomerId DESC, stockCode DESC

```
> The GROUPING SETS operator is only available in SQL. To perform the same in DataFrames, you use the rollup and cube operators

#### Rollups

 ```scala
val rolledUpDF = dfNoNull.rollup("Date", "Country").agg(sum("Quantity"))
  .selectExpr("Date", "Country", "`sum(Quantity)` as total_quantity")
  .orderBy("Date")
rolledUpDF.show()
```
|      Date|       Country|total_quantity|
|----------|--------------|--------------|
|      null|          null|       5176450|
|2010-12-01|United Kingdom|         23949|
|2010-12-01|       Germany|           117|
|2010-12-01|        France|           449|
|...|...|...|
|2010-12-03|        France|           239|
|2010-12-03|         Italy|           164|
|2010-12-03|       Belgium|           528|

null values is where you’ll find the grand totals. A null in both rollup columns specifies the grand total across both of those columns:

 ```scala
rolledUpDF.where("Country IS NULL").show()
rolledUpDF.where("Date IS NULL").show()
```
#### Cube

Cube multi-dimensional aggregate operator is an extension of groupBy operator that allows calculating subtotals and a grand total across all combinations of specified group of n + 1 dimensions (with n being the number of columns as cols and col1 and 1 for where values become null, i.e. undefined).

|Operator |	Return Type |	Description|
|---|---|---|
|cube| RelationalGroupedDataset| Calculates subtotals and a grand total for every permutation of the columns specified.|
|rollup| RelationalGroupedDataset|Calculates subtotals and a grand total over (ordered) combination of groups.|

cube is more than rollup operator, i.e. cube does rollup with aggregation over all the missing combinations given the columns. 

 ```scala
dfNoNull.cube("Date", "Country").agg(sum(col("Quantity")))
  .select("Date", "Country", "sum(Quantity)").orderBy("Date").show()
```
|Date|             Country|sum(Quantity)|
|----|--------------------|-------------|
|null|               Japan|        25218|
|null|            Portugal|        16180|
|null|         Unspecified|         3300|
|null|                null|      5176450|
|null|           Australia|        83653|
|...|...|...|
|null|              Norway|        19247|
|null|           Hong Kong|         4769|
|null|               Spain|        26824|
|null|      Czech Republic|          592|

#### Grouping Metadata

|Grouping ID|	Description|
|---|---|
|3|This will appear for the highest-level aggregation, which will gives us the total quantity regardless of customerId and stockCode.|
|2| This will appear for all aggregations of individual stock codes. This gives us the total quantity per stock code, regardless of customer.|
|1 |This will give us the total quantity on a per-customer basis, regardless of item purchased.|
|0 |This will give us the total quantity for individual customerId and stockCode combinations.|



0 for combinations of each column

1 for subtotals of column 1

2 for subtotals of column 2

 ```scala
import org.apache.spark.sql.functions.{grouping_id, sum, expr}

dfNoNull.cube("customerId", "stockCode").agg(grouping_id(), sum("Quantity"))
.orderBy(expr("grouping_id()").desc)
.show()
```

|customerId|stockCode|grouping_id()|sum(Quantity)|
|----------|---------|-------------|-------------|
|      null|     null|            3|      5176450|
|      null|    23217|            2|         1309|
|      null|   90059E|            2|           19|


#### Pivot

Pivots make it possible for you to convert a row into a column.

 ```scala
val pivoted = dfWithDate.groupBy("date").pivot("Country").sum()
```
This DataFrame will now have a column for every combination of country, numeric variable, and a column specifying the date. For example, for USA we have the following columns: USA_sum(Quantity), USA_sum(UnitPrice), USA_sum(CustomerID). This represents one for each numeric column in our dataset (because we just performed an aggregation over all of them).

 ```scala
pivoted.where("date > '2011-12-05'").select("date" ,"`USA_sum(Quantity)`").show()
```

|      date|USA_sum(Quantity)|
|----------|-----------------|
|2011-12-06|             null|
|2011-12-09|             null|
|2011-12-08|             -196|
|2011-12-07|             null|


### User-Defined Aggregation Functions

You can use UDAFs to compute custom calculations over groups of input data (as opposed to single rows). Spark maintains a single AggregationBuffer to store intermediate results for every group of input data.

To create a UDAF, you must inherit from the UserDefinedAggregateFunction base class and implement the following methods:
- `inputSchema` represents input arguments as a StructType
- `bufferSchema` represents intermediate UDAF results as a StructType
- `dataType` represents the return DataType
- `deterministic` is a Boolean value that specifies whether this UDAF will return the same result for a given input
- `initialize` allows you to initialize values of an aggregation buffer
- `update` describes how you should update the internal buffer based on a given row
- `merge` describes how two aggregation buffers should be merged
- `evaluate` will generate the final result of the aggregation

The following example implements a BoolAnd, which will inform us whether all the rows (for a given column) are true; if they’re not, it will return false:

 ```scala
import org.apache.spark.sql.expressions.MutableAggregationBuffer
import org.apache.spark.sql.expressions.UserDefinedAggregateFunction
import org.apache.spark.sql.Row
import org.apache.spark.sql.types._
class BoolAnd extends UserDefinedAggregateFunction {
  def inputSchema: org.apache.spark.sql.types.StructType =
    StructType(StructField("value", BooleanType) :: Nil)
  def bufferSchema: StructType = StructType(
    StructField("result", BooleanType) :: Nil
  )
  def dataType: DataType = BooleanType
  def deterministic: Boolean = true
  def initialize(buffer: MutableAggregationBuffer): Unit = {
    buffer(0) = true
  }
  def update(buffer: MutableAggregationBuffer, input: Row): Unit = {
    buffer(0) = buffer.getAs[Boolean](0) && input.getAs[Boolean](0)
  }
  def merge(buffer1: MutableAggregationBuffer, buffer2: Row): Unit = {
    buffer1(0) = buffer1.getAs[Boolean](0) && buffer2.getAs[Boolean](0)
  }
  def evaluate(buffer: Row): Any = {
    buffer(0)
  }
}
```
instantiate our class and/or register it as a function:
 ```scala
val ba = new BoolAnd
spark.udf.register("booland", ba)
import org.apache.spark.sql.functions._
spark.range(1)
  .selectExpr("explode(array(TRUE, TRUE, TRUE)) as t")
  .selectExpr("explode(array(TRUE, FALSE, TRUE)) as f", "t")
  .select(ba(col("t")), expr("booland(f)"))
  .show()
```

## Joins

### Joins types

- **Inner joins** (keep rows with keys that exist in the left and right datasets)
- **Outer joins** (keep rows with keys in either the left or right datasets)
- **Left outer joins** (keep rows with keys in the left dataset)
- **Right outer joins** (keep rows with keys in the right dataset)
- **Left semi joins** (keep the rows in the left, and only the left, dataset where the key appears in the right dataset)
- **Left anti joins** (keep the rows in the left, and only the left, dataset where they do not appear in the right dataset)
- **Natural joins** (perform a join by implicitly matching the columns between the two datasets with the same names)
- **Cross (or Cartesian) joins** (match every row in the left dataset with every row in the right dataset)

Data prep for example:
```scala
val person = Seq(
    (0, "Bill Chambers", 0, Seq(100)),
    (1, "Matei Zaharia", 1, Seq(500, 250, 100)),
    (2, "Michael Armbrust", 1, Seq(250, 100)))
  .toDF("id", "name", "graduate_program", "spark_status")
val graduateProgram = Seq(
    (0, "Masters", "School of Information", "UC Berkeley"),
    (2, "Masters", "EECS", "UC Berkeley"),
    (1, "Ph.D.", "EECS", "UC Berkeley"))
  .toDF("id", "degree", "department", "school")
val sparkStatus = Seq(
    (500, "Vice President"),
    (250, "PMC Member"),
    (100, "Contributor"))
  .toDF("id", "status")
  
person.createOrReplaceTempView("person")
graduateProgram.createOrReplaceTempView("graduateProgram")
sparkStatus.createOrReplaceTempView("sparkStatus")
```
### Inner Joins

Default join

 ```scala
val joinExpression = person.col("graduate_program") === graduateProgram.col("id")

person.join(graduateProgram, joinExpression).show()
```
 ```sql
SELECT * 
FROM person 
JOIN graduateProgram ON person.graduate_program = graduateProgram.id
```
Alternative:
 ```scala
var joinType = "inner"
person.join(graduateProgram, joinExpression, joinType).show()
```
 ```sql
 SELECT * FROM person INNER JOIN graduateProgram
  ON person.graduate_program = graduateProgram.id
```
### Outer Joins
 ```scala
joinType = "outer"
person.join(graduateProgram, joinExpression, joinType).show()
```
 ```sql
SELECT * FROM person FULL OUTER JOIN graduateProgram
  ON graduate_program = graduateProgram.id
```
### Left Outer Joins

 ```scala
joinType = "left_outer"
graduateProgram.join(person, joinExpression, joinType).show()
```
 ```sql
SELECT * FROM graduateProgram LEFT OUTER JOIN person
  ON person.graduate_program = graduateProgram.id
```
### Right Outer Joins

 ```scala
joinType = "right_outer"
person.join(graduateProgram, joinExpression, joinType).show()
```
 ```sql
SELECT * FROM person RIGHT OUTER JOIN graduateProgram
  ON person.graduate_program = graduateProgram.id
```
### Left Semi Joins
They do not actually include any values from the right DataFrame. They only compare values to see if the value exists in the second DataFrame. If the value does exist, those rows will be kept in the result, even if there are duplicate keys in the left DataFrame. Think of left semi joins as filters on a DataFrame, as opposed to the function of a conventional join:
 ```scala
joinType = "left_semi"
graduateProgram.join(person, joinExpression, joinType).show()


val gradProgram2 = graduateProgram.union(Seq(
    (0, "Masters", "Duplicated Row", "Duplicated School")).toDF())
gradProgram2.createOrReplaceTempView("gradProgram2")
gradProgram2.join(person, joinExpression, joinType).show()
```
 ```sql
SELECT * FROM gradProgram2 LEFT SEMI JOIN person
  ON gradProgram2.id = person.graduate_program
```
### Left Anti Joins
Left anti joins are the opposite of left semi joins.However, rather than keeping the values that exist in the second DataFrame, they keep only the values that do not have a corresponding key in the second DataFrame.
 ```scala
joinType = "left_anti"
graduateProgram.join(person, joinExpression, joinType).show()
```
 ```sql
SELECT * FROM graduateProgram LEFT ANTI JOIN person
  ON graduateProgram.id = person.graduate_program
```
### Natural Joins

Natural joins make implicit guesses at the columns on which you would like to join.
> Implicit is always dangerous! The following query will give us incorrect results because the two DataFrames/tables share a column name (id), but it means different things in the datasets. You should always use this join with caution.
```sql
SELECT * FROM graduateProgram NATURAL JOIN person
```

### Cross (Cartesian) Joins

The last of our joins are cross-joins or cartesian products. Cross-joins in simplest terms are inner joins that do not specify a predicate. Cross joins will join every single row in the left DataFrame to ever single row in the right DataFrame. This will cause an absolute explosion in the number of rows contained in the resulting DataFrame. If you have 1,000 rows in each DataFrame, the cross-join of these will result in 1,000,000 (1,000 x 1,000) rows. For this reason, you must very explicitly state that you want a cross-join by using the cross join keyword:
 ```scala
joinType = "cross"
graduateProgram.join(person, joinExpression, joinType).show()
```

 ```sql
SELECT * FROM graduateProgram CROSS JOIN person
  ON graduateProgram.id = person.graduate_program
```

If you truly intend to have a cross-join, you can call that out explicitly:
 ```scala
person.crossJoin(graduateProgram).show()
```

 ```sql
SELECT * FROM graduateProgram CROSS JOIN person
```
> You should use cross-joins only if you are absolutely, 100 percent sure that this is the join you need. There is a reason why you need to be explicit when defining a cross-join in Spark. They’re dangerous! Advanced users can set the session-level configuration spark.sql.crossJoin.enable to true in order to allow cross-joins without warnings or without Spark trying to perform another join for you.

#### Joins on Complex Types

 ```scala
import org.apache.spark.sql.functions.expr

person.withColumnRenamed("id", "personId")
  .join(sparkStatus, expr("array_contains(spark_status, id)")).show()
```

 ```sql
SELECT * FROM
  (select id as personId, name, graduate_program, spark_status FROM person)
  INNER JOIN sparkStatus ON array_contains(spark_status, id)
```

#### Handling Duplicate Column Names

Ex:
 ```scala
val gradProgramDupe = graduateProgram.withColumnRenamed("id", "graduate_program")
val joinExpr = gradProgramDupe.col("graduate_program") === person.col(
  "graduate_program")
  //No problem
  person.join(gradProgramDupe, joinExpr).show()
//Problem
person.join(gradProgramDupe, joinExpr).select("graduate_program").show()
//org.apache.spark.sql.AnalysisException: Reference 'graduate_program' is
//ambiguous, could be: graduate_program#40, graduate_program#1079.;
```
**Approach 1: Different join expression**
 ```scala
person.join(gradProgramDupe,"graduate_program").select("graduate_program").show()
```
**Approach 2: Dropping the column after the join**
 ```scala
person.join(gradProgramDupe, joinExpr).drop(person.col("graduate_program"))
  .select("graduate_program").show()
```
> Notice how the column uses the .col method instead of a column function. That allows us to implicitly specify that column by its specific ID.

**Approach 3: Renaming a column before the join**
 ```scala
val joinExpr = person.col("graduate_program") === graduateProgram.col("id")
person.join(graduateProgram, joinExpr).drop(graduateProgram.col("id")).show()
```
### How Spark Performs Joins

#### Communication Strategies
Spark approaches cluster communication in two different ways during joins. It either incurs a shuffle join, which results in an all-to-all communication or a broadcast join. 
**Big table–to–big table**
In a shuffle join, every node talks to every other node and they share data according to which node has a certain key or set of keys (on which you are joining). These joins are expensive because the network can become congested with traffic, especially if your data is not partitioned well.

**Big table–to–small table**
When the table is small enough to fit into the memory of a single worker node, with some breathing room of course, we can optimize our join. Although we can use a big table–to–big table communication strategy, it can often be more efficient to use a broadcast join. What this means is that we will replicate our small DataFrame onto every worker node in the cluster (be it located on one machine or many). Now this sounds expensive. However, what this does is prevent us from performing the all-to-all communication during the entire join process. Instead, we perform it only once at the beginning and then let each individual worker node perform the work without having to wait or communicate with any other worker node. At the beginning of this join will be a large communication, just like in the previous type of join. However, immediately after that first, there will be no further communication between nodes. This means that joins will be performed on every single node individually, making CPU the biggest bottleneck.
 ```scala
val joinExpr = person.col("graduate_program") === graduateProgram.col("id")

person.join(graduateProgram, joinExpr).explain()
```
explicitly give the optimizer a hint that we would like to use a broadcast join by using the correct function around the small DataFrame in question.These are not enforced
 ```scala
import org.apache.spark.sql.functions.broadcast
val joinExpr = person.col("graduate_program") === graduateProgram.col("id")
person.join(broadcast(graduateProgram), joinExpr).explain()
```
```sql
 SELECT /*+ MAPJOIN(graduateProgram) */ * FROM person JOIN graduateProgram
  ON person.graduate_program = graduateProgram.id
```
> This doesn’t come for free either: if you try to broadcast something too large, you can crash your driver node (because that collect is expensive).

**Little table–to–little table**
When performing joins with small tables, it’s usually best to let Spark decide how to join them. You can always force a broadcast join if you’re noticing strange behavior.


## Data Sources

#### Read modes

|Read mode|Description|
| --- | ---|
|`permissive` |Sets all fields to null when it encounters a corrupted record and places all corrupted records in a string column called _corrupt_record|
|`dropMalformed` | Drops the row that contains malformed records|
|`failFast` |Fails immediately upon encountering malformed records|

The default is permissive. `.option("mode", "dropMalformed")`

#### Write API Structure
The core structure for writing data is as follows:
 ```scala
dataFrame.write.format(...).option(...).partitionBy(...).bucketBy(...).sortBy(
  ...).save()
```

E.G.:
 ```scala
dataframe.write.format("csv")
  .option("mode", "OVERWRITE")
  .option("dateFormat", "yyyy-MM-dd")
  .option("path", "path/to/file(s)")
  .save()
```
#### Save modes

|Save mode | Description|
| --- | --- |
|`append` | Appends the output files to the list of files that already exist at that location|
| `overwrite` | Will completely overwrite any data that already exists there|
| `errorIfExists` | Throws an error and fails the write if data or files already exist at the specified location |
| `ignore` | If data or files exist at the location, do nothing with the current DataFrame|

The default is errorIfExists. `.option("mode", "OVERWRITE")`


### CSV Files
#### CSV Options
|Read/write	|Key	|Potential values	|Default|	Description|
| --- |            -----------------|------------|---|---|
|Both |`sep`                        |Any single string character|,|The single character that is used as separator for each field and value.|
|Both |`header`                     |true, false|false|A Boolean flag that declares whether the first line in the file(s) are the names of the columns.|
|Read |`escape`                     |Any string character|\|The character Spark should use to escape other characters in the file.|
|Read |`inferSchema`                |true, false|false|Specifies whether Spark should infer column types when reading the file.|
|Read |`ignoreLeadingWhiteSpace`    |true, false|false|Declares whether leading spaces from values being read should be skipped.|
|Read |`ignoreTrailingWhiteSpace`   |true, false|false|Declares whether trailing spaces from values being read should be skipped.|
|Both |`nullValue`                  |Any string character|“”|Declares what character represents a null value in the file.|
|Both |`nanValue`                   |Any string character|NaN|Declares what character represents a NaN or missing character in the CSV file.|
|Both |`positiveInf`                |Any string or character|Inf|Declares what character(s) represent a positive infinite value.|
|Both |`negativeInf`                |Any string or character|-Inf|Declares what character(s) represent a negative infinite value.|
|Both |`compression` or codec       |None, uncompressed, bzip2, deflate, gzip, lz4, or snappy|none|Declares what compression codec Spark should use to read or write the file.|
|Both |`dateFormat`                 |Any string or character that conforms to java’s SimpleDataFormat.|yyyy-MM-dd|Declares the date format for any columns that are date type.|
|Both |`timestampFormat`            |Any string or character that conforms to java’s SimpleDataFormat.|yyyy-MM-dd’T’HH:mm​:ss.SSSZZ|Declares the timestamp format for any columns that are timestamp type.|
|Read |`maxColumns`                 |Any integer|20480|Declares the maximum number of columns in the file.|
|Read |`maxCharsPerColumn`          |Any integer|1000000|Declares the maximum number of characters in a column.|
|Read |`escapeQuotes`               |true, false|true|Declares whether Spark should escape quotes that are found in lines.|
|Read |`maxMalformedLogPerPartition`|Any integer|10|Sets the maximum number of malformed rows Spark will log for each partition. Malformed records beyond this number will be ignored.|
|Write|`quoteAll`                   |true, false|false |Specifies whether all values should be enclosed in quotes, as opposed to just escaping values that have a quote character.|
|Read |`multiLine`                  |true, false|false|This option allows you to read multiline CSV files where each logical row in the CSV file might span multiple rows in the file itself.|

#### Reading CSV Files
 ```scala
spark.read.format("csv")
```
Will stop if a record is wrongly parsed or type mismatch
 ```scala
import org.apache.spark.sql.types.{StructField, StructType, StringType, LongType}
val myManualSchema = new StructType(Array(
  new StructField("DEST_COUNTRY_NAME", StringType, true),
  new StructField("ORIGIN_COUNTRY_NAME", StringType, true),
  new StructField("count", LongType, false)
))
spark.read.format("csv")
  .option("header", "true")
  .option("mode", "FAILFAST")
  .schema(myManualSchema)
  .load("/data/flight-data/csv/2010-summary.csv")
  .show(5)
```
#### Writing CSV Files


 ```scala
csvFile.write.format("csv").mode("overwrite").option("sep", "\t")
  .save("/tmp/my-tsv-file.tsv")
```

### JSON Files

#### JSON Options

|Read/write	|Key	|Potential values	|Default	|Description|
| ---| --- |---|---|---|
|Both|compression or codec|None, uncompressed, bzip2, deflate, gzip, lz4, or snappy|none|Declares what compression codec Spark should use to read or write the file.|
|Both|dateFormat|Any string or character that conforms to Java’s SimpleDataFormat.|yyyy-MM-dd|Declares the date format for any columns that are date type.|
|Both|timestampFormat|Any string or character that conforms to Java’s SimpleDataFormat.|yyyy-MM-dd’T’HH:​mm:ss.SSSZZ|Declares the timestamp format for any columns that are timestamp type.|
|Read|primitiveAsString|true, false|false|Infers all primitive values as string type.|
|Read|allowComments|true, false|false|Ignores Java/C++ style comment in JSON records.|
|Read|allowUnquotedFieldNames|true, false|false|Allows unquoted JSON field names.|
|Read|allowSingleQuotes|true, false|true|Allows single quotes in addition to double quotes.|
|Read|allowNumericLeadingZeros|true, false|false|Allows leading zeroes in numbers (e.g., 00012).|
|Read|allowBackslashEscapingAnyCharacter|true, false|false|Allows accepting quoting of all characters using backslash quoting mechanism.|
|Read|columnNameOfCorruptRecord|Any string|Value of spark.sql.column&NameOfCorruptRecord|Allows renaming the new field having a malformed string created by permissive mode. This will override the configuration value.|
|Read|multiLine|true, false|false|Allows for reading in non-line-delimited JSON files.|

#### Reading JSON Files

 ```scala
spark.read.format("json").option("mode", "FAILFAST").schema(myManualSchema)
  .load("/data/flight-data/json/2010-summary.json").show(5)
```
#### Reading JSON Files
 ```scala
csvFile.write.format("json").mode("overwrite").save("/tmp/my-json-file.json")
```
#### Reading JSON Files

 ```scala
csvFile.write.format("json").mode("overwrite").save("/tmp/my-json-file.json")
```
### Parquet Files

#### Reading Parquet Files
Parquet has very few options because it enforces its own schema when storing data
 ```scala
spark.read.format("parquet")
  .load("/data/flight-data/parquet/2010-summary.parquet").show(5)
```

#### Parquet option

|Read/Write	|Key	|Potential Values	|Default	|Description|
| --- | --- | --- | --- | --- |
|Write|compression or codec|None, uncompressed, bzip2, deflate, gzip, lz4, or snappy|None|Declares what compression codec Spark should use to read or write the file.|
|Read|mergeSchema|true, false|Value of the configuration spark.sql.parquet.mergeSchema|You can incrementally add columns to newly written Parquet files in the same table/folder. Use this option to enable or disable this feature.|


#### Writing Parquet Files
 ```scala
csvFile.write.format("parquet").mode("overwrite")
  .save("/tmp/my-parquet-file.parquet")
```
### ORC Files
#### Reading Orc Files
 ```scala
spark.read.format("orc").load("/data/flight-data/orc/2010-summary.orc").show(5)
```

#### Writing Orc Files
 ```scala
csvFile.write.format("orc").mode("overwrite").save("/tmp/my-json-file.orc")
```

### SQL Databases
Pass the driver in spark-shell
 ```bash
./bin/spark-shell \
--driver-class-path postgresql-9.4.1207.jar \
--jars postgresql-9.4.1207.jar
```
|Property Name	|Meaning|
| --- | --- |
|url|The JDBC URL to which to connect. The source-specific connection properties can be specified in the URL; for example, jdbc:postgresql://localhost/test?user=fred&password=secret.|
|dbtable|The JDBC table to read. Note that anything that is valid in a FROM clause of a SQL query can be used. For example, instead of a full table you could also use a subquery in parentheses.|
|driver|The class name of the JDBC driver to use to connect to this URL.|
|partitionColumn, lowerBound, upperBound|If any one of these options is specified, then all others must be set as well. In addition, numPartitions must be specified. These properties describe how to partition the table when reading in parallel from multiple workers. partitionColumn must be a numeric column from the table in question. Notice that lowerBound and upperBound are used only to decide the partition stride, not for filtering the rows in the table. Thus, all rows in the table will be partitioned and returned. This option applies only to reading.|
|numPartitions|The maximum number of partitions that can be used for parallelism in table reading and writing. This also determines the maximum number of concurrent JDBC connections. If the number of partitions to write exceeds this limit, we decrease it to this limit by calling coalesce(numPartitions) before writing.|
|fetchsize|The JDBC fetch size, which determines how many rows to fetch per round trip. This can help performance on JDBC drivers, which default to low fetch size (e.g., Oracle with 10 rows). This option applies only to reading.|
|batchsize|The JDBC batch size, which determines how many rows to insert per round trip. This can help performance on JDBC drivers. This option applies only to writing. The default is 1000.|
|isolationLevel|The transaction isolation level, which applies to current connection. It can be one of NONE, READ_COMMITTED, READ_UNCOMMITTED, REPEATABLE_READ, or SERIALIZABLE, corresponding to standard transaction isolation levels defined by JDBC’s Connection object. The default is READ_UNCOMMITTED. This option applies only to writing. For more information, refer to the documentation in java.sql.Connection.|
|truncate|This is a JDBC writer-related option. When SaveMode.Overwrite is enabled, Spark truncates an existing table instead of dropping and re-creating it. This can be more efficient, and it prevents the table metadata (e.g., indices) from being removed. However, it will not work in some cases, such as when the new data has a different schema. The default is false. This option applies only to writing.|
|createTableOptions|This is a JDBC writer-related option. If specified, this option allows setting of database-specific table and partition options when creating a table (e.g., CREATE TABLE t (name string) ENGINE=InnoDB). This option applies only to writing.|
|createTableColumnTypes|The database column data types to use instead of the defaults, when creating the table. Data type information should be specified in the same format as CREATE TABLE columns syntax (e.g., “name CHAR(64), comments VARCHAR(1024)”). The specified types should be valid Spark SQL data types. This option applies only to writing.|

#### Reading from SQL Databases
 ```scala
val driver =  "org.sqlite.JDBC"
val path = "/data/flight-data/jdbc/my-sqlite.db"
val url = s"jdbc:sqlite:/${path}"
val tablename = "flight_info"
```
Test connection
 ```scala
import java.sql.DriverManager
val connection = DriverManager.getConnection(url)
connection.isClosed()
connection.close()
```
Read in SQLite
 ```scala
val dbDataFrame = spark.read.format("jdbc").option("url", url)
  .option("dbtable", tablename).option("driver",  driver).load()
```
Read in SQL with password and user
```scala
val pgDF = spark.read
  .format("jdbc")
  .option("driver", "org.postgresql.Driver")
  .option("url", "jdbc:postgresql://database_server")
  .option("dbtable", "schema.tablename")
  .option("user", "username").option("password","my-secret-password").load()
```
##### Query Pushdown

First, Spark makes a best-effort attempt to filter data in the database itself before creating the DataFrame

##### Query SQL directly to Database
Rather than specifying a table name, you just specify a SQL query in parenthesis
```scala
val pushdownQuery = """(SELECT DISTINCT(DEST_COUNTRY_NAME) FROM flight_info)
  AS flight_info"""
val dbDataFrame = spark.read.format("jdbc")
  .option("url", url).option("dbtable", pushdownQuery).option("driver",  driver)
  .load()
```
##### Reading from databases in parallel
```scala
val dbDataFrame = spark.read.format("jdbc")
  .option("url", url).option("dbtable", tablename).option("driver", driver)
  .option("numPartitions", 10).load()
```

Using jdbc directly
```scala
val props = new java.util.Properties
props.setProperty("driver", "org.sqlite.JDBC")
val predicates = Array(
  "DEST_COUNTRY_NAME = 'Sweden' OR ORIGIN_COUNTRY_NAME = 'Sweden'",
  "DEST_COUNTRY_NAME = 'Anguilla' OR ORIGIN_COUNTRY_NAME = 'Anguilla'")
spark.read.jdbc(url, tablename, predicates, props).show()
spark.read.jdbc(url, tablename, predicates, props).rdd.getNumPartitions // 2
```
##### Partitioning based on a sliding window
```scala
val colName = "count"
val lowerBound = 0L
val upperBound = 348113L // this is the max count in our database
val numPartitions = 10

spark.read.jdbc(url,tablename,colName,lowerBound,upperBound,numPartitions,props)
  .count()
```
#### Writing to SQL Databases
```scala
val newPath = "jdbc:sqlite://tmp/my-sqlite.db"
csvFile.write.mode("overwrite").jdbc(newPath, tablename, props)

csvFile.write.mode("append").jdbc(newPath, tablename, props)
```
### Text Files
Each line in the file becomes a record in the DataFrame. It is then up to you to transform it accordingly.

#### Reading Text Files

```scala
spark.read.textFile("/data/flight-data/csv/2010-summary.csv")
  .selectExpr("split(value, ',') as rows").show()
```
#### Writing Text Files

```scala
csvFile.select("DEST_COUNTRY_NAME").write.text("/tmp/simple-text-file.txt")

csvFile.limit(10).select("DEST_COUNTRY_NAME", "count")
  .write.partitionBy("count").text("/tmp/five-csv-files2.csv")
```
### Partitioning
Writing by column, each file represent a column of the dataframe

```scala
csvFile.limit(10).write.mode("overwrite").partitionBy("DEST_COUNTRY_NAME")
  .save("/tmp/partitioned-files.parquet")
```
### Bucketing
The data is prepartitioned according to how you expect to use that data later on, meaning you can avoid expensive shuffles when joining or aggregating.
```scala
val numberBuckets = 10
val columnToBucketBy = "count"

csvFile.write.format("parquet").mode("overwrite")
  .bucketBy(numberBuckets, columnToBucketBy).saveAsTable("bucketedFiles")
```
### Complex Types
Only in ORC and Parquet


### Managing File Size
You can use the maxRecordsPerFile option and specify a number of your choosing.
```scala
df.write.option("maxRecordsPerFile", 5000)
```

##  Spark SQL

### The Hive metastore

To connect to the Hive metastore you need to set the Metastore version (spark.sql.hive.metastore.version) to correspond to the proper Hive metastore that you’re accessing. By default, this value is 1.2.1. You also need to set spark.sql.hive.metastore.jars if you’re going to change the way that the HiveMetastoreClient is initialized. Spark uses the default versions, but you can also specify Maven repositories or a classpath in the standard format for the Java Virtual Machine (JVM). In addition, you might need to supply proper class prefixes in order to communicate with different databases that store the Hive metastore. You’ll set these as shared prefixes that both Spark and Hive will share (spark.sql.hive.metastore.sharedPrefixes).

[Docs] http://spark.apache.org/docs/latest/sql-programming-guide.html#interacting-with-different-versions-of-hive-metastore)

### How to Run Spark SQL Queries

#### Spark SQL CLI

```bash
./bin/spark-sql
```
You configure Hive by placing your hive-site.xml, core-site.xml, and hdfs-site.xml files in conf/.

You can completely interoperate between SQL and DataFrames
```scala
spark.read.json("/data/flight-data/json/2015-summary.json")
  .createOrReplaceTempView("some_sql_view") // DF => SQL

spark.sql("""
SELECT DEST_COUNTRY_NAME, sum(count)
FROM some_sql_view GROUP BY DEST_COUNTRY_NAME
""")
  .where("DEST_COUNTRY_NAME like 'S%'").where("`sum(count)` > 10")
  .count() // SQL => DF
```

#### SparkSQL Thrift JDBC/ODBC Server
This accept remotly connections to spark via JDBC/ODBC
To start the JDBC/ODBC server, run the following in the Spark directory:
```bash
./sbin/start-thriftserver.sh
```
By default, the server listens on localhost:10000.
For environment configuration, use this:
```bash
export HIVE_SERVER2_THRIFT_PORT=<listening-port>
export HIVE_SERVER2_THRIFT_BIND_HOST=<listening-host>
./sbin/start-thriftserver.sh \
  --master <master-uri> \
  ...
```
For system properties:

```bash
./sbin/start-thriftserver.sh \
  --hiveconf hive.server2.thrift.port=<listening-port> \
  --hiveconf hive.server2.thrift.bind.host=<listening-host> \
  --master <master-uri>
  ...
```
You can then test this connection by running the following commands:
```bash
./bin/beeline
beeline> !connect jdbc:hive2://localhost:10000
```

### Catalog
The Catalog is an abstraction for the storage of metadata about the data stored in your tables as well as other helpful things like databases, tables, functions, and views. The catalog is available in the `org.apache.spark.sql.catalog.Catalog` package and contains a number of helpful functions for doing things like listing tables, databases, and functions.

### Tables

#### Spark-Managed Tables
The data within the tables as well as the data about the tables; that is, the metadata. When you use `saveAsTable` on a DataFrame, you are creating a managed table for which Spark will track of all of the relevant information.
You can set the default Hive warehouse location by setting the `spark.sql.warehouse.dir` configuration to the directory of your choosing when you create your SparkSession. You can also see tables in a specific database by using the query `show tables IN databaseName`, where databaseName represents the name of the database that you want to query.

#### Creating Tables
```sql
CREATE TABLE flights (
  DEST_COUNTRY_NAME STRING, ORIGIN_COUNTRY_NAME STRING, count LONG)
USING JSON OPTIONS (path '/data/flight-data/json/2015-summary.json')  <--- Fill data within
```
> **USING AND STORED AS**
> The specification of the USING syntax in the previous example is of significant importance. If you do not specify the format, Spark will default to a Hive SerDe configuration. This has performance implications for future readers and writers because Hive SerDes are much slower than Spark’s native serialization. Hive users can also use the STORED AS syntax to specify that this should be a Hive table.

Add comments to tables

```sql
CREATE TABLE flights_csv (
  DEST_COUNTRY_NAME STRING,
  ORIGIN_COUNTRY_NAME STRING COMMENT "remember, the US will be most prevalent",
  count LONG)
USING csv OPTIONS (header true, path '/data/flight-data/csv/2015-summary.csv')
```
It is possible to create a table from a query:
```sql
CREATE TABLE flights_from_select USING parquet AS SELECT * FROM flights
```
create a table only if it does not currently exist:
> In this example, we are creating a Hive-compatible table because we did not explicitly specify the format via USING. We can also do the following:
```sql
CREATE TABLE IF NOT EXISTS flights_from_select
  AS SELECT * FROM flights
```
Writing out a partitioned dataset:
```sql
CREATE TABLE partitioned_flights USING parquet PARTITIONED BY (DEST_COUNTRY_NAME)
AS SELECT DEST_COUNTRY_NAME, ORIGIN_COUNTRY_NAME, count FROM flights LIMIT 5
```
These tables will be available in Spark even through sessions; temporary tables do not currently exist in Spark.

#### Creating External Tables
In the example that follows, we create an unmanaged table. Spark will manage the table’s metadata; however, the files are not managed by Spark at all.
```sql
CREATE EXTERNAL TABLE hive_flights (
  DEST_COUNTRY_NAME STRING, ORIGIN_COUNTRY_NAME STRING, count LONG)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' LOCATION '/data/flight-data-hive/'
```
create an external table from a select clause:
```sql
CREATE EXTERNAL TABLE hive_flights_2
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION '/data/flight-data-hive/' AS SELECT * FROM flights
```
#### Inserting into Tables
```sql
INSERT INTO flights_from_select
  SELECT DEST_COUNTRY_NAME, ORIGIN_COUNTRY_NAME, count FROM flights LIMIT 20
```
You can optionally provide a partition specification if you want to write only into a certain partition. (Por optimized purpose)
```sql
INSERT INTO partitioned_flights
  PARTITION (DEST_COUNTRY_NAME="UNITED STATES")
  SELECT count, ORIGIN_COUNTRY_NAME FROM flights
  WHERE DEST_COUNTRY_NAME='UNITED STATES' LIMIT 12
```
#### Describing Table Metadata
```sql
DESCRIBE TABLE flights_csv
```
You can also see the partitioning scheme for the data by using the following (note, however, that this works only on partitioned tables):
```scala
SHOW PARTITIONS partitioned_flights
```
#### Refreshing Table Metadata
```scala
REFRESH table partitioned_flights
```
REPAIR TABLE refreshes the partitions maintained in the catalog for that given table. This command’s focus is on collecting new partition information — an example might be writing out a new partition manually and the need to repair the table accordingly:
```sql
MSCK REPAIR TABLE partitioned_flights
```
#### Dropping Tables

```sql
DROP TABLE flights_csv;
DROP TABLE IF EXISTS flights_csv;
```
>Dropping a table deletes the data in the table, so you need to be very careful when doing this.

#### Dropping unmanaged tables
If you are dropping an unmanaged table (e.g., hive_flights), no data will be removed but you will no longer be able to refer to this data by the table name.

#### Caching Tables
```scala
CACHE TABLE flights
UNCACHE TABLE FLIGHTS
```

### Views
view specifies a set of transformations on top of an existing table — **basically just saved query plans**

#### Creating Views
```sql
CREATE VIEW just_usa_view AS
  SELECT * FROM flights WHERE dest_country_name = 'United States'

-- Only aveilable in the current session
CREATE TEMP VIEW just_usa_view_temp AS
  SELECT * FROM flights WHERE dest_country_name = 'United States'  

-- Or, it can be a global temp view. Global temp views are resolved regardless of database and are viewable across the entire Spark application, but they are removed at the end of the session:
 CREATE GLOBAL TEMP VIEW just_usa_global_view_temp AS
  SELECT * FROM flights WHERE dest_country_name = 'United States'

SHOW TABLES

-- Replace
CREATE OR REPLACE TEMP VIEW just_usa_view_temp AS
  SELECT * FROM flights WHERE dest_country_name = 'United States'

--Query like another table
  SELECT * FROM just_usa_view_temp
```

```scala
val flights = spark.read.format("json")
  .load("/data/flight-data/json/2015-summary.json")
val just_usa_df = flights.where("dest_country_name = 'United States'")
just_usa_df.selectExpr("*").explain
```
Is the same like

```sql
EXPLAIN SELECT * FROM just_usa_view --NOT just_usa_view from above
--equivalently
EXPLAIN SELECT * FROM flights WHERE dest_country_name = 'United States'
```

#### Dropping Views
```sql
DROP VIEW IF EXISTS just_usa_view;
```
### Databases
If you not define one, Spark will use the default one. All SQL statments execute within the context of a dabase
See databases
```sql
SHOW DATABASES

CREATE DATABASE some_db

USE some_db

SHOW tables

SELECT * FROM flights -- fails with table/view not found

SELECT * FROM default.flights

SELECT current_database()

USE default;

DROP DATABASE IF EXISTS some_db;
```

### Select Statements
```sql
SELECT [ALL|DISTINCT] named_expression[, named_expression, ...]
    FROM relation[, relation, ...]
    [lateral_view[, lateral_view, ...]]
    [WHERE boolean_expression]
    [aggregation [HAVING boolean_expression]]
    [ORDER BY sort_expressions]
    [CLUSTER BY expressions]
    [DISTRIBUTE BY expressions]
    [SORT BY sort_expressions]
    [WINDOW named_window[, WINDOW named_window, ...]]
    [LIMIT num_rows]

named_expression:
    : expression [AS alias]

relation:
    | join_relation
    | (table_name|query|relation) [sample] [AS alias]
    : VALUES (expressions)[, (expressions), ...]
          [AS (column_name[, column_name, ...])]

expressions:
    : expression[, expression, ...]

sort_expressions:
    : expression [ASC|DESC][, expression [ASC|DESC], ...]
```
#### case…when…then Statements

```sql
SELECT
  CASE WHEN DEST_COUNTRY_NAME = 'UNITED STATES' THEN 1
       WHEN DEST_COUNTRY_NAME = 'Egypt' THEN 0
       ELSE -1 END
FROM partitioned_flights
```
### Advanced Topics

#### Complex Types

##### Structs

```sql
CREATE VIEW IF NOT EXISTS nested_data AS
  SELECT (DEST_COUNTRY_NAME, ORIGIN_COUNTRY_NAME) as country, count FROM flights

SELECT * FROM nested_data
SELECT country.DEST_COUNTRY_NAME, count FROM nested_data

SELECT country.*, count FROM nested_data
```
##### Lists


```sql
-- You can also use the function collect_set, which creates an array without duplicate values. These are both aggregation functions and therefore can be specified only in aggregations:
SELECT DEST_COUNTRY_NAME as new_name, collect_list(count) as flight_counts,
  collect_set(ORIGIN_COUNTRY_NAME) as origin_set
FROM flights GROUP BY DEST_COUNTRY_NAME

-- create an array manually within a column
SELECT DEST_COUNTRY_NAME, ARRAY(1, 2, 3) FROM flights

-- lists by position by using a Python-like array query syntax
SELECT DEST_COUNTRY_NAME as new_name, collect_list(count)[0]
FROM flights GROUP BY DEST_COUNTRY_NAME

-- convert an array back into rows.
CREATE OR REPLACE TEMP VIEW flights_agg AS
  SELECT DEST_COUNTRY_NAME, collect_list(count) as collected_counts
  FROM flights GROUP BY DEST_COUNTRY_NAME

-- Opposite
SELECT explode(collected_counts), DEST_COUNTRY_NAME FROM flights_agg
```
##### Functions

```sql
SHOW FUNCTIONS
SHOW SYSTEM FUNCTIONS

SHOW USER FUNCTIONS

SHOW FUNCTIONS "s*"; --functions that starts with
```
###### User-defined functions
```scala
def power3(number:Double):Double = number * number * number
spark.udf.register("power3", power3(_:Double):Double)
```

```sql
SELECT count, power3(count) FROM flights
```
You can also register functions through the Hive CREATE TEMPORARY FUNCTION syntax.

#### Subqueries

##### Uncorrelated predicate subqueries
```sql
SELECT dest_country_name FROM flights
GROUP BY dest_country_name ORDER BY sum(count) DESC LIMIT 5
```

|dest_country_name|
|-----------------|
|    United States|
|           Canada|
|           Mexico|
|   United Kingdom|
|            Japan|


```sql
SELECT * FROM flights
WHERE origin_country_name IN (SELECT dest_country_name FROM flights
      GROUP BY dest_country_name ORDER BY sum(count) DESC LIMIT 5)
```
This query is uncorrelated because it does not include any information from the outer scope of the query. It’s a query that you can run on its own.

##### Correlated predicate subqueries

```sql
SELECT * FROM flights f1
WHERE EXISTS (SELECT 1 FROM flights f2
            WHERE f1.dest_country_name = f2.origin_country_name)
AND EXISTS (SELECT 1 FROM flights f2
            WHERE f2.dest_country_name = f1.origin_country_name)
```
`EXISTS` just checks for some existence in the subquery and returns true if there is a value. You can flip this by placing the NOT operator in front of it.

##### Uncorrelated scalar queries

```sql
SELECT *, (SELECT max(count) FROM flights) AS maximum FROM flights
```
### Miscellaneous Features

#### Configurations

|Property Name|	Default	|Meaning|
|---|---|---|
|`spark.sql.inMemoryColumnarStorage.compressed` |true |When set to true, Spark SQL automatically selects a compression codec for each column based on statistics of the data.|
|`spark.sql.inMemoryColumnarStorage.batchSize` |10000 |Controls the size of batches for columnar caching. Larger batch sizes can improve memory utilization and compression, but risk OutOfMemoryErrors (OOMs) when caching data.|
|`spark.sql.files.maxPartitionBytes` |134217728 (128 MB) |The maximum number of bytes to pack into a single partition when reading files.|
|`spark.sql.files.openCostInBytes` |4194304 (4 MB) |The estimated cost to open a file, measured by the number of bytes that could be scanned in the same time. This is used when putting multiple files into a partition. It is better to overestimate; that way the partitions with small files will be faster than partitions with bigger files (which is scheduled first).|
|`spark.sql.broadcastTimeout` |300 |Timeout in seconds for the broadcast wait time in broadcast joins.|
|`spark.sql.autoBroadcastJoinThreshold` |10485760 (10 MB) |Configures the maximum size in bytes for a table that will be broadcast to all worker nodes when performing a join. You can disable broadcasting by setting this value to -1. Note that currently statistics are supported only for Hive Metastore tables for which the command ANALYZE TABLE COMPUTE STATISTICS noscan has been run.|
|`spark.sql.shuffle.partitions` |200 |Configures the number of partitions to use when shuffling data for joins or aggregations.|

#### Setting Configuration Values in SQL

```sql
SET spark.sql.shuffle.partitions=20
```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

```scala

```

