# Spark Style Guide

Spark is an amazingly powerful big data engine that's written in Scala.

This document draws on the Spark source code, the [Spark examples](http://spark.apache.org/examples.html), and popular open source Spark libraries like [spark-testing-base](https://github.com/holdenk/spark-testing-base).

Comprehensive Scala style guides already exist.  This document focuses specifically on the style issues for Spark programmers.

> Any style guide written in English is either so brief that it’s ambiguous, or so long that no one reads it.
> - [Bob Nystrom](http://journal.stuffwithstuff.com/2015/09/08/the-hardest-program-ive-ever-written/)

## <a name='TOC'>Table of Contents</a>

  1. [Scala Style Guides](#scala-style-guides)
  1. [Variables](#variables)
  1. [Columns](#columns)
  1. [Chained Method Calls](#chained-method-calls)
  1. [Spark SQL](#spark-sql)
  2. [Open Source](#open-source)
  2. [User Defined Functions](#user-defined-functions)
  3. [Custom Transformations](#custom-transformations)
  4. [null](#null)
  4. [JAR Files](#jar-files)
  4. [Documentation](#documentation)
  5. [Testing](#testing)

## <a name='scala-style-guides'>Scala Style Guides</a>

There is [an official Scala style guide](http://docs.scala-lang.org/style/) and a [Databricks Scala style guide](https://github.com/databricks/scala-style-guide).  The founder of Databricks is one of the Spark creators and several Databricks engineers are Spark core contributors, so you should follow the [Databricks scala-style-guide](https://github.com/databricks/scala-style-guide).

You can create Spark and [haters still gonna hate](https://www.reddit.com/r/scala/comments/2ze443/a_good_example_of_a_scala_style_guide_by_people/)!

### Automated Code Formatting Tools

[Scalafmt](http://scalameta.org/scalafmt/) and [scalariform](https://github.com/scala-ide/scalariform) are automated code formatting tools.  scalariform's default settings format code similar to the Databricks scala-style-guide and is a good place to start.  The [sbt-scalariform](https://github.com/sbt/sbt-scalariform) plugin automatically reformats code upon compile and is the best way to keep code formatted consistely without thinking.  Here are some scalariform settings that work well with Spark code.

```scala
SbtScalariform.scalariformSettings

ScalariformKeys.preferences := ScalariformKeys.preferences.value
  .setPreference(DoubleIndentConstructorArguments, true)
  .setPreference(SpacesAroundMultiImports, false)
  .setPreference(DanglingCloseParenthesis, Force)
```

## <a name='variables'>Variables</a>

Variables should use camelCase.  Variables that point to DataFrames, Datasets, and RDDs should be suffixed accordingly to make your code readable:

* Variables pointing to DataFrames should be suffixed with `DF` (following conventions in the [Spark Programming Guide](http://spark.apache.org/docs/latest/sql-programming-guide.html))


```scala
peopleDF.createOrReplaceTempView("people")
```

* Variables pointing to Datasets should be suffixed with `DS`


```scala
val stringsDS = sqlDF.map {
  case Row(key: Int, value: String) => s"Key: $key, Value: $value"
}
```

* Variables pointing to RDDs should be suffixed with `RDD`

```scala
val peopleRDD = spark.sparkContext.textFile("examples/src/main/resources/people.txt")
```

Use the variable `col` for `Column` arguments.

```scala
def min(col: Column)
```

Use `col1` and `col2` for methods that take two `Column` arguments.

```scala
def corr(col1: Column, col2: Column)
```

Use `cols` for methods that take an arbitrary number of `Column` arguments.

```scala
def array(cols: Column*)
```

For methods that take column name `String` arguments, follow the same pattern and use `colName`, `colName1`, `colName2`, and `colNames` as variables.

Collections of things should use plural variable names.

```scala
var animals = List("dog", "cat", "goose")

// DONT DO THIS
var animalList = List("dog", "cat", "goose")
```

Singluar nouns should be used for single objects.

```scala
val myCarColor = "red"
```

## <a name='chained-method-calls'>Chained Method Calls</a>

Spark methods are often deeply chained and should be broken up on multiple lines.

```scala
jdbcDF.write
  .format("jdbc")
  .option("url", "jdbc:postgresql:dbserver")
  .option("dbtable", "schema.tablename")
  .option("user", "username")
  .option("password", "password")
  .save()
```

Here's an example of a well formatted extract:

```scala
val extractDF = spark.read.parquet("someS3Path")
  .select(
    "name",
    "Date of Birth"
  )
  .transform(someCustomTransformation())
  .withColumnRenamed("Date of Birth", "date_of_birth")
  .filter(
    col("date_of_birth") > "1999-01-02"
  )
```

## <a name='spark-sql'>Spark SQL</a>

Use multiline strings to write properly indented SQL code:

```scala
val coolDF = spark.sql("""
select
  `first_name`,
  `last_name`,
  `hair_color`
from people
""")
```

## <a name='columns'>Columns</a>

Columns have name, type, nullable, and metadata properties.

Columns that contain boolean values should use predicate names like `is_nice_person` or `has_red_hair`.  Use `snake_case` for column names, so it's easier to write SQL code.

You can write `(col("is_summer") && col("is_europe"))` instead of `(col("is_summer") === true && col("is_europe") === true)`.  The predicate column names make the concise syntax nice and readable.

Columns should be typed properly.  Don't overuse `StringType` in your schema.

Columns should only be nullable if `null` values are allowed.  Code written for nullable columns should always address `null` values gracefully.

Use acronyms when needed to keep column names short.  Define any acronyms used at the top of the data file, so other programmers can follow along.

Use the following shorthand notation for columns that perform  comparisons.

* `gt`: greater than
* `lt`: less than
* `leq`: less than or equal to
* `geq`: greater than or equal to
* `eq`: equal to
* `between`

Here are some example column names:

* `player_age_gt_20`
* `player_age_gt_15_leq_30`
* `player_age_between_13_19`
* `player_age_eq_45`

### Immutable Columns

Custom transformations shouldn't overwrite an existing field in a schema during a transformation.  Add a new column to a DataFrame instead of mutating the data in an existing column.

Suppose you have a DataFrame with `name` and `nickname` columns and would like a column that coalesces the `name` and `nickname` columns.

```
+-----+--------+
| name|nickname|
+-----+--------+
|  joe|    null|
| null|   crazy|
|frank|    bull|
+-----+--------+
```

Don't overwrite the `name` field and create a DataFrame like this:

```
+-----+--------+
| name|nickname|
+-----+--------+
|  joe|    null|
|crazy|   crazy|
|frank|    bull|
+-----+--------+
```

Create a new column, so existing columns aren't changed and column immutability is preserved.

```
+-----+--------+---------+
| name|nickname|name_meow|
+-----+--------+---------+
|  joe|    null|      joe|
| null|   crazy|    crazy|
|frank|    bull|    frank|
+-----+--------+---------+
```

## <a name='open-source'>Open Source</a>

You should write generic open source code whenever possible.  Open source code is easily reusable (especially when it's uploaded to Spark Packages / Maven Repository) and forces you to design code without business logic.

The [`org.apache.spark.sql.functions`](https://spark.apache.org/docs/2.1.0/api/java/org/apache/spark/sql/functions.html) class provides some great examples of open source functions.

The [`Dataset`](https://spark.apache.org/docs/2.1.0/api/java/org/apache/spark/sql/Dataset.html) and [`Column`](https://spark.apache.org/docs/2.1.0/api/java/org/apache/spark/sql/Column.html) classes provide great examples of code that facilitates DataFrame transformations.

## <a name='user-defined-functions'>User Defined Functions</a>

*Coming soon...*

## <a name='custom-transformations'>Custom transformations</a>

Use multiple parameter lists when defining custom transformations, so you can chain your custom transformations with the `Dataset#transform` method.  You should disregard this advice from the Databricks Scala style guide: "Avoid using multiple parameter lists. They complicate operator overloading, and can confuse programmers less familiar with Scala."

You need to use multiple parameter lists to write awesome code like this:

```scala
def withCat(name: String)(df: DataFrame): DataFrame = {
  df.withColumn("cats", lit(s"$name meow"))
}
```

The `withCat()` custom transformation can be used as follows:

```scala
val niceDF = df.transform(withCat("puffy"))
```

### Naming conventions

* `with` precedes transformations that add columns:

  - `withCoolCat()` adds the column `cool_cat` to a DataFrame

  - `withIsNicePerson` adds the column `is_nice_person` to a DataFrame.

* `filter` precedes transformations that remove rows:

  - `filterNegativeGrowthPath()` removes the data rows where the `growth_path` column is negative

  - `filterBadData()` removes the bad data

* `enrich` precedes transformations that clobber columns.  DataFrame transformations should not be clobbered and `enrich` transformations should ideally never be used.

* `explode` precedes transformations that add rows to a DataFrame by "exploding" a row into multiple rows.

### Schema Dependent DataFrame Transformations

Schema dependent DataFrame transformations make assumptions about the underlying DataFrame schema.  Schema dependent DataFrame transformations should explicitly validate DataFrame dependencies to make the code and error messages more readable.

The following `withFullName()` DataFrame transformation assumes that the underlying DataFrame has `first_name` and `last_name` columns.

```scala
def withFullName()(df: DataFrame): DataFrame = {
  df.withColumn(
    "full_name",
    concat_ws(" ", col("first_name"), col("last_name"))
  )
}
```

You should use [spark-daria](https://github.com/mrpowers/spark-daria) to validate the schema requirements of a DataFrame transformation.

```scala
def withFullName()(df: DataFrame): DataFrame = {
  validatePresenceOfColumns(df, Seq("first_name", "last_name"))
  df.withColumn(
    "full_name",
    concat_ws(" ", col("first_name"), col("last_name"))
  )
}
```

See [this blog post](https://medium.com/@mrpowers/validating-spark-dataframe-schemas-28d2b3c69d2a) for a detailed description on validating DataFrame dependencies.

### Schema Independent DataFrame Transformations

Schema independent DataFrame transformations do not depend on the underlying DataFrame schema, as discussed in [this blog post](https://medium.com/@mrpowers/schema-independent-dataframe-transformations-d6b36e12dca6).

```scala
def withAgePlusOne(
  ageColName: String,
  resultColName: String
)(df: DataFrame): DataFrame = {
  df.withColumn(resultColName, col(ageColName) + 1)
}
```

### What type of DataFrame transformation should be used

Schema dependent transformations should be used for functions that rely on a large number of columns or functions that are only expected to be run on a certain schema (e.g. a data lake with a schema that doensn't change).

Schema independent transformations should be run for functions that will be run on a variety of DataFrame schemas.

## <a name='null'>null</a>

`null` should be used in DataFrames for values that are [unknown, missing, or irrelevant](https://medium.com/@mrpowers/dealing-with-null-in-spark-cfdbb12f231e#.fk27ontik).

Spark core functions frequently return `null` and your code can also add `null` to DataFrames (by returning `None` or explicitly returning `null`).

In general, it's better to keep all `null` references out of code and use `Option[T]` instead.  `Option` is a bit slower and explicit `null` references may be required for performance sensitve code.  Start with `Option` and only use explicit `null` references if `Option` becomes a performance bottleneck.

The schema for a column should set nullable to `false` if the column should not take `null` values.

## <a name='jar-files'>JAR Files</a>

JAR files should be named like this:

```
spark-testing-base_2.11-2.1.0_0.6.0.jar
```

Generically:

```
spark-testing-base_scalaVersion-sparkVersion_projectVersion.jar
```

If you're using sbt assembly, you can use the following line of code to build a JAR file using the correct naming conventions.

```scala
assemblyJarName in assembly := s"${name.value}_${scalaBinaryVersion.value}-${sparkVersion.value}_${version.value}.jar"
```

If you're using `sbt package`, you can add this code to your `build.sbt` file to generate a JAR file that follows the naming conventions.

```scala
artifactName := { (sv: ScalaVersion, module: ModuleID, artifact: Artifact) =>
  artifact.name + "_" + sv.binary + "-" + sparkVersion.value + "_" + module.revision + "." + artifact.extension
}
```

## <a name='documentation'>Documentation</a>

The following documentation guidelines generally copy the documentation in the [Spark source code](https://github.com/apache/spark/blob/d5861aba9d80ca15ad3f22793b79822e470d6913/sql/core/src/main/scala/org/apache/spark/sql/functions.scala).  For example, here's how the `rpad` method is defined in the Spark source code.

```scala
/**
 * Right-pad the string column with pad to a length of len. If the string column is longer
 * than len, the return value is shortened to len characters.
 *
 * @group string_funcs
 * @since 1.5.0
 */
def rpad(str: Column, len: Int, pad: String): Column = withExpr {
  StringRPad(str.expr, lit(len).expr, lit(pad).expr)
}
```

Here's an example of the the `Column#equalTo()` method that contains an example code snippet.

```scala
/**
 * Equality test.
 * {{{
 *   // Scala:
 *   df.filter( df("colA") === df("colB") )
 *
 *   // Java
 *   import static org.apache.spark.sql.functions.*;
 *   df.filter( col("colA").equalTo(col("colB")) );
 * }}}
 *
 * @group expr_ops
 * @since 1.3.0
 */
def equalTo(other: Any): Column = this === other
```

The `@since` annotation should be used to document when features are added to the API.

The `@note` annotaion should be used to detail important information about a function, like the following example.

```scala
/**
 * Aggregate function: returns the level of grouping, equals to
 *
 * {{{
 *   (grouping(c1) <<; (n-1)) + (grouping(c2) <<; (n-2)) + ... + grouping(cn)
 * }}}
 *
 * @note The list of columns should match with grouping columns exactly, or empty (means all the
 * grouping columns).
 *
 * @group agg_funcs
 * @since 2.0.0
 */
def grouping_id(cols: Column*): Column = Column(GroupingID(cols.map(_.expr)))
```

### Column Functions

Column functions should be annotated with the following groups, consistent with the Spark functions that return Column objects.

```
@groupname udf_funcs UDF functions
@groupname agg_funcs Aggregate functions
@groupname datetime_funcs Date time functions
@groupname sort_funcs Sorting functions
@groupname normal_funcs Non-aggregate functions
@groupname math_funcs Math functions
@groupname misc_funcs Misc functions
@groupname window_funcs Window functions
@groupname string_funcs String functions
@groupname collection_funcs Collection functions
@groupname Ungrouped Support functions for DataFrames
 ```

Here's an example of a well-documented Column function in the [spark-daria](https://github.com/MrPowers/spark-daria) project.

```scala
/**
  * Removes all whitespace in a string
  *
  * @group string_funcs
  * @since 2.0.0
  */
def removeAllWhitespace(col: Column): Column = {
  regexp_replace(col, "\\s+", "")
}
```

### DataFrame Transformations

Custom transformations can add/remove rows and columns from a DataFrame.  DataFrame transformation documentation should specify how the custom transformation is modifying the DataFrame and list the name of columns added to the DataFrame as appropriate.

## <a name='testing'>Testing</a>

Use the [spark-fast-tests](https://github.com/MrPowers/spark-fast-tests) library for writing DataFrame / Dataset / RDD tests with Spark.  [spark-testing-base](https://github.com/holdenk/spark-testing-base/) should be used for streaming tests.

Read [this blog post for a gentle introduction to testing Spark code](https://medium.com/@mrpowers/testing-spark-applications-8c590d3215fa), [this blog post on how to design easily testable Spark code](https://medium.com/@mrpowers/designing-easily-testable-spark-code-df0755ef00a4), and [this blog post on how to cut the run time of a Spark test suite](https://medium.com/@mrpowers/how-to-cut-the-run-time-of-a-spark-sbt-test-suite-by-40-52d71219773f). 

Instance methods should be preceded with a pound sign (e.g. `#and`) and static methods should be preceded with a period (e.g. `.standardizeName`) in the `describe` block.  This follows [Ruby testing conventions](http://betterspecs.org/#describe).

Here is an example of a test for the `#and` instance method defined in the [functions class](https://spark.apache.org/docs/2.1.0/api/java/org/apache/spark/sql/functions.html) as follows:

```scala
class FunctionsSpec extends FunSpec with DataFrameComparer {

  import spark.implicits._
	
  describe("#and") {
	
    it ("returns true if both columns are true") {
	
      // some code
	
    }
    
  }
	
}
```

Here is an example of a test for the `.standardizeName` static method:

```scala
describe(".standardizeName") {

  it("consistenly formats the name") {
  
    // some code
    
  }
  
}
```
