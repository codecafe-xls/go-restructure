[![GoDoc](https://godoc.org/github.com/alexflint/go-restructure?status.svg)](https://godoc.org/github.com/alexflint/go-restructure)
[![Build Status](https://travis-ci.org/alexflint/go-restructure.svg?branch=master)](https://travis-ci.org/alexflint/go-restructure)

## Match regular expressions into struct fields

```shell
go get github.com/alexflint/go-restructure
```

This package allows you to express regular expressions by defining a struct, and then capture matched sub-expressions into struct fields. Here is a very simple email address parser:

```go
import "github.com/alexflint/go-restructure"

type EmailAddress struct {
	_    struct{} `^`
	User string   `\w+`
	_    struct{} `@`
	Host string   `[^@]+`
	_    struct{} `$`
}

func main() {
	var addr EmailAddress
	restructure.Find(&addr, "joe@example.com")
	fmt.Println(addr.User) // prints "joe"
	fmt.Println(addr.Host) // prints "example.com"
}
```
(Note that the above is far too simplistic to be used as a serious email address validator.)

The regular expression that was executed was the concatenation of the struct tags:

```
^(\w+)@([^@]+)$
```

The first submatch was inserted into the `User` field and the second into the `Host` field.

You may also use the `regexp:` tag key, but keep in mind that you must escape quotes and backslashes:

```go
type EmailAddress struct {
	_    string `regexp:"^"`
	User string `regexp:"\\w+"`
	_    string `regexp:"@"`
	Host string `regexp:"[^@]+"`
	_    string `regexp:"$"`
}
```

### Nested Structs

Here is a slightly more sophisticated email address parser that uses nested structs:

```go
type Hostname struct {
	Domain string   `\w+`
	_      struct{} `\.`
	TLD    string   `\w+`
}

type EmailAddress struct {
	_    struct{} `^`
	User string   `[a-zA-Z0-9._%+-]+`
	_    struct{} `@`
	Host *Hostname
	_    struct{} `$`
}

func main() {
	var addr EmailAddress
	success, _ := restructure.Find(&addr, "joe@example.com")
	if success {
		fmt.Println(addr.User)        // prints "joe"
		fmt.Println(addr.Host.Domain) // prints "example"
		fmt.Println(addr.Host.TLD)    // prints "com"
	}
}
```

Compare this to using the standard library `regexp.FindStringSubmatchIndex` directly:

```go
func main() {
	content := "joe@example.com"
	expr := regexp.MustCompile(`^([a-zA-Z0-9._%+-]+)@((\w+)\.(\w+))$`)
	indices := expr.FindStringSubmatchIndex(content)
	if len(indices) > 0 {
		userBegin, userEnd := indices[2], indices[3]
		var user string
		if userBegin != -1 && userEnd != -1 {
			user = content[userBegin:userEnd]
		}

		domainBegin, domainEnd := indices[6], indices[7]
		var domain string
		if domainBegin != -1 && domainEnd != -1 {
			domain = content[domainBegin:domainEnd]
		}

		tldBegin, tldEnd := indices[8], indices[9]
		var tld string
		if tldBegin != -1 && tldEnd != -1 {
			tld = content[tldBegin:tldEnd]
		}

		fmt.Println(user)   // prints "joe"
		fmt.Println(domain) // prints "example"
		fmt.Println(tld)    // prints "com"
	}
}
```

### Optional fields

When nesting one struct within another, you can make the nested struct optional by marking it with `?`. The following example parses floating point numbers with optional sign and exponent:

```go
// Matches "123", "1.23", "1.23e-4", "-12.3E+5", ".123"
type Float struct {
	Sign     *Sign     `?`      // sign is optional
	Whole    string    `[0-9]*`
	Period   struct{}  `\.?`
	Frac     string    `[0-9]+`
	Exponent *Exponent `?`      // exponent is optional
}

// Matches "e+4", "E6", "e-03"
type Exponent struct {
	_    struct{} `[eE]`
	Sign *Sign    `?`         // sign is optional
	Num  string   `[0-9]+`
}

// Matches "+" or "-"
type Sign struct {
	Ch string `[+-]`
}
```

When an optional sub-struct is not matched, it will be set to nil:

```javascript
"1.23" -> {
  "Sign": nil,
  "Whole": "1",
  "Frac": "23",
  "Exponent": nil
}

"1.23e+45" -> {
  "Sign": nil,
  "Whole": "1",
  "Frac": "23",
  "Exponent": {
    "Sign": {
      "Ch": "+"
    },
    "Num": "45"
  }
}
```

### Getting begin and end positions for submatches

To get the begin and end position of submatches, use the `restructure.Submatch` struct in place of `string`:

Here is an example of matching python imports such as `import foo as bar`:

```go
type Import struct {
	_       struct{}             `^import\s+`
	Package restructure.Submatch `\w+`
	_       struct{}             `\s+as\s+`
	Alias   restructure.Submatch `\w+`
}

var importRegexp = restructure.MustCompile(Import{}, restructure.Options{})

func main() {
	var imp Import
	importRegexp.Find(&imp, "import foo as bar")
	fmt.Printf("IMPORT %s (bytes %d...%d)\n", imp.Package.String(), imp.Package.Begin, imp.Package.End)
	fmt.Printf("    AS %s (bytes %d...%d)\n", imp.Alias.String(), imp.Alias.Begin, imp.Alias.End)
}
```
Output:
```
IMPORT foo (bytes 7...10)
    AS bar (bytes 14...17)
```
