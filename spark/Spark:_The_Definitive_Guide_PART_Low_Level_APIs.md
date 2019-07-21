# Spark: The Definitive Guide: Part III. Low-Level APIs
**Matei Zaharia, Bill Chambers**


## Resilient Distributed Datasets (RDDs)

### Creating RDDs

```scala
// in Scala: converts a Dataset[Long] to RDD[Long]
spark.range(500).rdd

//Operate on RDD
spark.range(10).toDF().rdd.map(rowObject => rowObject.getLong(0))

spark.range(10).rdd.toDF()
```

### From a Local Collection
```scala
val myCollection = "Spark The Definitive Guide : Big Data Processing Made Simple"
    .split(" ")
val words = spark.sparkContext.parallelize(myCollection, 2)

// you can then name this RDD to show up in the Spark UI according to a given name:
words.setName("myWords")
words.name // myWords
```
### From Data Sources
```scala
spark.sparkContext.textFile("/some/path/withTextFiles")
spark.sparkContext.wholeTextFiles("/some/path/withTextFiles") //file becomes record
```
## Manipulating RDDs

### Transformations

#### distinct
```scala
words.distinct().count()
```

#### filter

```scala
// First create function to be apply
def startsWithS(individual:String) = {
     individual.startsWith("S")
}
// Then apply it
words.filter(word => startsWithS(word)).collect()
```
#### map

```scala
val words2 = words.map(word => (word, word(0), word.startsWith("S")))

words2.filter(record => record._3).take(5)
```

#### flatMap

```scala
words.flatMap(word => word.toSeq).take(5)
```
#### sort
```scala
words.sortBy(word => word.length() * -1).take(2)
```
#### Random Splits
```scala
val fiftyFiftySplit = words.randomSplit(Array[Double](0.5, 0.5))
```

### Actions

#### reduce
```scala
spark.sparkContext.parallelize(1 to 20).reduce(_ + _) // 210

def wordLengthReducer(leftWord:String, rightWord:String): String = {
  if (leftWord.length > rightWord.length)
    return leftWord
  else
    return rightWord
}

words.reduce(wordLengthReducer)
```
#### count
```scala
words.count()
```
#### countApprox
```scala
val confidence = 0.95
val timeoutMilliseconds = 400
words.countApprox(timeoutMilliseconds, confidence)
```
#### countApproxDistinct
In the first implementation, the argument we pass into the function is the relative accuracy. Smaller values create counters that require more space. The value must be greater than 0.000017:
```scala
words.countApproxDistinct(0.05)
```
With the other implementation you have a bit more control; you specify the relative accuracy based on two parameters: one for “regular” data and another for a sparse representation.The two arguments are p and sp where p is precision and sp is sparse precision. The relative accuracy is approximately 1.054 / sqrt(2P). Setting a nonzero (sp > p) can reduce the memory consumption and increase accuracy when the cardinality is small. Both values are integers:
```scala
words.countApproxDistinct(4, 10)
```
#### countByValue
This method counts the number of values in a given RDD.
```scala
words.countByValue()
```
#### countByValueApprox
```scala
words.countByValueApprox(1000, 0.95)
```
#### first
```scala
words.first()
```
#### max and min
```scala
spark.sparkContext.parallelize(1 to 20).max()
spark.sparkContext.parallelize(1 to 20).min()
```
#### take
```scala
words.take(5)
words.takeOrdered(5)
words.top(5)
val withReplacement = true
val numberToTake = 6
val randomSeed = 100L
words.takeSample(withReplacement, numberToTake, randomSeed)
```
### Saving Files
#### saveAsTextFile
```scala
words.saveAsTextFile("file:/tmp/bookTitle")

import org.apache.hadoop.io.compress.BZip2Codec
words.saveAsTextFile("file:/tmp/bookTitleCompressed", classOf[BZip2Codec])
```
#### saveAsTeSequenceFilesxtFile
A sequenceFile is a flat file consisting of binary key–value pairs. It is extensively used in MapReduce as input/output formats.
```scala
words.saveAsObjectFile("/tmp/my/sequenceFilePath")
```
### Caching
```scala
words.cache()
words.getStorageLevel
```
### Checkpointing
Checkpointing is the act of saving an RDD to disk so that future references to this RDD point to those intermediate partitions on disk rather than recomputing the RDD from its original source. Similar to caching but in disk
```scala
spark.sparkContext.setCheckpointDir("/some/path/for/checkpointing")
words.checkpoint()
```
### Pipe RDDs to System Commands
With pipe, you can return an RDD created by piping elements to a forked external process.
```scala
words.pipe("wc -l").collect() //wc -l is a bash command
```
### mapPartitions
```scala
words.mapPartitions(part => Iterator[Int](1)).sum() // 2
```

```scala
def indexedFunc(partitionIndex:Int, withinPartIterator: Iterator[String]) = {
  withinPartIterator.toList.map(
    value => s"Partition: $partitionIndex => $value").iterator
}
words.mapPartitionsWithIndex(indexedFunc).collect()
```
### foreachPartition
foreachPartition simply iterates over all the partitions of the data. The difference is that the function has no return value.
```scala
words.foreachPartition { iter =>
  import java.io._
  import scala.util.Random
  val randomFileName = new Random().nextInt()
  val pw = new PrintWriter(new File(s"/tmp/random-file-${randomFileName}.txt"))
  while (iter.hasNext) {
      pw.write(iter.next())
  }
  pw.close()
}
```
### glom
Takes every partition in your dataset and converts them to arrays
```scala
spark.sparkContext.parallelize(Seq("Hello", "World"), 2).glom().collect()
// Array(Array(Hello), Array(World))
```

```python
spark.sparkContext.parallelize(["Hello", "World"], 2).glom().collect()
# [['Hello'], ['World']]
```

## Advanced RDDs
Prev:
```scala
val myCollection = "Spark The Definitive Guide : Big Data Processing Made Simple"
  .split(" ")
val words = spark.sparkContext.parallelize(myCollection, 2)
```

```python
myCollection = "Spark The Definitive Guide : Big Data Processing Made Simple"\
  .split(" ")
words = spark.sparkContext.parallelize(myCollection, 2)
```

### Key-Value Basics (Key-Value RDDs)
simplest way to generate key-value:

```scala
words.map(word => (word.toLowerCase, 1))
```

```python
words.map(lambda word: (word.lower(), 1))
```

#### keyBy
Create key by a function

```scala
val keyword = words.keyBy(word => word.toLowerCase.toSeq(0).toString)
```

```python
keyword = words.keyBy(lambda word: word.lower()[0])
```
#### Mapping over Values

```scala
keyword.mapValues(word => word.toUpperCase).collect()
keyword.flatMapValues(word => word.toUpperCase).collect()
```

```python
keyword.mapValues(lambda word: word.upper()).collect()
keyword.flatMapValues(lambda word: word.upper()).collect()
```

#### Extracting Keys and Values

```scala
keyword.keys.collect()
keyword.values.collect()
```

```python
keyword.keys().collect()
keyword.values().collect()
```

```scala

```

```python

```

```scala

```

```python

```

```scala

```

```python

```

```scala

```

```python

```

```scala

```

```python

```

```scala

```

```python

```

```scala

```

```python

```

```scala

```

```python

```
