# AstBuilder &mdash; ANTLR-based SQL Parser

`AstBuilder` converts ANTLR `ParseTree`s into Catalyst entities using [visit callbacks](#visit-callbacks).

`AstBuilder` is the only requirement of the [AbstractSqlParser](AbstractSqlParser.md#astBuilder) abstraction (and used by [CatalystSqlParser](CatalystSqlParser.md) directly while [SparkSqlParser](SparkSqlParser.md) uses [SparkSqlAstBuilder](SparkSqlAstBuilder.md) instead).

??? note "AstBuilder and ANTLR"
    `AstBuilder` is a ANTLR `AbstractParseTreeVisitor` (as `SqlBaseBaseVisitor`) that is generated from the ANTLR grammar of Spark SQL.

    `SqlBaseBaseVisitor` is a ANTLR-specific base class that is generated at build time from a ANTLR grammar in [SqlBase.g4](https://github.com/apache/spark/blob/v3.0.0/sql/catalyst/src/main/antlr4/org/apache/spark/sql/catalyst/parser/SqlBase.g4).

    `SqlBaseBaseVisitor` is an [AbstractParseTreeVisitor](http://www.antlr.org/api/Java/org/antlr/v4/runtime/tree/AbstractParseTreeVisitor.html) in ANTLR.

## Visit Callbacks

### visitExists

Creates an [Exists](../spark-sql-Expression-Exists.md) expression

ANTLR labeled alternative: `#exists`

### visitExplain

Creates a [ExplainCommand](../logical-operators/ExplainCommand.md)

ANTLR rule: `explain`

### visitFirst

Creates a [First](../spark-sql-Expression-First.md) aggregate function expression

```
FIRST '(' expression (IGNORE NULLS)? ')'
```

ANTLR labeled alternative: `#first`

### visitFromClause

Creates a [LogicalPlan](../logical-operators/LogicalPlan.md)

Supports multiple comma-separated relations (that all together build a condition-less INNER JOIN) with optional [LATERAL VIEW](../spark-sql-Expression-Generator.md#lateral-view).

A relation can be one of the following or a combination thereof:

* Table identifier
* Inline table using `VALUES exprs AS tableIdent`
* Table-valued function (currently only `range` is supported)

ANTLR rule: `fromClause`

### visitFunctionCall

Creates one of the following:

* [UnresolvedFunction](../spark-sql-Expression-UnresolvedFunction.md) for a bare function (with no window specification)

* [UnresolvedWindowExpression](../spark-sql-Expression-UnresolvedWindowExpression.md) for a function evaluated in a windowed context with a `WindowSpecReference`

* [WindowExpression](../spark-sql-Expression-WindowExpression.md) for a function over a window

ANTLR rule: `functionCall`

```
import spark.sessionState.sqlParser

scala> sqlParser.parseExpression("foo()")
res0: org.apache.spark.sql.catalyst.expressions.Expression = 'foo()

scala> sqlParser.parseExpression("foo() OVER windowSpecRef")
res1: org.apache.spark.sql.catalyst.expressions.Expression = unresolvedwindowexpression('foo(), WindowSpecReference(windowSpecRef))

scala> sqlParser.parseExpression("foo() OVER (CLUSTER BY field)")
res2: org.apache.spark.sql.catalyst.expressions.Expression = 'foo() windowspecdefinition('field, UnspecifiedFrame)
```

### visitInlineTable

Creates a [UnresolvedInlineTable](../logical-operators/UnresolvedInlineTable.md) unary logical operator (as the child of [SubqueryAlias](../logical-operators/SubqueryAlias.md) for `tableAlias`)

```
VALUES expression (',' expression)* tableAlias
```

`expression` can be as follows:

* [CreateNamedStruct](../spark-sql-Expression-CreateNamedStruct.md) expression for multiple-column tables

* Any [Catalyst expression](../spark-sql-Expression.md) for one-column tables

`tableAlias` can be specified explicitly or defaults to `colN` for every column (starting from `1` for `N`).

ANTLR rule: `inlineTable`

### visitInsertIntoTable

Creates a [InsertIntoTable](../logical-operators/InsertIntoTable.md) (indirectly)

A 3-element tuple with a `TableIdentifier`, optional partition keys and the `exists` flag disabled

```
INSERT INTO TABLE? tableIdentifier partitionSpec?
```

ANTLR labeled alternative: `#insertIntoTable`

!!! note
    `insertIntoTable` is part of `insertInto` that is in turn used only as a helper labeled alternative in [singleInsertQuery](#singleInsertQuery) and [multiInsertQueryBody](#multiInsertQueryBody) ANTLR rules.

### visitInsertOverwriteTable

Creates a [InsertIntoTable](../logical-operators/InsertIntoTable.md) (indirectly)

A 3-element tuple with a `TableIdentifier`, optional partition keys and the `exists` flag

```
INSERT OVERWRITE TABLE tableIdentifier (partitionSpec (IF NOT EXISTS)?)?
```

In a way, `visitInsertOverwriteTable` is simply a more general version of the [visitInsertIntoTable](#visitInsertIntoTable) with the `exists` flag on or off based on existence of `IF NOT EXISTS`. The main difference is that [dynamic partitions](../spark-sql-dynamic-partition-inserts.md#dynamic-partitions) are used with no `IF NOT EXISTS`.

ANTLR labeled alternative: `#insertOverwriteTable`

!!! note
    `insertIntoTable` is part of `insertInto` that is in turn used only as a helper labeled alternative in [singleInsertQuery](#singleInsertQuery) and [multiInsertQueryBody](#multiInsertQueryBody) ANTLR rules.

### visitMultiInsertQuery

Creates a logical operator with a [InsertIntoTable](../logical-operators/InsertIntoTable.md) (and [UnresolvedRelation](../logical-operators/UnresolvedRelation.md) leaf operator)

```
FROM relation (',' relation)* lateralView*
INSERT OVERWRITE TABLE ...

FROM relation (',' relation)* lateralView*
INSERT INTO TABLE? ...
```

ANTLR rule: `multiInsertQueryBody`

### visitNamedExpression

Creates one of the following Catalyst expressions:

* [Alias](../spark-sql-Expression-Alias.md) (for a single alias)
* `MultiAlias` (for a parenthesis enclosed alias list)
* a bare [Expression](../spark-sql-Expression.md)

ANTLR rule: `namedExpression`

### visitNamedQuery

Creates a [SubqueryAlias](../logical-operators/SubqueryAlias.md)

### visitQuerySpecification

Creates [OneRowRelation](../logical-operators/OneRowRelation.md) or [LogicalPlan](../logical-operators/LogicalPlan.md)

??? note "OneRowRelation"
    `visitQuerySpecification` creates a `OneRowRelation` for a `SELECT` without a `FROM` clause.

    ```
    val q = sql("select 1")
    scala> println(q.queryExecution.logical.numberedTreeString)
    00 'Project [unresolvedalias(1, None)]
    01 +- OneRowRelation$
    ```

ANTLR rule: `querySpecification`

### visitPredicated

Creates an [Expression](../spark-sql-Expression.md)

ANTLR rule: `predicated`

### visitRelation

Creates a [LogicalPlan](../logical-operators/LogicalPlan.md) for a `FROM` clause.

ANTLR rule: `relation`

### visitSingleDataType

Creates a [DataType](../spark-sql-DataType.md)

ANTLR rule: `singleDataType`

### visitSingleExpression

Creates an [Expression](../spark-sql-Expression.md)

Takes the named expression and relays to [visitNamedExpression](#visitNamedExpression)

ANTLR rule: `singleExpression`

### visitSingleInsertQuery

Creates a [LogicalPlan](../logical-operators/LogicalPlan.md) with a [InsertIntoTable](../logical-operators/InsertIntoTable.md)

```
INSERT INTO TABLE? tableIdentifier partitionSpec? #insertIntoTable

INSERT OVERWRITE TABLE tableIdentifier (partitionSpec (IF NOT EXISTS)?)? #insertOverwriteTable
```

ANTLR labeled alternative: `#singleInsertQuery`

### visitSortItem

Creates a [SortOrder](../spark-sql-Expression-SortOrder.md) unevaluable unary expression

```
sortItem
    : expression ordering=(ASC | DESC)? (NULLS nullOrder=(LAST | FIRST))?
    ;

// queryOrganization
ORDER BY order+=sortItem (',' order+=sortItem)*
SORT BY sort+=sortItem (',' sort+=sortItem)*

// windowSpec
(ORDER | SORT) BY sortItem (',' sortItem)*)?
```

ANTLR rule: `sortItem`

### visitSingleStatement

Creates a [LogicalPlan](../logical-operators/LogicalPlan.md) from a single SQL statement

ANTLR rule: `singleStatement`

### visitStar

Creates a [UnresolvedStar](../spark-sql-Expression-UnresolvedStar.md)

ANTLR labeled alternative: `#star`

### visitSubqueryExpression

Creates a [ScalarSubquery](../spark-sql-Expression-SubqueryExpression-ScalarSubquery.md)

ANTLR labeled alternative: `#subqueryExpression`

### visitWindowDef

Creates a [WindowSpecDefinition](../spark-sql-Expression-WindowSpecDefinition.md)

```
// CLUSTER BY with window frame
'(' CLUSTER BY partition+=expression (',' partition+=expression)*) windowFrame? ')'

// PARTITION BY and ORDER BY with window frame
'(' ((PARTITION | DISTRIBUTE) BY partition+=expression (',' partition+=expression)*)?
  ((ORDER | SORT) BY sortItem (',' sortItem)*)?)
  windowFrame? ')'
```

ANTLR rule: `windowDef`

## Parsing Handlers

### withAggregation

Adds one of the following logical operators:

* [GroupingSets](../logical-operators/GroupingSets.md) for `GROUP BY &hellip; GROUPING SETS (&hellip;)`

* [Aggregate](../logical-operators/Aggregate.md) for `GROUP BY &hellip; (WITH CUBE | WITH ROLLUP)?`

### withGenerate

Adds a [Generate](../logical-operators/Generate.md) with a [UnresolvedGenerator](../spark-sql-Expression-UnresolvedGenerator.md) and [join](../logical-operators/Generate.md#join) flag enabled for `LATERAL VIEW` (in `SELECT` or `FROM` clauses).

### withHints

Adds a [Hint](../logical-operators/Hint.md) for `/*+ hint */` in `SELECT` queries.

!!! note
    Note `+` (plus) between `/*` and `*/`


`hint` is of the format `name` or `name (param1, param2, ...)`.

```
/*+ BROADCAST (table) */
```

### withInsertInto

Adds one of the following logical operators:

* [InsertIntoTable](../logical-operators/InsertIntoTable.md) for [visitSingleInsertQuery](#visitSingleInsertQuery) or [visitMultiInsertQuery](#visitMultiInsertQuery)

* [InsertIntoDir](../logical-operators/InsertIntoDir.md) for...FIXME

### withJoinRelations

Adds a [Join](../logical-operators/Join.md) for a [FROM clause](#visitFromClause) and [relation](#visitRelation) alone.

The following join types are supported:

* `INNER` (default)
* `CROSS`
* `LEFT` (with optional `OUTER`)
* `LEFT SEMI`
* `RIGHT` (with optional `OUTER`)
* `FULL` (with optional `OUTER`)
* `ANTI` (optionally prefixed with `LEFT`)

The following join criteria are supported:

* `ON booleanExpression`
* `USING '(' identifier (',' identifier)* ')'`

Joins can be `NATURAL` (with no join criteria)

### withQuerySpecification

Adds a query specification to a logical operator

For transform `SELECT` (with `TRANSFORM`, `MAP` or `REDUCE` qualifiers), `withQuerySpecification` does...FIXME

For regular `SELECT` (no `TRANSFORM`, `MAP` or `REDUCE` qualifiers), `withQuerySpecification` adds (in that order):

. [Generate](#withGenerate) unary logical operators (if used in the parsed SQL text)

. [Filter](../logical-operators/Filter.md) unary logical plan (if used in the parsed SQL text)

. [GroupingSets or Aggregate](#withAggregation) unary logical operators (if used in the parsed SQL text)

. `Project` and/or `Filter` unary logical operators

. [WithWindowDefinition](#withWindows) unary logical operator (if used in the parsed SQL text)

. [UnresolvedHint](#withHints) unary logical operator (if used in the parsed SQL text)

### withPredicate

* `NOT? IN '(' query ')'` adds an [In](../spark-sql-Expression-In.md) predicate expression with a [ListQuery](../spark-sql-Expression-ListQuery.md) subquery expression

* `NOT? IN '(' expression (',' expression)* ')'` adds an [In](../spark-sql-Expression-In.md) predicate expression

### withWindows

Adds a [WithWindowDefinition](../logical-operators/WithWindowDefinition.md) for [window aggregates](../spark-sql-functions-windows.md) (given `WINDOW` definitions).

Used for [withQueryResultClauses](#withQueryResultClauses) and [withQuerySpecification](#withQuerySpecification) with `windows` definition.

```
WINDOW identifier AS windowSpec
  (',' identifier AS windowSpec)*
```

## `aliasPlan` Method

```scala
aliasPlan(
  alias: ParserRuleContext,
  plan: LogicalPlan): LogicalPlan
```

`aliasPlan`...FIXME

`aliasPlan` is used when...FIXME

## `mayApplyAliasPlan` Method

```scala
mayApplyAliasPlan(
  tableAlias: TableAliasContext,
  plan: LogicalPlan): LogicalPlan
```

`mayApplyAliasPlan`...FIXME

`mayApplyAliasPlan` is used when...FIXME
