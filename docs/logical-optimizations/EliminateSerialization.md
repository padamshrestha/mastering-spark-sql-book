# EliminateSerialization Logical Optimization

`EliminateSerialization` is a <<spark-sql-Optimizer.adoc#batches, base logical optimization>> that <<apply, optimizes>> logical plans with link:spark-sql-LogicalPlan-DeserializeToObject.adoc[DeserializeToObject] (after `SerializeFromObject` or `TypedFilter`), `AppendColumns` (after `SerializeFromObject`), `TypedFilter` (after `SerializeFromObject`) logical operators.

`EliminateSerialization` is part of the <<spark-sql-Optimizer.adoc#Operator_Optimization_before_Inferring_Filters, Operator Optimization before Inferring Filters>> fixed-point batch in the standard batches of the <<spark-sql-Optimizer.adoc#, Catalyst Optimizer>>.

`EliminateSerialization` is simply a <<spark-sql-catalyst-Rule.adoc#, Catalyst rule>> for transforming <<spark-sql-LogicalPlan.adoc#, logical plans>>, i.e. `Rule[LogicalPlan]`.

Examples include:

1. <<example-map-filter, `map` followed by `filter` Logical Plan>>
2. <<example-map-map, `map` followed by another `map` Logical Plan>>
3. <<example-groupByKey-agg, `groupByKey` followed by `agg` Logical Plan>>

=== [[example-map-filter]] Example -- `map` followed by `filter` Logical Plan

```
scala> spark.range(4).map(n => n * 2).filter(n => n < 3).explain(extended = true)
== Parsed Logical Plan ==
'TypedFilter <function1>, long, [StructField(value,LongType,false)], unresolveddeserializer(upcast(getcolumnbyordinal(0, LongType), LongType, - root class: "scala.Long"))
+- SerializeFromObject [input[0, bigint, true] AS value#185L]
   +- MapElements <function1>, class java.lang.Long, [StructField(value,LongType,true)], obj#184: bigint
      +- DeserializeToObject newInstance(class java.lang.Long), obj#183: java.lang.Long
         +- Range (0, 4, step=1, splits=Some(8))

== Analyzed Logical Plan ==
value: bigint
TypedFilter <function1>, long, [StructField(value,LongType,false)], cast(value#185L as bigint)
+- SerializeFromObject [input[0, bigint, true] AS value#185L]
   +- MapElements <function1>, class java.lang.Long, [StructField(value,LongType,true)], obj#184: bigint
      +- DeserializeToObject newInstance(class java.lang.Long), obj#183: java.lang.Long
         +- Range (0, 4, step=1, splits=Some(8))

== Optimized Logical Plan ==
SerializeFromObject [input[0, bigint, true] AS value#185L]
+- Filter <function1>.apply
   +- MapElements <function1>, class java.lang.Long, [StructField(value,LongType,true)], obj#184: bigint
      +- DeserializeToObject newInstance(class java.lang.Long), obj#183: java.lang.Long
         +- Range (0, 4, step=1, splits=Some(8))

== Physical Plan ==
*SerializeFromObject [input[0, bigint, true] AS value#185L]
+- *Filter <function1>.apply
   +- *MapElements <function1>, obj#184: bigint
      +- *DeserializeToObject newInstance(class java.lang.Long), obj#183: java.lang.Long
         +- *Range (0, 4, step=1, splits=Some(8))
```

=== [[example-map-map]] Example -- `map` followed by another `map` Logical Plan

```
// Notice unnecessary mapping between String and Int types
val query = spark.range(3).map(_.toString).map(_.toInt)

scala> query.explain(extended = true)
...
TRACE SparkOptimizer:
=== Applying Rule org.apache.spark.sql.catalyst.optimizer.EliminateSerialization ===
 SerializeFromObject [input[0, int, true] AS value#91]                                                                                                                     SerializeFromObject [input[0, int, true] AS value#91]
 +- MapElements <function1>, class java.lang.String, [StructField(value,StringType,true)], obj#90: int                                                                     +- MapElements <function1>, class java.lang.String, [StructField(value,StringType,true)], obj#90: int
!   +- DeserializeToObject value#86.toString, obj#89: java.lang.String                                                                                                        +- Project [obj#85 AS obj#89]
!      +- SerializeFromObject [staticinvoke(class org.apache.spark.unsafe.types.UTF8String, StringType, fromString, input[0, java.lang.String, true], true) AS value#86]         +- MapElements <function1>, class java.lang.Long, [StructField(value,LongType,true)], obj#85: java.lang.String
!         +- MapElements <function1>, class java.lang.Long, [StructField(value,LongType,true)], obj#85: java.lang.String                                                            +- DeserializeToObject newInstance(class java.lang.Long), obj#84: java.lang.Long
!            +- DeserializeToObject newInstance(class java.lang.Long), obj#84: java.lang.Long                                                                                          +- Range (0, 3, step=1, splits=Some(8))
!               +- Range (0, 3, step=1, splits=Some(8))
...
== Parsed Logical Plan ==
'SerializeFromObject [input[0, int, true] AS value#91]
+- 'MapElements <function1>, class java.lang.String, [StructField(value,StringType,true)], obj#90: int
   +- 'DeserializeToObject unresolveddeserializer(upcast(getcolumnbyordinal(0, StringType), StringType, - root class: "java.lang.String").toString), obj#89: java.lang.String
      +- SerializeFromObject [staticinvoke(class org.apache.spark.unsafe.types.UTF8String, StringType, fromString, input[0, java.lang.String, true], true) AS value#86]
         +- MapElements <function1>, class java.lang.Long, [StructField(value,LongType,true)], obj#85: java.lang.String
            +- DeserializeToObject newInstance(class java.lang.Long), obj#84: java.lang.Long
               +- Range (0, 3, step=1, splits=Some(8))

== Analyzed Logical Plan ==
value: int
SerializeFromObject [input[0, int, true] AS value#91]
+- MapElements <function1>, class java.lang.String, [StructField(value,StringType,true)], obj#90: int
   +- DeserializeToObject cast(value#86 as string).toString, obj#89: java.lang.String
      +- SerializeFromObject [staticinvoke(class org.apache.spark.unsafe.types.UTF8String, StringType, fromString, input[0, java.lang.String, true], true) AS value#86]
         +- MapElements <function1>, class java.lang.Long, [StructField(value,LongType,true)], obj#85: java.lang.String
            +- DeserializeToObject newInstance(class java.lang.Long), obj#84: java.lang.Long
               +- Range (0, 3, step=1, splits=Some(8))

== Optimized Logical Plan ==
SerializeFromObject [input[0, int, true] AS value#91]
+- MapElements <function1>, class java.lang.String, [StructField(value,StringType,true)], obj#90: int
   +- MapElements <function1>, class java.lang.Long, [StructField(value,LongType,true)], obj#85: java.lang.String
      +- DeserializeToObject newInstance(class java.lang.Long), obj#84: java.lang.Long
         +- Range (0, 3, step=1, splits=Some(8))

== Physical Plan ==
*SerializeFromObject [input[0, int, true] AS value#91]
+- *MapElements <function1>, obj#90: int
   +- *MapElements <function1>, obj#85: java.lang.String
      +- *DeserializeToObject newInstance(class java.lang.Long), obj#84: java.lang.Long
         +- *Range (0, 3, step=1, splits=Some(8))
```

=== [[example-groupByKey-agg]] Example -- `groupByKey` followed by `agg` Logical Plan

```
scala> spark.range(4).map(n => (n, n % 2)).groupByKey(_._2).agg(typed.sum(_._2)).explain(true)
== Parsed Logical Plan ==
'Aggregate [value#454L], [value#454L, unresolvedalias(typedsumdouble(org.apache.spark.sql.execution.aggregate.TypedSumDouble@4fcb0de4, Some(unresolveddeserializer(newInstance(class scala.Tuple2), _1#450L, _2#451L)), Some(class scala.Tuple2), Some(StructType(StructField(_1,LongType,true), StructField(_2,LongType,false))), input[0, double, true] AS value#457, unresolveddeserializer(upcast(getcolumnbyordinal(0, DoubleType), DoubleType, - root class: "scala.Double"), value#457), input[0, double, true] AS value#456, DoubleType, DoubleType, false), Some(<function1>))]
+- AppendColumns <function1>, class scala.Tuple2, [StructField(_1,LongType,true), StructField(_2,LongType,false)], newInstance(class scala.Tuple2), [input[0, bigint, true] AS value#454L]
   +- SerializeFromObject [assertnotnull(input[0, scala.Tuple2, true], top level non-flat input object)._1.longValue AS _1#450L, assertnotnull(input[0, scala.Tuple2, true], top level non-flat input object)._2 AS _2#451L]
      +- MapElements <function1>, class java.lang.Long, [StructField(value,LongType,true)], obj#449: scala.Tuple2
         +- DeserializeToObject newInstance(class java.lang.Long), obj#448: java.lang.Long
            +- Range (0, 4, step=1, splits=Some(8))

== Analyzed Logical Plan ==
value: bigint, TypedSumDouble(scala.Tuple2): double
Aggregate [value#454L], [value#454L, typedsumdouble(org.apache.spark.sql.execution.aggregate.TypedSumDouble@4fcb0de4, Some(newInstance(class scala.Tuple2)), Some(class scala.Tuple2), Some(StructType(StructField(_1,LongType,true), StructField(_2,LongType,false))), input[0, double, true] AS value#457, cast(value#457 as double), input[0, double, true] AS value#456, DoubleType, DoubleType, false) AS TypedSumDouble(scala.Tuple2)#462]
+- AppendColumns <function1>, class scala.Tuple2, [StructField(_1,LongType,true), StructField(_2,LongType,false)], newInstance(class scala.Tuple2), [input[0, bigint, true] AS value#454L]
   +- SerializeFromObject [assertnotnull(input[0, scala.Tuple2, true], top level non-flat input object)._1.longValue AS _1#450L, assertnotnull(input[0, scala.Tuple2, true], top level non-flat input object)._2 AS _2#451L]
      +- MapElements <function1>, class java.lang.Long, [StructField(value,LongType,true)], obj#449: scala.Tuple2
         +- DeserializeToObject newInstance(class java.lang.Long), obj#448: java.lang.Long
            +- Range (0, 4, step=1, splits=Some(8))

== Optimized Logical Plan ==
Aggregate [value#454L], [value#454L, typedsumdouble(org.apache.spark.sql.execution.aggregate.TypedSumDouble@4fcb0de4, Some(newInstance(class scala.Tuple2)), Some(class scala.Tuple2), Some(StructType(StructField(_1,LongType,true), StructField(_2,LongType,false))), input[0, double, true] AS value#457, value#457, input[0, double, true] AS value#456, DoubleType, DoubleType, false) AS TypedSumDouble(scala.Tuple2)#462]
+- AppendColumnsWithObject <function1>, [assertnotnull(input[0, scala.Tuple2, true], top level non-flat input object)._1.longValue AS _1#450L, assertnotnull(input[0, scala.Tuple2, true], top level non-flat input object)._2 AS _2#451L], [input[0, bigint, true] AS value#454L]
   +- MapElements <function1>, class java.lang.Long, [StructField(value,LongType,true)], obj#449: scala.Tuple2
      +- DeserializeToObject newInstance(class java.lang.Long), obj#448: java.lang.Long
         +- Range (0, 4, step=1, splits=Some(8))

== Physical Plan ==
*HashAggregate(keys=[value#454L], functions=[typedsumdouble(org.apache.spark.sql.execution.aggregate.TypedSumDouble@4fcb0de4, Some(newInstance(class scala.Tuple2)), Some(class scala.Tuple2), Some(StructType(StructField(_1,LongType,true), StructField(_2,LongType,false))), input[0, double, true] AS value#457, value#457, input[0, double, true] AS value#456, DoubleType, DoubleType, false)], output=[value#454L, TypedSumDouble(scala.Tuple2)#462])
+- Exchange hashpartitioning(value#454L, 200)
   +- *HashAggregate(keys=[value#454L], functions=[partial_typedsumdouble(org.apache.spark.sql.execution.aggregate.TypedSumDouble@4fcb0de4, Some(newInstance(class scala.Tuple2)), Some(class scala.Tuple2), Some(StructType(StructField(_1,LongType,true), StructField(_2,LongType,false))), input[0, double, true] AS value#457, value#457, input[0, double, true] AS value#456, DoubleType, DoubleType, false)], output=[value#454L, value#463])
      +- AppendColumnsWithObject <function1>, [assertnotnull(input[0, scala.Tuple2, true], top level non-flat input object)._1.longValue AS _1#450L, assertnotnull(input[0, scala.Tuple2, true], top level non-flat input object)._2 AS _2#451L], [input[0, bigint, true] AS value#454L]
         +- MapElements <function1>, obj#449: scala.Tuple2
            +- DeserializeToObject newInstance(class java.lang.Long), obj#448: java.lang.Long
               +- *Range (0, 4, step=1, splits=Some(8))
```

=== [[apply]] Executing Rule -- `apply` Method

[source, scala]
----
apply(plan: LogicalPlan): LogicalPlan
----

NOTE: `apply` is part of the <<spark-sql-catalyst-Rule.adoc#apply, Rule Contract>> to execute (apply) a rule on a <<spark-sql-catalyst-TreeNode.adoc#, TreeNode>> (e.g. <<spark-sql-LogicalPlan.adoc#, LogicalPlan>>).

`apply`...FIXME
