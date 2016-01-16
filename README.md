# Neugram is data processing system (including a scripting language).

Neugram is a numerical processing library and language. Because in
practice numerical processing is all about finding an moving data
around, and often doing parts of the processing elsewhere (for example,
inside an SQL query's WHERE clause), the library includes a small
interpreted programming language. The langauge is starting as a subset
of Go with a couple of key numerical features added.

The key data structure is the table. A table has a small (known)
number of named columns and many rows.

Very imporant: it's easy to import Go packages and call Go from inside
Neugram. Serious work should be implemented in Go, Neugram is the
numerical glue.

# Files and Packages

Files end in *.ng*.

A package is a single file.

A file is a sequence of statements.

The first statement in a file is "package *nameoffile.ng*"

A package can be imported with "import path/to/nameoffile.ng".
In a file, imports must come directly after package. The interpreter
allows mid-file imports.

A program starts with "package main".

We borrow GOPATH.

# Types

- Allow the definition of unused Go types, but no arithmetic on unsigned ints.

## Concrete types:

```
	string
	integer       (like a *big.Int)
	int64
	float         (like a *big.Float)
	float64
	float32
	complex128
	struct
```

## Abstract types

Neugram has an abstract type interface{} that behaves just like Go's
interface{}. It can have methods and hold any concrete type.

In addition, Neugram has a family of type aliases for certain interface
types that provide additional syntax.

### Maps

The abstract type alias ```map[interface{}]interface{}``` is equivalent to:

```
	interface {
		Get(key interface{}) (interface{}, error)
		Set(key, value interface{})
		Delete(key interface{})
		Range() interface { Next() (interface{}, interface{}, error) }
		ZeroKeyValue() (interface{}, interface{})
	}
```

Get returns NotFound if the key is not in the map.

A map may optionally implement any of:

```
	interface { Set(key, value interface{}) }
	interface { Delete(key interface{}) }
	interface { Range() interface { Next() (k, v interface{}, err error) } }
```

When the end of a range is reached, Next returns EndOfRange.

The key and value of a map can be specialized to any comparable T and U.
If a map has a specialized key T or value U, a zero value of that type
is returned by ZeroKeyValue.

Thus a map[T]U can be converted to map[interface{}]U, map[T]interface{},
map[interface{}]interface{}.

When Go maps are passed to Neugram they are represented as a
Neugram map. A Neugram map can only be used as a Go map if it
is dynamically determined to be stored by an underlying Go map.
(TODO: provide a common library function to convert maps if
necessary.)

A Neugram implementation may also use the Go type map[T]U as an
implementation of the Neugram map[T]U.

Maps in Neugram use the same syntax as Go maps.

Constructing a map with a composite literal uses a growable im-memory
hash map as the implementation.

### Tables

A table is a multi-dimensional list with a specified value type.

A table ```[]interface{}``` is equivalent to:

```
	interface {
		Get(index ...int) interface{}
		Dim() int
		Len(dim int) int
		ZeroValue() interface{}
	}
```

A table may optionally implement any of:

```
	interface { Set(value interface{}, index ...int) }
	interface { Slice(index ...int) []interface{} }
	interface { Col() []string }               // requires Dim()==2
	interface { SliceCol(name ...string) []interface{} }
```

A table can be specialized to ```[]T``` if a zero value of the type T
is returned by ZeroValue. Any []T can be cast to []interface{}.

Constructing a table with a composite literal uses an in-memory
contiguous array as the implementation.

TODO: append, multi-dimensional semantics are complex.

### Table syntax

TODO, this stuff is hard.

## A little generic: num

Functions and types can be parameterized over a single type parameter.
The type parameter can only resolve to numeric scalar types and must
have the name **num**. Its appearence anywhere in the type declaration
means the function is generic:

```
	min := func(x, y num) num {
		if x < y {
			return x
		}
		return y
	}
	print(min(float64(4.5), float64(4.1))) // prints 4.1
	print(min(integer(4), integer(3)))     // prints 3
	print(min(integer(4), float64(3.2)))   // compile error
```

When declaring a variable using type inference from a constant,
the variable adopts the type of num.

```
	add2 := func(x num) num {
		two := 2
		return x + two
	}
	add2(float32(4.5)) // two was a float32
	add2(int64(4))     // two was an int64
```

If in a scope where num is not assigned a type, the default numeric
type is float64.

The type []num can be used to represent a matrix whose type matches
the type parameter.

```
	func fill(mat []num, v num) {
		w, h := len(mat)
		for x := 0; x < w; x++ {
			for y := 0; y < h; y++ {
				mat[y|x] = v
			}
		}
	}
```

## Methods

Methods are tricky in a forward only statement-based language like this.
Current thought is a second keyword like type that ties a set of methods
to a type name.

```
methodik AnInt integer {
	func (a) f() integer { return a }
}

methodik T *struct{
	x integer
	y []int64
} {
	func (a) f(x integer) integer {
		return a.x
	}
}
```

Down side is this is a second way to make a type name.

TODO: embedding?

# importgo

A valid program:

```
importgo "fmt"
fmt.Printf("hello, world")
```

There's a bit of a song and dance to get here.

First off, the typechecker should rely on nothing but the export data
of the Go package (as represented by a *types.Package object).
This is because the typechecker will be used in a neugram -> Go
compiler later down the road, and we should be able to compile without
loading the package into the compiler.

Second, the evaluator should eventually use -buildmode=plugin:

https://docs.google.com/document/d/1nr-TQHw_er6GOQRsF6T43GGhFDelrAP0NqSS_00RgZQ/preview

The plugin API will allow loading package functions and global
variables by a method, ```Lookup(name string) (interface{}, error)```.
We want access to types as well, so for any Go package we want to
load, we need to generate a Go package that wraps it, and exposes a
global variable containing the zero value of the exported types:

```
package wrap_bytes

import "bytes"

var ExportData string // loadable as *types.Package for "bytes"

var Exports = map[string]interface{}{
	...,
	"Buffer": *bytes.Buffer(nil),
}
```

So the evaluator will:

1. Attempt to load this compiled wrapper package. If it doesn't exist,
generate the code, compile it, and load it as a plugin.

1. Provide ExportData contents to the numgrad typechecker.

1. Lookup Exports and Use the reflect package on the values in Exports
to create Go types and call Go functions and methods.

# Syntax

Almost entirely derived from Go.

New keywords: num (reserved). Doesn't have to be keywords,
but I'm sick of the gotchas that arise from letting people write
```type int int16```.

TODO: literal syntax for tables is giving me a headache. For now, using the composite literal syntax for the fake struct record:

Must do better, but I really hate syntax.

*Problem:* this syntax is row-major, but our slicing and reasoning is column-major.

## Tables

The Go type frame.Frame has an innocently named optional method Slice
that gets a ton of fun syntax:

Given x:
```
 Col0 Col1 Col2
0 0.0  0.1  0.2
1 1.0  1.1  1.2
2 2.0  2.1  2.2
```

or

```
ident3x3 := []float64{
	{1, 0, 0},
	{0, 1, 0},
	{0, 0, 1},
}

presidents := []val{
	{|"ID", "Name", "Term1", "Term2"|},
	{1, "George Washington", 1789, 1792},
	{2, "John Adams", 1797, 0},
	{3, "Thomas Jefferson", 1800, 1804},
	{4, "James Madison", 1808, 1812},
}

presidents["Name"] == presidents[1] == []val{
	{|"Name"|},
	{"George Washington"},
	{"John Adams"},
	{"Thomas Jefferson"},
	{"James Madison"},
}

x = []num{
	{|"Col0", "Col1", "Col2"|},
	{0.0, 0.1, 0.2},
	{1.0, 1.1, 1.2},
	{2.0, 2.1  2.2},
}

x[1] == x["Col1"] == []num{
	{|"Col1"|},
	{0.1},
	{1.1},
	{2.1},
}

x[,2] == []num{
	{|"Col0", "Col1", "Col2"|},
	{2.0, 2.1  2.2},
}

x[0|2] == x["Col0"|"Col2"] == []num{
	{|"Col0", "Col2"|},
	{0.0, 0.2},
	{1.0, 1.2},
	{2.0, 2.2},
}

x[0:1] == x[0|1] == x["Col0"|"Col1"]

x[1,0:1] == []num{
	{|"Col1"|},
	{0.1},
	{1.1},
}

x[1,1] == 1.1 // all slicing variants return a table, except this one
```
