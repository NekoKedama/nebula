# Nebula Graph Query Language (nGQL)

## About nGQL
`nGQL` is a declarative, textual query language like SQL, but for graphs. Unlike SQL, nGQL is all about expressing graph patterns. nGQL is a work in progress. We will add more features and further simplify the existing ones. There might be inconsistency between the syntax specs and implementation for the time being.
## Goals
- Easy to learn
- Easy to understand
- To focus on the online queries, also to provide the foundation for the offline computation

## Features
- Syntax is close to SQL, but not exactly the same (Easy to learn)
- Expandable
- Case insensitive
- Support basic graph traverse
- Support pattern match
- Support aggregation
- Support graph mutation
- Support distributed transaction (future release)
- Statement composition, but **NO** statement embedding (Easy to read)

## Prerequisite
- Directed property graph with schema

## Terminology
- **Graph Space** : A physically isolated space for different graph
- **Tag** : A label associated with a list of properties
 - Each tag has a name (human readable string), and internally each tag will be assigned a 32-bit integer
 - Each tag associates with a list of properties, each property has a name and a type
 - There could be dependencies between tags. The dependency is a constrain, for instance, if tag S depends on tag T, then tag S cannot exist unless tag T exists
- **Vertex** : A Node in the graph
 - Each vertex has a unique 64-bit (signed integer) ID (**VID**)
 - Each vertex can associate with multiple **tags**
- **Edge** : A Link between two vertices
 - Each edge can be uniquely identified by a tuple **<src_vid, dst_vid, edge_type, rank>**
 - ***Edge type (ET)*** is a human readable string, internally it will be assigned a 32-bit integer. The edge type decides the property list (schema) on the edge
 - ***Edge rank*** is an immutable user-assigned 64-bit signed integer. It affects the edge order between two vertices. The edge with a higher rank value comes first. When not specified, the default rank value is zero
 - Each edge can only be of one type
- **Path** : A ***non-forked*** connection with multiple vertices and edges between them
 - The length of a path is the number of the edges on the path, which is one less than the number of vertices
 - A path can be represented by a list of vertices, edge types, and rank. An edge is a special path with length==1
 ```
 <vid, edge_type[:rank], vid, ...>
 ```

## Language Specification
### General
- The entire set of statements can be categorized into three classes: **query**, **mutation**, and **administration**
- Every statement can yield a data set as the result. Each data set contains a schema (column name and type) and multiple data rows

### Composition
- Statements could be composed in two ways:
 - Statements could be piped together using operator "**|**", much like the pipe in the shell scripts. The result yielded from the previous statement could be redirected to the next statement as input
 - More than one statements can be batched together, separated by "**;**". The result of the last statement (or a <span style="color:blue">**RETURN**</span> statement is executed) will be returned as the result of the batch

### Data Types
- Simple type: **vid**, **integer** (int64), **double**, **float**, **bool**, **string**, **path**, **timestamp**, **year**, **month** (year/month), **date**, **datetime**
 - **vid** : 64-bit signed integer, representing a vertex ID
- List of simple types, such as **integer[]**, **double[]**, **string[]**
- **Map**: A list of KV pairs. The key must be a **string**, the value must be the same type for the given map
- **Object** (future release??): A list of KV pairs. The key mush be a **string**, the value can be any simple type
- **Tuple List**: *This is only used for return values*. It's composed by both meta data and data (multiple rows). The meta data includes the column names and their types.

### Type Conversion
- A simple typed value can be implicitly converted into a list
- A list can be implicitly converted into a one-column tuple list
 - "<type\>\_list" can be used as the column name

### Common BNF
<simple\_type> ::= **vid** | **integer** | **double** | **float** | **bool** | **string** | **path** | **timestamp** | **year** | **month** | **date** | **datetime**

<composite_type> ::=

<type\> ::= <simple_type> | <composite\_type>

<vid\_list> ::= **vid** (, **vid**)\* | "{" **vid** (, **vid**)\* "}"

<label\> ::= \[:alpha\] ([:alnum:] | "\_")\*

<underscore\_label> ::= ("\_")* <label\>

<field\_name> ::= <label\>

<field\_def\_list> ::= <field\_def> (, <field\_def>)\*

<field\_def> ::= <field_name>:<type\>

<tuple\_list\_decl> ::= <tuple\_schema> ":" <tuple\_data>

<tuple\_schema> ::= <field\_def\_list>

<tuple\_data> ::= <tuple\> (, <tuple\>)\* | "{" <tuple\> (, <tuple\>)\* "}"

<tuple\> ::= "(" **VALUE** (, **VALUE**)\* ")"

<var\> ::= "$" <label\>

### Statements
#### Choose a graph space
Nebula supports multiple graph spaces. Data in different graph spaces are physically isolated. Before executing a query, a graph space needs to be selected using the following statement

<span style="color:blue">**USE**</span>  <graphspace_name>

#### Return a data set
Simply return a single value or a data set

<span style="color:blue">**RETURN**</span> <return\_value\_decl>

<return\_value\_decl> ::= **vid** | <vid\_list> | <tuple\_list\_decl> | <var\>

#### Create a tag
The following statement defines a **new** tag

<span style="color:blue">**CREATE TAG**</span> <tag\_name> (<prop\_def\_list>)

<tag\_name> ::= <label\> <br>
<prop\_def\_list> ::= <prop\_def>+ <br>
<prop\_def> ::= <prop\_name>,<type\> <br>
<prop\_name> ::= <label\> <br>

#### Modify a tag type

#### Create an edge type
The following statement defines a **new** edge type

<span style="color:blue">**CREATE EDGE**</span> <edge\_type\_name> (<prop\_def\_list>)

<edge\_type\_name> := <label\>

#### Modify an edge type

#### Insert vertices
The following statement inserts one or more vertices

<span style="color:blue">**INSERT VERTEX**</span> [<span style="color:blue">**NO OVERWRITE**</span>] <tag\_list> <span style="color:blue">**VALUES**</span> <vertex\_list> <br/>

<tag\_list> ::= <tag\_name>(<prop\_list>) (, <tag\_name>(<prop\_list>))\* <br/>
<vertex\_list> ::= <vertex\_id>:(<prop\_value\_list>) (, <vertex\_id>:(<prop\_value\_list>))\* <br/>
<vertex\_id> ::= **vid** <br/>
<prop\_list> ::= <prop\_name> (, <prop\_name>)\* <br/>
<prop\_value\_list> ::= **VALUE** (, **VALUE**)\* <br/>

#### Insert edges

The following statement inserts one or more edges

<span style="color:blue">**INSERT EDGE**</span> [<span style="color:blue">**NO OVERWRITE**</span>] <edge\_type\_name> [(<prop\_list>)] <span style="color:blue">**VALUES**</span> (<edge\_value>)+

edge\_value ::= <vertex\_id> -> <vertex\_id> [@ <weight\>] : <prop\_value\_list>

#### Update a vertex
The following statement updates a vertex

<span style="color:blue">**UPDATE VERTEX**</span>  <vertex\_id>
<span style="color:blue">**SET**</span> \<update\_decl\>
[<span style="color:blue">**WHERE**</span> <conditions\>]
[<span style="color:blue">**YIELD**</span> <field\_list>]

<update\_decl> ::= <update\_form1> | <update\_form2> <br>
<update\_form1> ::= <prop\_name> = <expression\> {,<prop\_name> = <expression\>}+ <br>
<update\_form2> ::= (<prop\_list>) = (<value\_list>) | (<prop\_list>) = <var\> <br>

#### Update an edge
The following statement updates an edge

<span style="color:blue">**UPDATE EDGE**</span>  <vertex\_id> -> <vertex\_id> [@<weight\>] <span style="color:blue">**OF**</span> <edge\_type>
<span style="color:blue">**SET**</span> <update\_decl>
[<span style="color:blue">**WHERE**</span> <conditions\>]
[<span style="color:blue">**YIELD**</span> <field\_list>]

#### Traverse the graph
Navigate from given vertices to their neighbors according to the given conditions. It returns either a list of vertex IDs, or a list of tuples

<span style="color:blue">**GO**</span>
[<steps\_decl> <span style="color:blue">**STEPS**</span>]
<span style="color:blue">**FROM**</span> <data\_set\_decl>
[<span style="color:blue">**OVER**</span> [<span style="color:blue">**REVERSELY**</span>] <edge\_type\_decl>]
[<span style="color:blue">**WHERE**</span> <filter\_list>]
[<span style="color:blue">**YIELD**</span> <field\_list>]

<steps\_decl> ::= **integer** | **integer** <span style="color:blue">**TO**</span> **integer** | <span style="color:blue">**UPTO**</span> **integer** <br>
<data\_set\_decl> ::= [data\_set] [[<span style="color:blue">**AS**</span>] <label\>]<br/>
<data\_set> ::= **vid** | <vid\_list> | <tuple\_list\_decl> | <var\><br/>
<edge\_type\_decl> ::= <edge\_type\_list> [<span style="color:blue">**AS**</span> <label\>]
<edge\_type\_list> ::= <edge\_type> {, <edge\_type>}\* <br>
<edge\_type> ::= <label\> <br>

<filter\_list> ::= <filter\> {<span style="color:blue">**AND**</span> | <span style="color:blue">**OR**</span> <filter\>}\* <br>
<filter\> ::= <expression\> <span style="color:blue">**>**</span> | <span style="color:blue">**>=**</span> | <span style="color:blue">**<**</span> | <span style="color:blue">**<=**</span> | <span style="color:blue">**==**</span> | <span style="color:blue">**!=**</span> <expression\> | <expression\> <span style="color:blue">**IN**</span> <value\_list\> <br>
<field\_list> ::= <return\_field> {, <return\_field>}\* <br>
<return\_field> ::= <expression\> [<span style="color:blue">**AS**</span> <label\>] <br>

<span style="color:blue">**WHERE**</span> clause only applies to the results that are going to be returned. It will not be applied to the intermediate results (See the detail description of the <span style="color:blue">**STEP[S]**</span> clause)

When <span style="color:blue">**STEP[S]**</span> clause is skipped, it implies **one step**

When going out for one step from the given vertex, all neighbors will be checked against the <span style="color:blue">**WHERE**</span> clause, only results satisfied the <span style="color:blue">**WHERE**</span> clause will be returned

When going out for more than one step, <span style="color:blue">**WHERE**</span> clause will only be applied to the final results. It will not be applied to the intermediate results. Here is an example

```
GO 2 STEPS FROM me OVER friend WHERE birthday > "1988/1/1"
```

Obviously, you will probably guess the meaning of the query is to get all my fof (friend of friend) whose birthday is after 1988/1/1. You are absolutely right. We will not apply the filter to my friends (in the first step)

Here is another example

```
GO UPTO 3 STEPS FROM me OVER friend WHERE birthday > "1988/1/1/"
```

This query tries to find any friend of me whose birthday is after 1988/1/1. If it finds at least one, it will return all the results. If it cannot find any, it will check my friends of friends to see if anyone's birthday is after 1988/1/1. It will return all the non-empty results, otherwise it will check my friends of friends of friends.

So, similarly, next query tries to find anyone whose birthday is after 1988/1/1 starting from my 3-hop friends, and finishing at my 5-hop friends

```
GO 3 TO 5 STEPS FROM me OVER friend WHERE birthday > "1988/1/1/"
```

#### Search
Following statements looks for vertices or edges that match certain conditions

<span style="color:blue">**FIND VERTEX**</span>
<span style="color:blue">**WHERE**</span> <filter\_list>
[<span style="color:blue">**YIELD**</span> <field\_list>]

<span style="color:blue">**FIND EDGE**</span>
<span style="color:blue">**WHERE**</span> <filter\_list>
[<span style="color:blue">**YIELD**</span> <field\_list>]

#### Pattern match
The following statement does a pattern match, and can return tuple list or paths

<span style="color:blue">**MATCH**</span>
<regular\_path\_expression>
[<span style="color:blue">**FROM**</span> <data\_set\_decl>]
[<span style="color:blue">**WHERE**</span> <filter\_list>]
[<span style="color:blue">**YIELD**</span> <field\_list>]

### Property Reference
It's common to refer a property in the statement, such as in <span style="color:blue">**WHERE**</span> clause and <span style="color:blue">**YIELD**</span> clause. In nGQL, the reference to a property is defined as

<property_ref> ::= <object\> "." <prop\_name> <br>
<object\> ::= <alias\_name> | <alias\_with\_tag> | <var\> <br>
<alias\_name> ::= <label\> <br>
<alias\_with\_tag> ::= <alias\_name> '[' <tag\_name> "]" <br>

<var\> always starts with "$". There are two special variables: $- and $$.

$- refers to the input stream, while $$ refers to the destination objects

All property names start with a letter. There are a few system property names starting with "\_". All properties names starting with "\_" are reserved.

Here are some built-in properties:
<br/>\_id : Vertex id
<br/>\_type : Edge type
<br/>\_src : Source ID of the edge
<br/>\_dst : Destination ID of the edge
<br/>\_rank : Edge rank unuber
