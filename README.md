# YAML support for the Go language

> **Note**
> 
> This is a fork of [gopkg.in/yaml.v3](https://github.com/go-yaml/yaml)
> which provides more specific error types when using `Unmarshal` to decode
> documents (see [example](#example-error-types)).

Introduction
------------

The yaml package enables Go programs to comfortably encode and decode YAML
values. It was developed within [Canonical](https://www.canonical.com) as
part of the [juju](https://juju.ubuntu.com) project, and is based on a
pure Go port of the well-known [libyaml](http://pyyaml.org/wiki/LibYAML)
C library to parse and generate YAML data quickly and reliably.

Compatibility
-------------

The yaml package supports most of YAML 1.2, but preserves some behavior
from 1.1 for backwards compatibility.

Specifically, as of v3 of the yaml package:

 - YAML 1.1 bools (_yes/no, on/off_) are supported as long as they are being
   decoded into a typed bool value. Otherwise they behave as a string. Booleans
   in YAML 1.2 are _true/false_ only.
 - Octals encode and decode as _0777_ per YAML 1.1, rather than _0o777_
   as specified in YAML 1.2, because most parsers still use the old format.
   Octals in the  _0o777_ format are supported though, so new files work.
 - Does not support base-60 floats. These are gone from YAML 1.2, and were
   actually never supported by this package as it's clearly a poor choice.

and offers backwards
compatibility with YAML 1.1 in some cases.
1.2, including support for
anchors, tags, map merging, etc. Multi-document unmarshalling is not yet
implemented, and base-60 floats from YAML 1.1 are purposefully not
supported since they're a poor design and are gone in YAML 1.2.

Installation and usage
----------------------

The import path for the package is *github.com/conduitio/yaml/v3*.

To install it, run:

    go get github.com/conduitio/yaml/v3

API documentation
-----------------

API documentation is hosted on pkg.go.dev:

  - [https://pkg.go.dev/github.com/conduitio/yaml](https://pkg.go.dev/github.com/conduitio/yaml)

API stability
-------------

The package API for yaml v3 will remain stable as described in [gopkg.in](https://gopkg.in).


License
-------

The yaml package is licensed under the MIT and Apache License 2.0 licenses.
Please see the LICENSE file for details.


Example (basic)
---------------

```Go
package main

import (
        "fmt"
        "log"

        "github.com/conduitio/yaml/v3"
)

var data = `
a: Easy!
b:
  c: 2
  d: [3, 4]
`

// Note: struct fields must be public in order for unmarshal to
// correctly populate the data.
type T struct {
        A string
        B struct {
                RenamedC int   `yaml:"c"`
                D        []int `yaml:",flow"`
        }
}

func main() {
        t := T{}
    
        err := yaml.Unmarshal([]byte(data), &t)
        if err != nil {
                log.Fatalf("error: %v", err)
        }
        fmt.Printf("--- t:\n%v\n\n", t)
    
        d, err := yaml.Marshal(&t)
        if err != nil {
                log.Fatalf("error: %v", err)
        }
        fmt.Printf("--- t dump:\n%s\n\n", string(d))
    
        m := make(map[interface{}]interface{})
    
        err = yaml.Unmarshal([]byte(data), &m)
        if err != nil {
                log.Fatalf("error: %v", err)
        }
        fmt.Printf("--- m:\n%v\n\n", m)
    
        d, err = yaml.Marshal(&m)
        if err != nil {
                log.Fatalf("error: %v", err)
        }
        fmt.Printf("--- m dump:\n%s\n\n", string(d))
}
```

This example will generate the following output:

```
--- t:
{Easy! {2 [3 4]}}

--- t dump:
a: Easy!
b:
  c: 2
  d: [3, 4]


--- m:
map[a:Easy! b:map[c:2 d:[3 4]]]

--- m dump:
a: Easy!
b:
  c: 2
  d:
  - 3
  - 4
```

Example (error types)
---------------------

```Go
package main

import (
	"fmt"

	"github.com/conduitio/yaml/v3"
)


var data = `
a: Easy!
b:
  c: abc   # expected int
  d: [3, 4]
foo: unknown
bar:
  Duplicate: first
  Duplicate: again
`

type T struct {
	A string
	B struct {
		RenamedC int   `yaml:"c"`
		D        []int `yaml:",flow"`
	}
	Bar struct {
		Duplicate string
	}
}

func main() {
	var t T

	dec := yaml.NewDecoder(bytes.NewBufferString(data))
	dec.KnownFields(true) // enable *yaml.UnknownFieldError

	err := dec.Decode(&t)
	if err != nil {
		fmt.Println("oops, something went wrong")
		if terr, ok := err.(*yaml.TypeError); ok {
			for _,unmarshalErr := range terr.Errors {
				fmt.Printf("  %T: line %d col %d: %v\n", unmarshalErr, unmarshalErr.Line(), unmarshalErr.Column(), unmarshalErr.Error())
			}
		}
	}
	d, _ := yaml.Marshal(&t)
	fmt.Printf("--- t dump:\n%s\n\n", string(d))
}
```

Produces:

```
oops, something went wrong
  *yaml.InvalidTypeError: line 4 col 6: cannot unmarshal !!str `abc` into int
  *yaml.UnknownFieldError: line 6 col 1: field foo not found in type yaml_test.T
  *yaml.DuplicateMappingKeyError: line 9 col 3: mapping key "Duplicate" already defined at line 8
--- t dump:
a: Easy!
b:
    c: 0
    d: [3, 4]
bar:
    duplicate: ""
```
