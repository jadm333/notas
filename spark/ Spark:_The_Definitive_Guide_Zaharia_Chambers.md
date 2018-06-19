# Spark: The Definitive Guide: Big Data Processing Made Simple
###### Matei Zaharia, Bill Chambers


### Chp 1
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

### Chp 3

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

### Chp 3

Para correr un script de spark
```bash
./bin/spark-submit 
  --class org.apache.spark.examples.SparkPi
  --master local 
  ./examples/jars/spark-examples_2.11-2.2.0.jar 10
```

## Schemas
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

## Columns

```scala
import org.apache.spark.sql.functions.{col, column}
col("someColumnName")
column("someColumnName")
```

### Explicit column references
```scala
df.col("count")
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

## DataFrame Transformations

- We can add rows or columns
- We can remove rows or columns
- We can transform a row into a column (or vice versa)
- We can change the order of rows based on the values in columns

### Creating DataFrames

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
### select and selectExpr

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

### Literals
Son lo mismo:
```scala
import org.apache.spark.sql.functions.lit
df.select(expr("*"), lit(1).as("One")).show(2)

val sqlWay = spark.sql("""
SELECT *, 1 as One FROM flight_data_2015 LIMIT 2
""").show()
```

### Agregar columna

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

### Renombrar columna

```scala
df.withColumnRenamed("DEST_COUNTRY_NAME", "dest").columns
```

### Reserved Characters and Keywords
```scala
import org.apache.spark.sql.functions.expr

val dfWithLongColName = df.withColumn(
  "This Long Column-Name",
  expr("ORIGIN_COUNTRY_NAME"))
```

### Case Sensitive

```sql
set spark.sql.caseSensitive true
```
### Quitar columnas

```scala
dfWithLongColName.drop("ORIGIN_COUNTRY_NAME", "DEST_COUNTRY_NAME")
```

### Changing a Column’s Type (cast)

```scala
df.withColumn("count2", col("count").cast("long"))
```
### Filtrar columnas

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

### Unique rows

```scala
df.select("ORIGIN_COUNTRY_NAME", "DEST_COUNTRY_NAME").distinct().count()
```

```sql
SELECT COUNT(DISTINCT(ORIGIN_COUNTRY_NAME, DEST_COUNTRY_NAME)) FROM dfTable
```
### Random Samples

```scala
val seed = 5
val withReplacement = false
val fraction = 0.5
df.sample(withReplacement, fraction, seed).count()
```

### Random Splits

```scala
val seed = 96845
val dataFrames = df.randomSplit(Array(0.25, 0.75), seed)
dataFrames(0).count() > dataFrames(1).count() // False
```

### Concatenating and Appending Rows (Union)

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

### Sorting Rows

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

### limit

```scala
df.limit(5).show()
```

```sql
SELECT * FROM dfTable LIMIT 6
```

### `coalesce` and `repartition`

```scala
df.rdd.getNumPartitions // 1
df.repartition(5)
```

If you know that you’re going to be filtering by a certain column often, it can be worth repartitioning based on that column:

```scala
df.repartition(col("DEST_COUNTRY_NAME"))
df.repartition(5, col("DEST_COUNTRY_NAME"))
```

# ¿`coalesce`?


### Collecting Rows to the Driver

```scala
val collectDF = df.limit(10)
collectDF.take(5) // take works with an Integer count
collectDF.show() // this prints it out nicely
collectDF.show(5, false)
collectDF.collect()

// to iterator
collectDF.toLocalIterator()
```

## Chp 6 Working with Different Types of Data

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

#### Lit

Se crea una columna literal (i.e. el mismo valor para todos los registros)

```scala
import org.apache.spark.sql.functions.lit
df.select(lit(5), lit("five"), lit(5.0))
```
```sql
SELECT 5, "five", 5.0
```

#### Booleans

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

#### Numbers columns

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

#### Strings columns

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
##### Regular Expressions regex

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

#### Dates and Timestamps

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


#### NULLS

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

###### Ordering NULLS

`asc_nulls_first`, `desc_nulls_first`, `asc_nulls_last`, or `desc_nulls_last`

#### Complex Types

There are three kinds of complex types: structs, arrays, and maps

##### Structs
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
##### Arrays

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


##### Maps
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


#### JSON

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


#### User-Defined Functions (UDFs)

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

## Chp 6 Aggregations

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

** Big table–to–small table**
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