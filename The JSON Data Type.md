## The JSON Data Type

- [创建JSON字符串、并存储](#json-values)
- [对JSON值进行规范化、合并和自动映射](#json-normalization)
- [搜索和修改JSON值](#json-paths)
- [JSON值的比较和排序](#json-comparison)
- [在JSON和非JSON值之间进行转换](#json-converting-between-types)
- [JSON值的聚合](#json-aggregation)  

在MySQL 5.7.8中，MySQL支持由[RFC 7159](https://tools.ietf.org/html/rfc7159)定义的本地`JSON`数据类型，它支持对JSON(JavaScript对象标记)中的数据进行有效访问。将"JSON"数据存储在`JSON`数据类型字段中相比存储在字符数据类型中有以下优势：

- JSON数据类型的字段存储`JSON`格式字符串时会对要存储的字符串进行格式校验。
- 优化的存储格式。存储在JSON数据类型列中的JSON字符串会被转换为一种内部格式，允许对文档元素进行快速读取访问。当服务器稍后必须读取以二进制格式存储的JSON值时，不再需要从文本表示中解析该值。二进制格式的结构是使服务器能够通过键或数组索引直接查找子对象或嵌套的值，而不需要在文档之前或之后读取所有值。

> Note
>
> 本文使用`JSON`来表示JSON数据类型，并且使用常规字体的“JSON”来表示JSON数据。

存储`JSON`文档所需的空间与`LONGBLOB`或`LONGTEXT`大致相同;具体参见[Section 11.8, “Data Type Storage Requirements”](https://dev.mysql.com/doc/refman/5.7/en/storage-requirements.html)，有一点务必记得，存储在`JSON`列中的任何JSON文档的大小限制为`max_allowed_packet`系统变量的值 (当服务器在内存中操作这个JSON值时，可能会大雨这个限制；但是在服务器中存储一定是被这个参数所限制的)。

`JSON`列不能设置默认值。

基于`JSON`格式，mysql提供了一套韩束来操作JSON值，比如创建，修改和查询。下面的讨论我们会给出一些关于这些操作的例子。要深入了解每个函数，请查看[Section 12.16, “JSON Functions”](https://dev.mysql.com/doc/refman/5.7/en/json-functions.html)。

还可以使用一组用于操作GeoJSON值的空间函数。 参见 [Section 12.15.11, “Spatial GeoJSON Functions”](https://dev.mysql.com/doc/refman/5.7/en/spatial-geojson-functions.html)。

`JSON`字段，和其他的二进制类型一样，不能直接进行索引；相应的，你可以创建从`JSON`列中取出一个存量值的索引。更加详细的资料请参考 [Indexing a Generated Column to Provide a JSON Column Index](https://dev.mysql.com/doc/refman/5.7/en/create-table-secondary-indexes.html#json-column-indirect-index)。

MySQL优化器还会在JSON表达式的虚拟列上查找合适的索引。  

MySQL NDB Cluster 7.5（7.5.2及更高版本）支持“JSON”列和MySQL的JSON函数，包括从`JSON`列生成的列中创建索引，作为无法对`JSON`列进行索引的解决方案。 每个[`NDB`]（https://dev.mysql.com/doc/refman/5.7/en/mysql-cluster.html） 表最多支持3个`JSON列。

接下来的一小段提供一些有关于创建和操作JSON的基本操作。

### 创建JSON

一个JSON数组包括一组被逗号分隔、[开始，]结束的字符串：

```
["abc", 10, null, true, false]
```

A JSON object contains a set of key-value pairs separated by commas and enclosed within `{` and `}` characters:
一个JSON对象包括一组被逗号分隔、`{`开始，`}`结束的键值对：

```
{"k1": "value", "k2": 10}
```

As the examples illustrate, JSON arrays and objects can contain scalar values that are strings or numbers, the JSON null literal, or the JSON boolean true or false literals. Keys in JSON objects must be strings. Temporal (date, time, or datetime) scalar values are also permitted:
如上面两个例子看到的，JSON数组和对象可以包括字符串、数字、null、以及true和false。但是JSON对象中的所有key一定是字符串。JSON中也可以包括时间格式（诸如日期、时间以及日期时间）：

```
["12:18:29.000000", "2015-07-29", "2015-07-29 12:18:29.000000"]
```

JSON数组元素和JSON对象中键值对可以嵌套：

```
[99, {"id": "HK500", "cost": 75.99}, ["hot", "cold"]]
{"k1": "value", "k2": [10, 20]}
```

You can also obtain JSON values from a number of functions supplied by MySQL for this purpose (see [Section 12.16.2, “Functions That Create JSON Values”](https://dev.mysql.com/doc/refman/5.7/en/json-creation-functions.html)) as well as by casting values of other types to the `JSON` type using [`CAST(*value* AS JSON)`](https://dev.mysql.com/doc/refman/5.7/en/cast-functions.html#function_cast) (see [Converting between JSON and non-JSON values](https://dev.mysql.com/doc/refman/5.7/en/json.html#json-converting-between-types)). The next several paragraphs describe how MySQL handles JSON values provided as input.

In MySQL, JSON values are written as strings. MySQL parses any string used in a context that requires a JSON value, and produces an error if it is not valid as JSON. These contexts include inserting a value into a column that has the `JSON` data type and passing an argument to a function that expects a JSON value (usually shown as *json_doc* or *json_val* in the documentation for MySQL JSON functions), as the following examples demonstrate:

- Attempting to insert a value into a `JSON` column succeeds if the value is a valid JSON value, but fails if it is not:

  ```
  mysql> CREATE TABLE t1 (jdoc JSON);
  Query OK, 0 rows affected (0.20 sec)

  mysql> INSERT INTO t1 VALUES('{"key1": "value1", "key2": "value2"}');
  Query OK, 1 row affected (0.01 sec)

  mysql> INSERT INTO t1 VALUES('[1, 2,');
  ERROR 3140 (22032) at line 2: Invalid JSON text: 
  "Invalid value." at position 6 in value (or column) '[1, 2,'.
  ```

  Positions for “at position *N*” in such error messages are 0-based, but should be considered rough indications of where the problem in a value actually occurs.

- The [`JSON_TYPE()`](https://dev.mysql.com/doc/refman/5.7/en/json-attribute-functions.html#function_json-type) function expects a JSON argument and attempts to parse it into a JSON value. It returns the value's JSON type if it is valid and produces an error otherwise:

  ```
  mysql> SELECT JSON_TYPE('["a", "b", 1]');
  +----------------------------+
  | JSON_TYPE('["a", "b", 1]') |
  +----------------------------+
  | ARRAY                      |
  +----------------------------+

  mysql> SELECT JSON_TYPE('"hello"');
  +----------------------+
  | JSON_TYPE('"hello"') |
  +----------------------+
  | STRING               |
  +----------------------+

  mysql> SELECT JSON_TYPE('hello');
  ERROR 3146 (22032): Invalid data type for JSON data in argument 1
  to function json_type; a JSON string or JSON type is required.
  ```

MySQL handles strings used in JSON context using the `utf8mb4` character set and `utf8mb4_bin` collation. Strings in other character sets are converted to `utf8mb4` as necessary. (For strings in the `ascii` or `utf8` character sets, no conversion is needed because `ascii` and `utf8` are subsets of `utf8mb4`.)

As an alternative to writing JSON values using literal strings, functions exist for composing JSON values from component elements. [`JSON_ARRAY()`](https://dev.mysql.com/doc/refman/5.7/en/json-creation-functions.html#function_json-array) takes a (possibly empty) list of values and returns a JSON array containing those values:

```
mysql> SELECT JSON_ARRAY('a', 1, NOW());
+----------------------------------------+
| JSON_ARRAY('a', 1, NOW())              |
+----------------------------------------+
| ["a", 1, "2015-07-27 09:43:47.000000"] |
+----------------------------------------+
```

[`JSON_OBJECT()`](https://dev.mysql.com/doc/refman/5.7/en/json-creation-functions.html#function_json-object) takes a (possibly empty) list of key-value pairs and returns a JSON object containing those pairs:

```
mysql> SELECT JSON_OBJECT('key1', 1, 'key2', 'abc');
+---------------------------------------+
| JSON_OBJECT('key1', 1, 'key2', 'abc') |
+---------------------------------------+
| {"key1": 1, "key2": "abc"}            |
+---------------------------------------+
```

[`JSON_MERGE()`](https://dev.mysql.com/doc/refman/5.7/en/json-modification-functions.html#function_json-merge) takes two or more JSON documents and returns the combined result:

```
mysql> SELECT JSON_MERGE('["a", 1]', '{"key": "value"}');
+--------------------------------------------+
| JSON_MERGE('["a", 1]', '{"key": "value"}') |
+--------------------------------------------+
| ["a", 1, {"key": "value"}]                 |
+--------------------------------------------+
```

For information about the merging rules, see [Normalization, Merging, and Autowrapping of JSON Values](https://dev.mysql.com/doc/refman/5.7/en/json.html#json-normalization).

JSON values can be assigned to user-defined variables:

```
mysql> SET @j = JSON_OBJECT('key', 'value');
mysql> SELECT @j;
+------------------+
| @j               |
+------------------+
| {"key": "value"} |
+------------------+
```

However, user-defined variables cannot be of `JSON` data type, so although `@j` in the preceding example looks like a JSON value and has the same character set and collation as a JSON value, it does *not* have the `JSON` data type. Instead, the result from [`JSON_OBJECT()`](https://dev.mysql.com/doc/refman/5.7/en/json-creation-functions.html#function_json-object) is converted to a string when assigned to the variable.

Strings produced by converting JSON values have a character set of `utf8mb4` and a collation of `utf8mb4_bin`:

```
mysql> SELECT CHARSET(@j), COLLATION(@j);
+-------------+---------------+
| CHARSET(@j) | COLLATION(@j) |
+-------------+---------------+
| utf8mb4     | utf8mb4_bin   |
+-------------+---------------+
```

Because `utf8mb4_bin` is a binary collation, comparison of JSON values is case sensitive.

```
mysql> SELECT JSON_ARRAY('x') = JSON_ARRAY('X');
+-----------------------------------+
| JSON_ARRAY('x') = JSON_ARRAY('X') |
+-----------------------------------+
|                                 0 |
+-----------------------------------+
```

Case sensitivity also applies to the JSON `null`, `true`, and `false` literals, which always must be written in lowercase:

```
mysql> SELECT JSON_VALID('null'), JSON_VALID('Null'), JSON_VALID('NULL');
+--------------------+--------------------+--------------------+
| JSON_VALID('null') | JSON_VALID('Null') | JSON_VALID('NULL') |
+--------------------+--------------------+--------------------+
|                  1 |                  0 |                  0 |
+--------------------+--------------------+--------------------+

mysql> SELECT CAST('null' AS JSON);
+----------------------+
| CAST('null' AS JSON) |
+----------------------+
| null                 |
+----------------------+
1 row in set (0.00 sec)

mysql> SELECT CAST('NULL' AS JSON);
ERROR 3141 (22032): Invalid JSON text in argument 1 to function cast_as_json:
"Invalid value." at position 0 in 'NULL'.
```

Case sensitivity of the JSON literals differs from that of the SQL `NULL`, `TRUE`, and `FALSE` literals, which can be written in any lettercase:

```
mysql> SELECT ISNULL(null), ISNULL(Null), ISNULL(NULL);
+--------------+--------------+--------------+
| ISNULL(null) | ISNULL(Null) | ISNULL(NULL) |
+--------------+--------------+--------------+
|            1 |            1 |            1 |
+--------------+--------------+--------------+
```

Sometimes it may be necessary or desirable to insert quote characters (`"` or `'`) into a JSON document. Assume for this example that you want to insert some JSON objects containing strings representing sentences that state some facts about MySQL, each paired with an appropriate keyword, into a table created using the SQL statement shown here:

```
mysql> CREATE TABLE facts (sentence JSON);
```

Among these keyword-sentence pairs is this one:

```
mascot: The MySQL mascot is a dolphin named "Sakila".
```

One way to insert this as a JSON object into the `facts` table is to use the MySQL [`JSON_OBJECT()`](https://dev.mysql.com/doc/refman/5.7/en/json-creation-functions.html#function_json-object) function. In this case, you must escape each quote character using a backslash, as shown here:

```
mysql> INSERT INTO facts VALUES 
     >   (JSON_OBJECT("mascot", "Our mascot is a dolphin named \"Sakila\"."));
```

This does not work in the same way if you insert the value as a JSON object literal, in which case, you must use the double backslash escape sequence, like this:

```
mysql> INSERT INTO facts VALUES 
     >   ('{"mascot": "Our mascot is a dolphin named \\"Sakila\\"."}');
```

Using the double backslash keeps MySQL from performing escape sequence processing, and instead causes it to pass the string literal to the storage engine for processing. After inserting the JSON object in either of the ways just shown, you can see that the backslashes are present in the JSON column value by doing a simple [`SELECT`](https://dev.mysql.com/doc/refman/5.7/en/select.html), like this:

```
mysql> SELECT sentence FROM facts;
+---------------------------------------------------------+
| sentence                                                |
+---------------------------------------------------------+
| {"mascot": "Our mascot is a dolphin named \"Sakila\"."} |
+---------------------------------------------------------+
```

To look up this particular sentence employing `mascot` as the key, you can use the column-path operator [`->`](https://dev.mysql.com/doc/refman/5.7/en/json-search-functions.html#operator_json-column-path), as shown here:

```
mysql> SELECT col->"$.mascot" FROM qtest;
+---------------------------------------------+
| col->"$.mascot"                             |
+---------------------------------------------+
| "Our mascot is a dolphin named \"Sakila\"." |
+---------------------------------------------+
1 row in set (0.00 sec)
```

This leaves the backslashes intact, along with the surrounding quote marks. To display the desired value using `mascot` as the key, but without including the surrounding quote marks or any escapes, use the inline path operator [`->>`](https://dev.mysql.com/doc/refman/5.7/en/json-search-functions.html#operator_json-inline-path), like this:

```
mysql> SELECT sentence->>"$.mascot" FROM facts;
+-----------------------------------------+
| sentence->>"$.mascot"                   |
+-----------------------------------------+
| Our mascot is a dolphin named "Sakila". |
+-----------------------------------------+
```

Note

The previous example does not work as shown if the [`NO_BACKSLASH_ESCAPES`](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html#sqlmode_no_backslash_escapes) server SQL mode is enabled. If this mode is set, a single backslash instead of double backslashes can be used to insert the JSON object literal, and the backslashes are preserved. If you use the `JSON_OBJECT()` function when performing the insert and this mode is set, you must alternate single and double quotes, like this:

```
mysql> INSERT INTO facts VALUES 
     > (JSON_OBJECT('mascot', 'Our mascot is a dolphin named "Sakila".'));
```

See the description of the [`JSON_UNQUOTE()`](https://dev.mysql.com/doc/refman/5.7/en/json-modification-functions.html#function_json-unquote) function for more information about the effects of this mode on escaped characters in JSON values.

### Normalization, Merging, and Autowrapping of JSON Values

When a string is parsed and found to be a valid JSON document, it is also normalized: Members with keys that duplicate a key found earlier in the document are discarded (even if the values differ). The object value produced by the following[`JSON_OBJECT()`](https://dev.mysql.com/doc/refman/5.7/en/json-creation-functions.html#function_json-object) call does not include the second `key1` element because that key name occurs earlier in the value:

```
mysql> SELECT JSON_OBJECT('key1', 1, 'key2', 'abc', 'key1', 'def');
+------------------------------------------------------+
| JSON_OBJECT('key1', 1, 'key2', 'abc', 'key1', 'def') |
+------------------------------------------------------+
| {"key1": 1, "key2": "abc"}                           |
+------------------------------------------------------+
```

Note

This “first key wins” handling of duplicate keys is not consistent with [RFC 7159](https://tools.ietf.org/html/rfc7159). This is a known issue in MySQL 5.7, which is fixed in MySQL 8.0. (Bug #86866, Bug #26369555)

MySQL also discards extra whitespace between keys, values, or elements in the original JSON document. To make lookups more efficient, it also sorts the keys of a JSON object. *You should be aware that the result of this ordering is subject to change and not guaranteed to be consistent across releases.*

MySQL functions that produce JSON values (see [Section 12.16.2, “Functions That Create JSON Values”](https://dev.mysql.com/doc/refman/5.7/en/json-creation-functions.html)) always return normalized values.

#### Merging JSON Values

In contexts that combine multiple arrays, the arrays are merged into a single array by concatenating arrays named later to the end of the first array. In the following example, [`JSON_MERGE()`](https://dev.mysql.com/doc/refman/5.7/en/json-modification-functions.html#function_json-merge) merges its arguments into a single array:

```
mysql> SELECT JSON_MERGE('[1, 2]', '["a", "b"]', '[true, false]');
+-----------------------------------------------------+
| JSON_MERGE('[1, 2]', '["a", "b"]', '[true, false]') |
+-----------------------------------------------------+
| [1, 2, "a", "b", true, false]                       |
+-----------------------------------------------------+
```

Normalization is also performed when values are inserted into JSON columns, as shown here:

```
mysql> CREATE TABLE t1 (c1 JSON);

mysql> INSERT INTO t1 VALUES 
     >     ('{"x": 17, "x": "red"}'),
     >     ('{"x": 17, "x": "red", "x": [3, 5, 7]}');

mysql> SELECT c1 FROM t1;
+-----------+
| c1        |
+-----------+
| {"x": 17} |
| {"x": 17} |
+-----------+
```

Multiple objects when merged produce a single object. If multiple objects have the same key, the value for that key in the resulting merged object is an array containing the key values:

```
mysql> SELECT JSON_MERGE('{"a": 1, "b": 2}', '{"c": 3, "a": 4}');
+----------------------------------------------------+
| JSON_MERGE('{"a": 1, "b": 2}', '{"c": 3, "a": 4}') |
+----------------------------------------------------+
| {"a": [1, 4], "b": 2, "c": 3}                      |
+----------------------------------------------------+
```

Nonarray values used in a context that requires an array value are autowrapped: The value is surrounded by `[` and `]`characters to convert it to an array. In the following statement, each argument is autowrapped as an array (`[1]`, `[2]`). These are then merged to produce a single result array:

```
mysql> SELECT JSON_MERGE('1', '2');
+----------------------+
| JSON_MERGE('1', '2') |
+----------------------+
| [1, 2]               |
+----------------------+
```

Array and object values are merged by autowrapping the object as an array and merging the two arrays:

```
mysql> SELECT JSON_MERGE('[10, 20]', '{"a": "x", "b": "y"}');
+------------------------------------------------+
| JSON_MERGE('[10, 20]', '{"a": "x", "b": "y"}') |
+------------------------------------------------+
| [10, 20, {"a": "x", "b": "y"}]                 |
+------------------------------------------------+
```

### Searching and Modifying JSON Values

A JSON path expression selects a value within a JSON document.

Path expressions are useful with functions that extract parts of or modify a JSON document, to specify where within that document to operate. For example, the following query extracts from a JSON document the value of the member with the`name` key:

```
mysql> SELECT JSON_EXTRACT('{"id": 14, "name": "Aztalan"}', '$.name');
+---------------------------------------------------------+
| JSON_EXTRACT('{"id": 14, "name": "Aztalan"}', '$.name') |
+---------------------------------------------------------+
| "Aztalan"                                               |
+---------------------------------------------------------+
```

Path syntax uses a leading `$` character to represent the JSON document under consideration, optionally followed by selectors that indicate successively more specific parts of the document:

- A period followed by a key name names the member in an object with the given key. The key name must be specified within double quotation marks if the name without quotes is not legal within path expressions (for example, if it contains a space).

- `[*N*]` appended to a *path* that selects an array names the value at position *N* within the array. Array positions are integers beginning with zero. If *path* does not select an array value, *path*[0] evaluates to the same value as *path*:

  ```
  mysql> SELECT JSON_SET('"x"', '$[0]', 'a');
  +------------------------------+
  | JSON_SET('"x"', '$[0]', 'a') |
  +------------------------------+
  | "a"                          |
  +------------------------------+
  1 row in set (0.00 sec)
  ```

- Paths can contain `*` or `**` wildcards:

  - `.[*]` evaluates to the values of all members in a JSON object.
  - `[*]` evaluates to the values of all elements in a JSON array.
  - `*prefix****suffix*` evaluates to all paths that begin with the named prefix and end with the named suffix.

- A path that does not exist in the document (evaluates to nonexistent data) evaluates to `NULL`.

Let `$` refer to this JSON array with three elements:

```
[3, {"a": [5, 6], "b": 10}, [99, 100]]
```

Then:

- `$[0]` evaluates to `3`.
- `$[1]` evaluates to `{"a": [5, 6], "b": 10}`.
- `$[2]` evaluates to `[99, 100]`.
- `$[3]` evaluates to `NULL` (it refers to the fourth array element, which does not exist).

Because `$[1]` and `$[2]` evaluate to nonscalar values, they can be used as the basis for more-specific path expressions that select nested values. Examples:

- `$[1].a` evaluates to `[5, 6]`.
- `$[1].a[1]` evaluates to `6`.
- `$[1].b` evaluates to `10`.
- `$[2][0]` evaluates to `99`.

As mentioned previously, path components that name keys must be quoted if the unquoted key name is not legal in path expressions. Let `$` refer to this value:

```
{"a fish": "shark", "a bird": "sparrow"}
```

The keys both contain a space and must be quoted:

- `$."a fish"` evaluates to `shark`.
- `$."a bird"` evaluates to `sparrow`.

Paths that use wildcards evaluate to an array that can contain multiple values:

```
mysql> SELECT JSON_EXTRACT('{"a": 1, "b": 2, "c": [3, 4, 5]}', '$.*');
+---------------------------------------------------------+
| JSON_EXTRACT('{"a": 1, "b": 2, "c": [3, 4, 5]}', '$.*') |
+---------------------------------------------------------+
| [1, 2, [3, 4, 5]]                                       |
+---------------------------------------------------------+
mysql> SELECT JSON_EXTRACT('{"a": 1, "b": 2, "c": [3, 4, 5]}', '$.c[*]');
+------------------------------------------------------------+
| JSON_EXTRACT('{"a": 1, "b": 2, "c": [3, 4, 5]}', '$.c[*]') |
+------------------------------------------------------------+
| [3, 4, 5]                                                  |
+------------------------------------------------------------+
```

In the following example, the path `$**.b` evaluates to multiple paths (`$.a.b` and `$.c.b`) and produces an array of the matching path values:

```
mysql> SELECT JSON_EXTRACT('{"a": {"b": 1}, "c": {"b": 2}}', '$**.b');
+---------------------------------------------------------+
| JSON_EXTRACT('{"a": {"b": 1}, "c": {"b": 2}}', '$**.b') |
+---------------------------------------------------------+
| [1, 2]                                                  |
+---------------------------------------------------------+
```

In MySQL 5.7.9 and later, you can use [`*column*->*path*`](https://dev.mysql.com/doc/refman/5.7/en/json-search-functions.html#operator_json-column-path) with a JSON column identifier and JSON path expression as a synonym for [`JSON_EXTRACT(*column*, *path*)`](https://dev.mysql.com/doc/refman/5.7/en/json-search-functions.html#function_json-extract). See [Section 12.16.3, “Functions That Search JSON Values”](https://dev.mysql.com/doc/refman/5.7/en/json-search-functions.html), for more information. See also [Indexing a Generated Column to Provide a JSON Column Index](https://dev.mysql.com/doc/refman/5.7/en/create-table-secondary-indexes.html#json-column-indirect-index).

Some functions take an existing JSON document, modify it in some way, and return the resulting modified document. Path expressions indicate where in the document to make changes. For example, the [`JSON_SET()`](https://dev.mysql.com/doc/refman/5.7/en/json-modification-functions.html#function_json-set), [`JSON_INSERT()`](https://dev.mysql.com/doc/refman/5.7/en/json-modification-functions.html#function_json-insert), and[`JSON_REPLACE()`](https://dev.mysql.com/doc/refman/5.7/en/json-modification-functions.html#function_json-replace) functions each take a JSON document, plus one or more path/value pairs that describe where to modify the document and the values to use. The functions differ in how they handle existing and nonexisting values within the document.

Consider this document:

```
mysql> SET @j = '["a", {"b": [true, false]}, [10, 20]]';
```

[`JSON_SET()`](https://dev.mysql.com/doc/refman/5.7/en/json-modification-functions.html#function_json-set) replaces values for paths that exist and adds values for paths that do not exist:.

```
mysql> SELECT JSON_SET(@j, '$[1].b[0]', 1, '$[2][2]', 2);
+--------------------------------------------+
| JSON_SET(@j, '$[1].b[0]', 1, '$[2][2]', 2) |
+--------------------------------------------+
| ["a", {"b": [1, false]}, [10, 20, 2]]      |
+--------------------------------------------+
```

In this case, the path `$[1].b[0]` selects an existing value (`true`), which is replaced with the value following the path argument (`1`). The path `$[2][2]` does not exist, so the corresponding value (`2`) is added to the value selected by `$[2]`.

[`JSON_INSERT()`](https://dev.mysql.com/doc/refman/5.7/en/json-modification-functions.html#function_json-insert) adds new values but does not replace existing values:

```
mysql> SELECT JSON_INSERT(@j, '$[1].b[0]', 1, '$[2][2]', 2);
+-----------------------------------------------+
| JSON_INSERT(@j, '$[1].b[0]', 1, '$[2][2]', 2) |
+-----------------------------------------------+
| ["a", {"b": [true, false]}, [10, 20, 2]]      |
+-----------------------------------------------+
```

[`JSON_REPLACE()`](https://dev.mysql.com/doc/refman/5.7/en/json-modification-functions.html#function_json-replace) replaces existing values and ignores new values:

```
mysql> SELECT JSON_REPLACE(@j, '$[1].b[0]', 1, '$[2][2]', 2);
+------------------------------------------------+
| JSON_REPLACE(@j, '$[1].b[0]', 1, '$[2][2]', 2) |
+------------------------------------------------+
| ["a", {"b": [1, false]}, [10, 20]]             |
+------------------------------------------------+
```

The path/value pairs are evaluated left to right. The document produced by evaluating one pair becomes the new value against which the next pair is evaluated.

`JSON_REMOVE()` takes a JSON document and one or more paths that specify values to be removed from the document. The return value is the original document minus the values selected by paths that exist within the document:

```
mysql> SELECT JSON_REMOVE(@j, '$[2]', '$[1].b[1]', '$[1].b[1]');
+---------------------------------------------------+
| JSON_REMOVE(@j, '$[2]', '$[1].b[1]', '$[1].b[1]') |
+---------------------------------------------------+
| ["a", {"b": [true]}]                              |
+---------------------------------------------------+
```

The paths have these effects:

- `$[2]` matches `[10, 20]` and removes it.
- The first instance of `$[1].b[1]` matches `false` in the `b` element and removes it.
- The second instance of `$[1].b[1]` matches nothing: That element has already been removed, the path no longer exists, and has no effect.

### Comparison and Ordering of JSON Values

JSON values can be compared using the [`=`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_equal), [`<`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_less-than), [`<=`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_less-than-or-equal), [`>`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_greater-than), [`>=`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_greater-than-or-equal), [`<>`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_not-equal), [`!=`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_not-equal), and [`<=>`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_equal-to) operators.

The following comparison operators and functions are not yet supported with JSON values:

- [`BETWEEN`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_between)
- [`IN()`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#function_in)
- [`GREATEST()`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#function_greatest)
- [`LEAST()`](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#function_least)

A workaround for the comparison operators and functions just listed is to cast JSON values to a native MySQL numeric or string data type so they have a consistent non-JSON scalar type.

Comparison of JSON values takes place at two levels. The first level of comparison is based on the JSON types of the compared values. If the types differ, the comparison result is determined solely by which type has higher precedence. If the two values have the same JSON type, a second level of comparison occurs using type-specific rules.

The following list shows the precedences of JSON types, from highest precedence to the lowest. (The type names are those returned by the [`JSON_TYPE()`](https://dev.mysql.com/doc/refman/5.7/en/json-attribute-functions.html#function_json-type) function.) Types shown together on a line have the same precedence. Any value having a JSON type listed earlier in the list compares greater than any value having a JSON type listed later in the list.

```
BLOB
BIT
OPAQUE
DATETIME
TIME
DATE
BOOLEAN
ARRAY
OBJECT
STRING
INTEGER, DOUBLE
NULL
```

For JSON values of the same precedence, the comparison rules are type specific:

- `BLOB`

  The first *N* bytes of the two values are compared, where *N* is the number of bytes in the shorter value. If the first *N*bytes of the two values are identical, the shorter value is ordered before the longer value.

- `BIT`

  Same rules as for `BLOB`.

- `OPAQUE`

  Same rules as for `BLOB`. `OPAQUE` values are values that are not classified as one of the other types.

- `DATETIME`

  A value that represents an earlier point in time is ordered before a value that represents a later point in time. If two values originally come from the MySQL `DATETIME` and `TIMESTAMP` types, respectively, they are equal if they represent the same point in time.

- `TIME`

  The smaller of two time values is ordered before the larger one.

- `DATE`

  The earlier date is ordered before the more recent date.

- `ARRAY`

  Two JSON arrays are equal if they have the same length and values in corresponding positions in the arrays are equal.

  If the arrays are not equal, their order is determined by the elements in the first position where there is a difference. The array with the smaller value in that position is ordered first. If all values of the shorter array are equal to the corresponding values in the longer array, the shorter array is ordered first.

  Example:

  ```
  [] < ["a"] < ["ab"] < ["ab", "cd", "ef"] < ["ab", "ef"]
  ```

- `BOOLEAN`

  The JSON false literal is less than the JSON true literal.

- `OBJECT`

  Two JSON objects are equal if they have the same set of keys, and each key has the same value in both objects.

  Example:

  ```
  {"a": 1, "b": 2} = {"b": 2, "a": 1}
  ```

  The order of two objects that are not equal is unspecified but deterministic.

- `STRING`

  Strings are ordered lexically on the first *N* bytes of the `utf8mb4` representation of the two strings being compared, where *N* is the length of the shorter string. If the first *N* bytes of the two strings are identical, the shorter string is considered smaller than the longer string.

  Example:

  ```
  "a" < "ab" < "b" < "bc"
  ```

  This ordering is equivalent to the ordering of SQL strings with collation `utf8mb4_bin`. Because `utf8mb4_bin` is a binary collation, comparison of JSON values is case sensitive:

  ```
  "A" < "a"
  ```

- `INTEGER`, `DOUBLE`

  JSON values can contain exact-value numbers and approximate-value numbers. For a general discussion of these types of numbers, see [Section 9.1.2, “Numeric Literals”](https://dev.mysql.com/doc/refman/5.7/en/number-literals.html).

  The rules for comparing native MySQL numeric types are discussed in [Section 12.2, “Type Conversion in Expression Evaluation”](https://dev.mysql.com/doc/refman/5.7/en/type-conversion.html), but the rules for comparing numbers within JSON values differ somewhat:

  - In a comparison between two columns that use the native MySQL [`INT`](https://dev.mysql.com/doc/refman/5.7/en/integer-types.html) and [`DOUBLE`](https://dev.mysql.com/doc/refman/5.7/en/floating-point-types.html) numeric types, respectively, it is known that all comparisons involve an integer and a double, so the integer is converted to double for all rows. That is, exact-value numbers are converted to approximate-value numbers.

  - On the other hand, if the query compares two JSON columns containing numbers, it cannot be known in advance whether numbers will be integer or double. To provide the most consistent behavior across all rows, MySQL converts approximate-value numbers to exact-value numbers. The resulting ordering is consistent and does not lose precision for the exact-value numbers. For example, given the scalars 9223372036854775805, 9223372036854775806, 9223372036854775807 and 9.223372036854776e18, the order is such as this:

    ```
    9223372036854775805 < 9223372036854775806 < 9223372036854775807
    < 9.223372036854776e18 = 9223372036854776000 < 9223372036854776001
    ```

  Were JSON comparisons to use the non-JSON numeric comparison rules, inconsistent ordering could occur. The usual MySQL comparison rules for numbers yield these orderings:

  - Integer comparison:

    ```
    9223372036854775805 < 9223372036854775806 < 9223372036854775807
    ```

    (not defined for 9.223372036854776e18)

  - Double comparison:

    ```
    9223372036854775805 = 9223372036854775806 = 9223372036854775807 = 9.223372036854776e18
    ```

For comparison of any JSON value to SQL `NULL`, the result is `UNKNOWN`.

For comparison of JSON and non-JSON values, the non-JSON value is converted to JSON according to the rules in the following table, then the values compared as described previously.

### Converting between JSON and non-JSON values

The following table provides a summary of the rules that MySQL follows when casting between JSON values and values of other types:

**Table 11.2 JSON Conversion Rules**

| other type                               | CAST(other type AS JSON)                 | CAST(JSON AS other type)                 |
| ---------------------------------------- | ---------------------------------------- | ---------------------------------------- |
| JSON                                     | No change                                | No change                                |
| utf8 character type (`utf8mb4`, `utf8`, `ascii`) | The string is parsed into a JSON value.  | The JSON value is serialized into a `utf8mb4`string. |
| Other character types                    | Other character encodings are implicitly converted to `utf8mb4` and treated as described for utf8 character type. | The JSON value is serialized into a `utf8mb4`string, then cast to the other character encoding. The result may not be meaningful. |
| `NULL`                                   | Results in a `NULL` value of type JSON.  | Not applicable.                          |
| Geometry types                           | The geometry value is converted into a JSON document by calling [`ST_AsGeoJSON()`](https://dev.mysql.com/doc/refman/5.7/en/spatial-geojson-functions.html#function_st-asgeojson). | Illegal operation. Workaround: Pass the result of [`CAST(*json_val* AS CHAR)`](https://dev.mysql.com/doc/refman/5.7/en/cast-functions.html#function_cast) to[`ST_GeomFromGeoJSON()`](https://dev.mysql.com/doc/refman/5.7/en/spatial-geojson-functions.html#function_st-geomfromgeojson). |
| All other types                          | Results in a JSON document consisting of a single scalar value. | Succeeds if the JSON document consists of a single scalar value of the target type and that scalar value can be cast to the target type. Otherwise, returns `NULL` and produces a warning. |

`ORDER BY` and `GROUP BY` for JSON values works according to these principles:

- Ordering of scalar JSON values uses the same rules as in the preceding discussion.
- For ascending sorts, SQL `NULL` orders before all JSON values, including the JSON null literal; for descending sorts, SQL `NULL` orders after all JSON values, including the JSON null literal.
- Sort keys for JSON values are bound by the value of the [`max_sort_length`](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_max_sort_length) system variable, so keys that differ only after the first [`max_sort_length`](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_max_sort_length) bytes compare as equal.
- Sorting of nonscalar values is not currently supported and a warning occurs.

For sorting, it can be beneficial to cast a JSON scalar to some other native MySQL type. For example, if a column named`jdoc` contains JSON objects having a member consisting of an `id` key and a nonnegative value, use this expression to sort by `id` values:

```
ORDER BY CAST(JSON_EXTRACT(jdoc, '$.id') AS UNSIGNED)
```

If there happens to be a generated column defined to use the same expression as in the `ORDER BY`, the MySQL optimizer recognizes that and considers using the index for the query execution plan. See [Section 8.3.10, “Optimizer Use of Generated Column Indexes”](https://dev.mysql.com/doc/refman/5.7/en/generated-column-index-optimizations.html).

### Aggregation of JSON Values

For aggregation of JSON values, SQL `NULL` values are ignored as for other data types. Non-`NULL` values are converted to a numeric type and aggregated, except for [`MIN()`](https://dev.mysql.com/doc/refman/5.7/en/group-by-functions.html#function_min), [`MAX()`](https://dev.mysql.com/doc/refman/5.7/en/group-by-functions.html#function_max), and [`GROUP_CONCAT()`](https://dev.mysql.com/doc/refman/5.7/en/group-by-functions.html#function_group-concat). The conversion to number should produce a meaningful result for JSON values that are numeric scalars, although (depending on the values) truncation and loss of precision may occur. Conversion to number of other JSON values may not produce a meaningful result.
