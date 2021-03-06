# User-Defined Functions

*User-Defined Functions* (aka *UDF*) is a feature of Spark SQL to define new link:spark-sql-Column.adoc[Column]-based functions that extend the vocabulary of Spark SQL's DSL for transforming link:spark-sql-Dataset.adoc[Datasets].

[IMPORTANT]
====
Use the link:spark-sql-functions.adoc[higher-level standard Column-based functions] (with link:spark-sql-Dataset.adoc[Dataset operators]) whenever possible before reverting to developing user-defined functions since link:spark-sql-udfs-blackbox.adoc[UDFs are a blackbox] for Spark SQL and it cannot (and does not even try to) optimize them.

As Reynold Xin from the Apache Spark project has once said on Spark's dev mailing list:

> There are simple cases in which we can analyze the UDFs byte code and infer what it is doing, but it is pretty difficult to do in general.

Check out link:spark-sql-udfs-blackbox.adoc[UDFs are Blackbox -- Don't Use Them Unless You've Got No Choice] if you want to know the internals.
====

You define a new UDF by defining a Scala function as an input parameter of <<udf-function, `udf` function>>. It accepts Scala functions of up to 10 input parameters.

```
val dataset = Seq((0, "hello"), (1, "world")).toDF("id", "text")

// Define a regular Scala function
val upper: String => String = _.toUpperCase

// Define a UDF that wraps the upper Scala function defined above
// You could also define the function in place, i.e. inside udf
// but separating Scala functions from Spark SQL's UDFs allows for easier testing
import org.apache.spark.sql.functions.udf
val upperUDF = udf(upper)

// Apply the UDF to change the source dataset
scala> dataset.withColumn("upper", upperUDF('text)).show
+---+-----+-----+
| id| text|upper|
+---+-----+-----+
|  0|hello|HELLO|
|  1|world|WORLD|
+---+-----+-----+
```

You can register UDFs to use in link:spark-sql-SparkSession.adoc#sql[SQL-based query expressions] via link:spark-sql-UDFRegistration.adoc[UDFRegistration] (that is available through link:spark-sql-SparkSession.adoc#udf[`SparkSession.udf` attribute]).

[source, scala]
----
val spark: SparkSession = ...
scala> spark.udf.register("myUpper", (input: String) => input.toUpperCase)
----

You can query for available link:spark-sql-functions.adoc[standard] and user-defined functions using the link:spark-sql-Catalog.adoc[Catalog] interface (that is available through link:spark-sql-SparkSession.adoc#catalog[`SparkSession.catalog` attribute]).

[source, scala]
----
val spark: SparkSession = ...
scala> spark.catalog.listFunctions.filter('name like "%upper%").show(false)
+-------+--------+-----------+-----------------------------------------------+-----------+
|name   |database|description|className                                      |isTemporary|
+-------+--------+-----------+-----------------------------------------------+-----------+
|myupper|null    |null       |null                                           |true       |
|upper  |null    |null       |org.apache.spark.sql.catalyst.expressions.Upper|true       |
+-------+--------+-----------+-----------------------------------------------+-----------+
----

NOTE: UDFs play a vital role in Spark MLlib to define new link:spark-mllib/spark-mllib-transformers.adoc[Transformers] that are function objects that transform `DataFrames` into `DataFrames` by introducing new columns.

=== [[udf-function]] udf Functions (in functions object)

[source, scala]
----
udf[RT: TypeTag](f: Function0[RT]): UserDefinedFunction
...
udf[RT: TypeTag, A1: TypeTag, A2: TypeTag, A3: TypeTag, A4: TypeTag, A5: TypeTag, A6: TypeTag, A7: TypeTag, A8: TypeTag, A9: TypeTag, A10: TypeTag](f: Function10[A1, A2, A3, A4, A5, A6, A7, A8, A9, A10, RT]): UserDefinedFunction
----

`org.apache.spark.sql.functions` object comes with `udf` function to let you define a UDF for a Scala function `f`.

```
val df = Seq(
  (0, "hello"),
  (1, "world")).toDF("id", "text")

// Define a "regular" Scala function
// It's a clone of upper UDF
val toUpper: String => String = _.toUpperCase

import org.apache.spark.sql.functions.udf
val upper = udf(toUpper)

scala> df.withColumn("upper", upper('text)).show
+---+-----+-----+
| id| text|upper|
+---+-----+-----+
|  0|hello|HELLO|
|  1|world|WORLD|
+---+-----+-----+

// You could have also defined the UDF this way
val upperUDF = udf { s: String => s.toUpperCase }

// or even this way
val upperUDF = udf[String, String](_.toUpperCase)

scala> df.withColumn("upper", upperUDF('text)).show
+---+-----+-----+
| id| text|upper|
+---+-----+-----+
|  0|hello|HELLO|
|  1|world|WORLD|
+---+-----+-----+
```

TIP: Define custom UDFs based on "standalone" Scala functions (e.g. `toUpperUDF`) so you can test the Scala functions using Scala way (without Spark SQL's "noise") and once they are defined reuse the UDFs in link:spark-mllib/spark-mllib-transformers.adoc#UnaryTransformer[UnaryTransformers].
