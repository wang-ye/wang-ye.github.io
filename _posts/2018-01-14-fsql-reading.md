---
layout: post
title:  "Fsql - A Go Tool"
date:   2018-01-14 22:49:03 -0800
---
[fsql](https://github.com/kshvmdn/fsql) is a utility in golang that converts file searching into a sql-like query. For example, when running shell command

```shell
fsql "SELECT name FROM ~/Desktop, ~/Downloads WHERE name LIKE %csc%"
```
, it will display all file names containing "csc" under locations `~/Desktop` and `~/Downloads`. Under the hood, it is essentially a converter that translates SQL-like syntax to a Go program to list the files. Although a powerful tool, fsql is well organized and is easy for Go beginners to understand. In this post, I will discuss its internals and my learnings when reading it. I will divide it into several parts: Overall flow, Parsing, Execute and Go features.

## Overall Flow
The starting point is [**main.go**](https://github.com/kshvmdn/fsql/blob/master/cmd/fsql/main.go). It first extracts the query from command line, and then calls the *Run* method to analyze and execute the query.

```go
// In cmd/fsql/main.go
func main() {
    // flag processing code omitted
    // readInput() returns all arguments in the command line.
    if err := fsql.Run(readInput()); err != nil {
        log.Fatal(err.Error())
    }
}
```

The *Run* method provides the overall flow. It first parses the query and stores the parsed result in the Query struct *q*. Then, it visits the file system based on the criteria in *q*. Finally, the results are sent to stdout for printing. In the rest of this section, we will focus on the parser internals.

```go
// In fsql.go
// Run parses the input and executes the resultant query.
func Run(input string) error {
    q, err := parser.Run(input)
    if err != nil {
        return err
    }

    // Find length of the longest name to normalize name output.
    var max = 0
    var results = make([]map[string]interface{}, 0)

    err = q.Execute(
        func(path string, info os.FileInfo, result map[string]interface{}) {
            results = append(results, result)
            if !q.HasAttribute("name") {
                return
            }
            if s, ok := result["name"].(string); ok && len(s) > max {
                max = len(s)
            }
        },
    )
    if err != nil {
        return err
    }

    for _, result := range results {
        var buf bytes.Buffer
        for j, attribute := range q.Attributes {
            // If the current attribute is "name", pad the output string by `max`
            // spaces.
            format := "%v"
            if attribute == "name" {
                format = fmt.Sprintf("%%-%ds", max)
            }
            buf.WriteString(fmt.Sprintf(format, result[attribute]))
            if j != len(q.Attributes)-1 {
                buf.WriteString("\t")
            }
        }
        fmt.Printf("%s\n", buf.String())
    }

    return nil
}
```

The following two sections will discuss the *parser.Run* and *query.Execute* method in details.

## Parsing Process
In the parsing process, Fsql parses and convert given query string into a data structure **Query**, which conveys the same information as the original query. We will first discuss the **Query** struct, and then figure out how the parsing process converts the raw query string to the Query object.

### The Query Struct
Here is the struct **Query** definition:

```go
// Query represents an input query.
type Query struct {
    Attributes []string
    Modifiers  map[string][]Modifier

    Sources       map[string][]string
    SourceAliases map[string]string

    ConditionTree *ConditionNode
}
```

Query has five key fields. Combining these fields together, we know the file locations, file patterns to look for and the file attributes to display.

The attributes fields here are the file attributes we want to show.
The available attributes are defined as 

```go
var allAttributes = []string{"mode", "size", "time", "hash", "name"}
```

You can use "Select mode, size ..." to only show attributes mode and size. To select all attributes, we can use the special keywords "ALL" and "*".

Modifier specifies how input and output values should be processed. You can see the full list of modifiers [here](https://github.com/kshvmdn/fsql#attribute-modifiers). You can specify the size or time of the file using 

```shell
SELECT ALL FROM /tmp WHERE FORMAT(size, KB) >= 10.5
```

Sources dictates where to find the files. They usually appear in the where statement. The SourceAliases is the aliases for the sources. It is often used to simplify the SQL queries.

The ConditionTree is relatively complex. It is used to represent the condition in WHERE clause. As name suggests, conditions are tree-based structure to represent the condition **(a = b) AND (c = d)**.

### Tokenization
The *first step* of the whole parsing process is tokenization. Fsql defines a set of syntaxes along with the corresponding tokenizer/parser. Tokenization happens first. There is a predefined set of tokens such as Select, Where, Like, Identifier .... You can find the detailed list [here](https://github.com/kshvmdn/fsql/blob/master/tokenizer/token.go#L8). The tokenization process converts query string into a set of tokens. For example, the query

```go
input := `
    SELECT
    name, size
    FROM
    ~/Desktop WHERE
    name LIKE %go
`
```

can be translated to the following token sequence

```go
expected := []Token{
  {Type: Select, Raw: "SELECT"},
  {Type: Identifier, Raw: "name"},
  {Type: Comma, Raw: ","},
  {Type: Identifier, Raw: "size"},
  {Type: From, Raw: "FROM"},
  {Type: Identifier, Raw: "~/Desktop"},
  {Type: Where, Raw: "WHERE"},
  {Type: Identifier, Raw: "name"},
  {Type: Like, Raw: "LIKE"},
  {Type: Identifier, Raw: "%go"},
}
```

### Parsing
After tokenization stage, parser goes over all the tokens and convert them to a *Query* struct.

```go
// parser/parser.go
// Run parses the input string and returns the parsed AST (query).
func Run(input string) (*query.Query, error) {
    return (&parser{}).parse(input)
}

type parser struct {
    tokenizer *tokenizer.Tokenizer
    current   *tokenizer.Token
    expected  tokenizer.TokenType
}

// parse runs the respective parser function on each clause of the query.
func (p *parser) parse(input string) (*query.Query, error) {
    q := query.NewQuery()
    p.tokenizer = tokenizer.NewTokenizer(input)
    if err := p.parseSelectClause(q); err != nil {
        return nil, err
    }
    if err := p.parseFromClause(q); err != nil {
        return nil, err
    }
    if err := p.parseWhereClause(q); err != nil {
        return nil, err
    }
    return q, nil
}
```

The *Run* method first calls parser's *parse* method, which converts the query string to a Query. After that, parseSelectClause, parseFromClause and parseWhereClause are called in sequence to parse the query. The first two methods are relatively simple, and I will focus on the last method - *parseWhereClause*.

```go
// parser/parser.go
// parseWhereClause parses the WHERE clause of the query.
func (p *parser) parseWhereClause(q *query.Query) error {
    if p.expect(tokenizer.Where) == nil {
        err := p.currentError()
        if p.expect(tokenizer.Identifier) == nil {
            return nil
        }
        return err
    }
    root, err := p.parseConditionTree()
    if err != nil {
        return err
    }
    q.ConditionTree = root

    return nil
}
```

*parseWhereClause* looks for the token "WHERE" first. If present, it will try to understand the condition by parsing the remaining tokens. *parseConditionTree* converts the condition expression into a tree-based structure called *ConditionNode*. The condition can be a sub-query or condition with logical operators such as *OR* or *AND*.

```go
// In parser/condition.go
// parseConditionTree parses the condition tree passed to the WHERE clause.
func (p *parser) parseConditionTree() (*query.ConditionNode, error) {
    stack := lane.NewStack()
    errFailedToParse := errors.New("failed to parse conditions")

    for {
        if p.current = p.tokenizer.Next(); p.current == nil {
            break
        }

        switch p.current.Type {

        case tokenizer.Not:
            // Ignored for simplicity ..
        case tokenizer.Identifier:
            condition, err := p.parseCondition()
            if err != nil {
                return nil, err
            }
            if condition == nil {
                return nil, p.currentError()
            }

            if condition.IsSubquery {
                if err := p.parseSubquery(condition); err != nil {
                    return nil, err
                }
            }

            leafNode := &query.ConditionNode{Condition: condition}
            if prevNode, ok := stack.Pop().(*query.ConditionNode); !ok {
                stack.Push(leafNode)
            } else if prevNode.Condition == nil {
                prevNode.Right = leafNode
                stack.Push(prevNode)
            } else {
                return nil, errFailedToParse
            }

        case tokenizer.And, tokenizer.Or:
            leftNode, ok := stack.Pop().(*query.ConditionNode)
            if !ok {
                return nil, errFailedToParse
            }

            node := query.ConditionNode{
                Type: &p.current.Type,
                Left: leftNode,
            }
            stack.Push(&node)
        // Remaining cases ignored for simplicity.

        }
    }
    // Remaining code ignored for simplicity.
}
```

In *parseConditionTree*, it uses stack to store the parsing results. I only keep the case for logical operators (AND, OR) as well as the subquery. The subquery scenario is a bit tricky. To better understand it, let's first review an subquery example:

```shell
$ fsql
>>> SELECT all FROM . WHERE name IN (
...   SELECT name FROM ~/Desktop/files.bak/
... );
```

In Fsql, the subquery must follows the "IN (" string. *parseSubquery* first recursively calls *Run* method, and converts the subquery into a Query struct. Then, it executes the query by calling *Execute*, and stores the query results in condition struct.

```go
// parseSubquery parses a subquery by recursively evaluating it's condition(s).
// If the subquery contains references to aliases from the superquery, it's
// Subquery attribute is set. Otherwise, we evaluate it's Subquery and set
// it's Value to the result.
func (p *parser) parseSubquery(condition *query.Condition) error {
    q, err := Run(condition.Value.(string))
    if err != nil {
        return err
    }

    // If the subquery has aliases, we'll have to parse the subquery against
    // each file, so we don't do anything here.
    if len(q.SourceAliases) > 0 {
        condition.Subquery = q
        return nil
    }

    value := make(map[interface{}]bool, 0)
    workFunc := func(path string, info os.FileInfo, res map[string]interface{}) {
        for _, attr := range [...]string{"name", "size", "time", "mode"} {
            if q.HasAttribute(attr) {
                value[res[attr]] = true
                return
            }
        }
    }

    if err = q.Execute(workFunc); err != nil {
        return err
    }

    condition.Value = value
    condition.IsSubquery = false
    return nil
}
```

## The Execute Method
After the parsing phase, we have successfully converted the sql like query into a Query struct. Execute method is then used to walk through the filesystems and retrieve the files matching the given *Query* object.

```go
// parser/parser.go
// Execute runs the query by walking the full path of each source and
// evaluating the condition tree for each file. This method calls workFunc on
// each "successful" file.
func (q *Query) Execute(workFunc interface{}) error {
    seen := map[string]bool{}
    excluder := &regexpExclude{exclusions: q.Sources["exclude"]}

    for _, src := range q.Sources["include"] {
        // TODO: Improve our method of detecting if src is a glob pattern. This
        // currently doesn't support usage of square brackets, since the tokenizer
        // doesn't recognize these as part of a directory.
        //
        // Pattern reference: https://golang.org/pkg/path/filepath/#Match.
        if strings.ContainsAny(src, "*?") {
            // If src does _resemble_ a glob pattern, we find all matches and
            // evaluate the condition tree against each.
            matches, err := filepath.Glob(src)
            if err != nil {
                return err
            }

            for _, match := range matches {
                if err = filepath.Walk(match, q.walkFunc(seen, excluder, workFunc)); err != nil {
                    return err
                }
            }
            continue
        }

        if err := filepath.Walk(src, q.walkFunc(seen, excluder, workFunc)); err != nil {
            return err
        }
    }

    return nil
}
```

## Go Language Features
As a golang beginner, I found the following differences between golang and other languages.

1. The test file and the actual go file appears in the same directory.
No dedicated test folder. This is different from Python
2. Regarding error Handling, there are no exceptions. Each method *returns* the error along with other return values.
3. Error response is checked using the following pattern:

```go
if err := p.parseSubquery(condition); err != nil {
    return nil, err
}
```
