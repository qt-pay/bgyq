## [LGTM] Golang Reflect 101

Go is a static language with well reflection support. The remaining of this article will explain the reflection functionalities provided in the `reflect` standard package.

It is very helpful to read the [overview of Go type system](https://go101.org/article/type-system-overview.html) and [interfaces in Go](https://go101.org/article/interface.html) articles before reading the remaining of the current article.

### Overview of Go Reflection

Go reflection brings many dynamic functionalities to Go programming. Many standard code packages, such as the `fmt` and `encoding` packages, heavily rely on the reflection functionalities.

We can inspect Go values through the values of the `Type` and `Value` types defined in the `reflect` standard package. The remaining of this article will show some examples on how to use values of the two types.

One of the Go reflection design goals is any non-reflection operation should be also possible to be applied through the reflection ways. For all kinds of reasons, this goal is not 100 percent achieved. However, most non-reflection operations can be applied through the reflection ways now. On the other hand, through the reflection ways, we can do some operations which are impossible to be achieved through non-reflection ways. The operations which can't and can only be achieved through the reflection ways will be mentioned in the following sections.

### The `reflect.Type` Type and Values

In Go, we can create a `reflect.Type` value from an arbitrary non-interface value by calling the `reflect.TypeOf` function. The result `reflect.Type` value represents the type of the non-interface value. Surely, we can also pass an interface value to a `reflect.TypeOf` function call, but the call will return a `reflect.Type` value which represents the dynamic type of the interface value. In fact, the `reflect.TypeOf` function has only one parameter of type `interface{}` and always returns a `reflect.Type` value which represents the dynamic type of the only interface parameter. Then how to get a `reflect.Type` value which represents an interface type? We must use indirect ways which will be introduced below to achieve this goal.

```go
// Type is the representation of a Go type.
// Two Type values are equal if they represent identical types.
type Type interface {
	...
   
}

// TypeOf returns the reflection Type that represents the dynamic type of i.
// If i is a nil interface value, TypeOf returns nil.
func TypeOf(i interface{}) Type {
	eface := *(*emptyInterface)(unsafe.Pointer(&i))
	return toType(eface.typ)
}
```



The `reflect.Type` type is an interface type. It [specifies several methods](https://golang.org/pkg/reflect/#Type). We can call these methods to inspect the information of the type represented by a `reflect.Type` receiver value. Some of these methods apply for all [kinds of types](https://golang.org/pkg/reflect/#Kind), some of them are one kind or several kinds specific. Please read the documentation of each method for details. Calling one of the methods through an improper `reflect.Type` receiver value will produce a panic.

An example:

```go
package main

import "fmt"
import "reflect"

func main() {
	type A = [16]int16
	var c <-chan map[A][]byte
	tc := reflect.TypeOf(c)
	fmt.Println(tc.Kind())    // chan
	fmt.Println(tc.ChanDir()) // <-chan
	tm := tc.Elem()
	ta, tb := tm.Key(), tm.Elem()
	// The next line prints: map array slice
	fmt.Println(tm.Kind(), ta.Kind(), tb.Kind())
	tx, ty := ta.Elem(), tb.Elem()

	// byte is an alias of uint8
	fmt.Println(tx.Kind(), ty.Kind()) // int16 uint8
	fmt.Println(tx.Bits(), ty.Bits()) // 16 8
	fmt.Println(tx.ConvertibleTo(ty)) // true
	fmt.Println(tb.ConvertibleTo(ta)) // false

	// Slice and map types are incomparable.
	fmt.Println(tb.Comparable()) // false
	fmt.Println(tm.Comparable()) // false
	fmt.Println(ta.Comparable()) // true
	fmt.Println(tc.Comparable()) // true
}
```

There are [26 kinds of types](https://golang.org/pkg/reflect/#Kind) in Go.

In the above example, we use the method `Elem` to get the element types of some container types (a channel type, a map type, a slice type and an array type). In fact, we can also use this method to get the base type of a pointer type. For example,

```go
package main

import "fmt"
import "reflect"

type T []interface{m()}
func (T) m() {}

func main() {
	tp := reflect.TypeOf(new(interface{}))
	tt := reflect.TypeOf(T{})
	fmt.Println(tp.Kind(), tt.Kind()) // ptr slice

	// Get two interface Types indirectly.
	ti, tim := tp.Elem(), tt.Elem()
	// The next line prints: interface interface
	fmt.Println(ti.Kind(), tim.Kind())

	fmt.Println(tt.Implements(tim))  // true
	fmt.Println(tp.Implements(tim))  // false
	fmt.Println(tim.Implements(tim)) // true

	// All types implement any blank interface type.
	fmt.Println(tp.Implements(ti))  // true
	fmt.Println(tt.Implements(ti))  // true
	fmt.Println(tim.Implements(ti)) // true
	fmt.Println(ti.Implements(ti))  // true
}
```

The above example also shows how to (indirectly) get a `reflect.Type` value which represents an interface type.

We can get all of the field types (of a struct type) and the method information of a type through reflection. We can also get the parameter and result type information of a function type through reflection.

```go
package main

import "fmt"
import "reflect"

type F func(string, int) bool
func (f F) m(s string) bool {
	return f(s, 32)
}
func (f F) M() {}

type I interface{m(s string) bool; M()}

func main() {
	var x struct {
		F F
		i I
	}
	tx := reflect.TypeOf(x)
	fmt.Println(tx.Kind())        // struct
	fmt.Println(tx.NumField())    // 2
	fmt.Println(tx.Field(1).Name) // i
	// Package path is an intrinsic property of
	// non-exported selectors (fields or methods).
	fmt.Println(tx.Field(0).PkgPath) // 
	fmt.Println(tx.Field(1).PkgPath) // main

	tf, ti := tx.Field(0).Type, tx.Field(1).Type
	fmt.Println(tf.Kind())               // func
	fmt.Println(tf.IsVariadic())         // false
	fmt.Println(tf.NumIn(), tf.NumOut()) // 2 1
	t0, t1, t2 := tf.In(0), tf.In(1), tf.Out(0)
	// The next line prints: string int bool
	fmt.Println(t0.Kind(), t1.Kind(), t2.Kind())

	fmt.Println(tf.NumMethod(), ti.NumMethod()) // 1 2
	fmt.Println(tf.Method(0).Name)              // M
	fmt.Println(ti.Method(1).Name)              // m
	_, ok1 := tf.MethodByName("m")
	_, ok2 := ti.MethodByName("m")
	fmt.Println(ok1, ok2) // false true
}
```

From the above example, we could find that,

1. for non-interface types, the `reflect.Type.NumMethod` only returns the number of exported methods (including implicitly declared ones) of a type. We are unable to get the information of a non-exported method by using the `reflect.Type.MethodByName` method. For interface types, the limits don't exist (the fact was not mentioned in the docs of the two methods before Go 1.16). Such situation also applies to the corresponding methods of the `reflect.Value` type introduced in the next section.
2. although a `reflect.Type.NumField` method call returns the number of all fields (including non-exported ones) of a struct type, it is [not a good idea](https://golang.org/pkg/reflect/#pkg-note-BUG) to use the `reflect.Type.FieldByName` method to get the information of a non-exported field.

We may [inspect struct field tags through reflection](https://golang.org/pkg/reflect/#StructTag). The types of struct field tags are `reflect.StructTag`, which has two methods, `Get` and `Lookup`, to inspect the key-value pairs specified in field tags. An example of inspecting struct field tags:

```go
package main

import "fmt"
import "reflect"

type T struct {
	X    int  `max:"99" min:"0" default:"0"`
	Y, Z bool `optional:"yes"`
}

func main() {
	t := reflect.TypeOf(T{})
	x := t.Field(0).Tag
	y := t.Field(1).Tag
	z := t.Field(2).Tag
	fmt.Println(reflect.TypeOf(x)) // reflect.StructTag
	// v is a string
	v, present := x.Lookup("max")     
	fmt.Println(len(v), present)      // 2 true
	fmt.Println(x.Get("max"))         // 99
	fmt.Println(x.Lookup("optional")) //  false
	fmt.Println(y.Lookup("optional")) // yes true
	fmt.Println(z.Lookup("optional")) // yes true
}
```

Note,

- tag keys may not contain space (Unicode value 32), quote (Unicode value 34) and colon (Unicode value 58) characters.
- to form a valid key-value pair, no space characters are allowed to follow the semicolon in the supposed key-value pair. So
  ``optional: "yes"`` doesn't form key-value pairs.
- space characters in tag values are important (not be ignored). So
  ``json:"author, omitempty"``,
  ``json:" author,omitempty"`` and
  ``json:"author,omitempty"`` are different from each other.
- each struct field tag should present as a single line to be wholly meaningful.

Beside the `reflect.TypeOf` function, we can also use some other functions in the `reflect` standard package to create `reflect.Type` values which represent some unnamed composite types.

```go
package main

import "fmt"
import "reflect"

func main() {
	ta := reflect.ArrayOf(5, reflect.TypeOf(123))
	fmt.Println(ta) // [5]int
	tc := reflect.ChanOf(reflect.SendDir, ta)
	fmt.Println(tc) // chan<- [5]int
	tp := reflect.PtrTo(ta)
	fmt.Println(tp) // *[5]int
	ts := reflect.SliceOf(tp)
	fmt.Println(ts) // []*[5]int
	tm := reflect.MapOf(ta, tc)
	fmt.Println(tm) // map[[5]int]chan<- [5]int
	tf := reflect.FuncOf([]reflect.Type{ta},
				[]reflect.Type{tp, tc}, false)
	fmt.Println(tf) // func([5]int) (*[5]int, chan<- [5]int)
	tt := reflect.StructOf([]reflect.StructField{
		{Name: "Age", Type: reflect.TypeOf("abc")},
	})
	fmt.Println(tt)            // struct { Age string }
	fmt.Println(tt.NumField()) // 1
}
```



There are more `reflect.Type` methods which are not used in above examples, please read the `reflect` package documentation for their usages.

Note, up to now (Go 1.19), there are no ways to create interface types through reflection. This is a known limitation of Go reflection.

Another limitation is, although we can create a struct type embedding other types as anonymous fields through reflection, the struct type may or may not obtain the methods of the embedded types, and creating a struct type with anonymous fields even might panic at run time. In other words, the behavior of creating struct types with anonymous fields is partially compiler dependent.

The third limitation is we can't declare new types through reflection.

### The `reflect.Value` Type and Values

Similarly, we can create a `reflect.Value` value from an arbitrary non-interface value by calling the `reflect.ValueOf` function. The result `reflect.Value` value represents the non-interface value. Same as the `reflect.TypeOf` function, the `reflect.ValueOf` function also has only one parameter of type `interface{}`. When an interface argument is passed to a `reflect.ValueOf` function call, the call will return a `reflect.Value` value which represents the dynamic value of the interface argument. To get a `reflect.Value` value which represents an interface value, we must use indirect ways which will be introduced below to achieve this goal.

```GO
// Value is the reflection interface to a Go value.
type Value struct {
    
}


// Indirect returns the value that v points to.
// If v is a nil pointer, Indirect returns a zero Value.
// If v is not a pointer, Indirect returns v.
func Indirect(v Value) Value {
	if v.Kind() != Ptr {
		return v
	}
	return v.Elem()
}

// ValueOf returns a new Value initialized to the concrete value
// stored in the interface i. ValueOf(nil) returns the zero Value.
func ValueOf(i interface{}) Value {
	if i == nil {
		return Value{}
	}

	escapes(i)

	return unpackEface(i)
}

```



The value represented by a `reflect.Value` value `v` is often called the underlying value of `v`.

There are [plenty of methods](https://golang.org/pkg/reflect/) declared for the `reflect.Value` type. We can call these methods to inspect the information of (and manipulate) the underlying value of a `reflect.Value` receiver value. Some of these methods apply for all kinds of values, some of them are one kind or several kinds specific. Please read the `reflect` standard package documentation for details. Calling a kind-specific method with an improper `reflect.Value` receiver value will produce a panic.

The `CanSet` method of a `reflect.Value` value returns whether or not the underlying value of the `reflect.Value` value is modifiable (can be assigned to). If the Go value is modifiable, we can call the `Set` method of the corresponding `reflect.Value` value to modify the Go value. Note, the `reflect.Value` values returned directly by `reflect.ValueOf` function calls are always read-only.

An example:

```go
package main

import "fmt"
import "reflect"

func main() {
	n := 123
	p := &n
	vp := reflect.ValueOf(p)
	fmt.Println(vp.CanSet(), vp.CanAddr()) // false false
	vn := vp.Elem() // get the value referenced by vp
	fmt.Println(vn.CanSet(), vn.CanAddr()) // true true
	vn.Set(reflect.ValueOf(789)) // <=> vn.SetInt(789)
	fmt.Println(n)               // 789
}
```



Non-exported fields of struct values can't be modified through reflections.

```go
package main

import "fmt"
import "reflect"

func main() {
	var s struct {
		X interface{} // an exported field
		y interface{} // a non-exported field
	}
	vp := reflect.ValueOf(&s)
	// If vp represents a pointer. the following
	// line is equivalent to "vs := vp.Elem()".
	vs := reflect.Indirect(vp)
	// vx and vy both represent interface values.
	vx, vy := vs.Field(0), vs.Field(1)
	fmt.Println(vx.CanSet(), vx.CanAddr()) // true true
	// vy is addressable but not modifiable.
	fmt.Println(vy.CanSet(), vy.CanAddr()) // false true
	vb := reflect.ValueOf(123)
	vx.Set(vb)     // okay, for vx is modifiable
	// vy.Set(vb)  // will panic, for vy is unmodifiable
	fmt.Println(s) // {123 }
	fmt.Println(vx.IsNil(), vy.IsNil()) // false true
}
```



From the above two examples, we can learn that there are two ways to get a `reflect.Value` value whose underlying value is referenced by the underlying value (a pointer value) of another `reflect.Value` value.

1. One way is by calling the `Elem` method of a `reflect.Value` value which represents the pointer value.
2. The other way is to pass a `reflect.Value` value which represents the pointer value to a `reflect.Indirect` function call. (If the argument passed to a `reflect.Indirect` function call doesn't represent a pointer value, then the call returns a copy of the argument.)

Note, the `reflect.Value.Elem` method can be also used to get a `reflect.Value` value which represents the dynamic value of an interface value. For example,

```go
package main

import "fmt"
import "reflect"

func main() {
	var z = 123
	var y = &z
	var x interface{} = y
	v := reflect.ValueOf(&x)
	vx := v.Elem()
	vy := vx.Elem()
	vz := vy.Elem()
	vz.Set(reflect.ValueOf(789))
	fmt.Println(z) // 789
}
```



The `reflect` standard package also declares some `reflect.Value` related functions. Each of these functions corresponds to a built-in function or a non-reflection functionality, The following example demonstrates how to bind a (kind of) custom generic function to different function values.

```go
package main

import "fmt"
import "reflect"

func InvertSlice(args []reflect.Value) []reflect.Value {
	inSlice, n := args[0], args[0].Len()
	outSlice := reflect.MakeSlice(inSlice.Type(), 0, n)
	for i := n-1; i >= 0; i-- {
		element := inSlice.Index(i)
		outSlice = reflect.Append(outSlice, element)
	}
	return []reflect.Value{outSlice}
}

func Bind(p interface{},
		f func ([]reflect.Value) []reflect.Value) {
	// invert represents a function value.
	invert := reflect.ValueOf(p).Elem()
	invert.Set(reflect.MakeFunc(invert.Type(), f))
}

func main() {
	var invertInts func([]int) []int
	Bind(&invertInts, InvertSlice)
	fmt.Println(invertInts([]int{2, 3, 5})) // [5 3 2]

	var invertStrs func([]string) []string
	Bind(&invertStrs, InvertSlice)
	fmt.Println(invertStrs([]string{"Go", "C"})) // [C Go]
}
```



If the underlying value of a `reflect.Value` is a function value, then we can call the `Call` method of the `reflect.Value` to call the underlying function.

```go
package main

import "fmt"
import "reflect"

type T struct {
	A, b int
}

func (t T) AddSubThenScale(n int) (int, int) {
	return n * (t.A + t.b), n * (t.A - t.b)
}

func main() {
	t := T{5, 2}
	vt := reflect.ValueOf(t)
	vm := vt.MethodByName("AddSubThenScale")
	results := vm.Call([]reflect.Value{reflect.ValueOf(3)})
	fmt.Println(results[0].Int(), results[1].Int()) // 21 9

	neg := func(x int) int {
		return -x
	}
	vf := reflect.ValueOf(neg)
	fmt.Println(vf.Call(results[:1])[0].Int()) // -21
	fmt.Println(vf.Call([]reflect.Value{
		vt.FieldByName("A"), // panic on changing to "b"
	})[0].Int()) // -5
}
```

Please note that, non-exported fields shouldn't be used as arguments of reflection calls. If the line `vt.FieldByName("A")` in the above example is replaced with `vt.FieldByName("b")`, a panic will occur.

A reflection example for map values.

```go
package main

import "fmt"
import "reflect"

func main() {
	valueOf := reflect.ValueOf
	m := map[string]int{"Unix": 1973, "Windows": 1985}
	v := valueOf(m)
	// A zero second Value argument means to delete an entry.
	v.SetMapIndex(valueOf("Windows"), reflect.Value{})
	v.SetMapIndex(valueOf("Linux"), valueOf(1991))
	for i := v.MapRange(); i.Next(); {
		fmt.Println(i.Key(), "\t:", i.Value())
	}
}
```

Please note that, the `MapRange` method is supported since Go 1.12.

A reflection example for channel values.

```go
package main

import "fmt"
import "reflect"

func main() {
	c := make(chan string, 2)
	vc := reflect.ValueOf(c)
	vc.Send(reflect.ValueOf("C"))
	succeeded := vc.TrySend(reflect.ValueOf("Go"))
	fmt.Println(succeeded) // true
	succeeded = vc.TrySend(reflect.ValueOf("C++"))
	fmt.Println(succeeded) // false
	fmt.Println(vc.Len(), vc.Cap()) // 2 2
	vs, succeeded := vc.TryRecv()
	fmt.Println(vs.String(), succeeded) // C true
	vs, sentBeforeClosed := vc.Recv()
	fmt.Println(vs.String(), sentBeforeClosed) // Go true
	vs, succeeded = vc.TryRecv()
	fmt.Println(vs.String()) // 
	fmt.Println(succeeded)   // false
}
```

The `TrySend` and `TryRecv` methods correspond to one-case-one-default [`select` control flow code blocks](https://go101.org/article/channel.html#select).

We can use the `reflect.Select` function to simulate a `select` code block with dynamic number of `case` branches at run time.

```go
package main

import "fmt"
import "reflect"

func main() {
	c := make(chan int, 1)
	vc := reflect.ValueOf(c)
	succeeded := vc.TrySend(reflect.ValueOf(123))
	fmt.Println(succeeded, vc.Len(), vc.Cap()) // true 1 1

	vSend, vZero := reflect.ValueOf(789), reflect.Value{}
	branches := []reflect.SelectCase{
		{Dir: reflect.SelectDefault, Chan: vZero, Send: vZero},
		{Dir: reflect.SelectRecv, Chan: vc, Send: vZero},
		{Dir: reflect.SelectSend, Chan: vc, Send: vSend},
	}
	selIndex, vRecv, sentBeforeClosed := reflect.Select(branches)
	fmt.Println(selIndex)         // 1
	fmt.Println(sentBeforeClosed) // true
	fmt.Println(vRecv.Int())      // 123
	vc.Close()
	// Remove the send case branch this time,
	// for it may cause panic.
	selIndex, _, sentBeforeClosed = reflect.Select(branches[:2])
	fmt.Println(selIndex, sentBeforeClosed) // 1 false
}
```



The respective underlying values of some `reflect.Value` values may be nothing. For example, zero `reflect.Value` values.

```go
package main

import "reflect"
import "fmt"

func main() {
	var z reflect.Value // a zero Value value
	fmt.Println(z)      // 
	v := reflect.ValueOf((*int)(nil)).Elem()
	fmt.Println(v)      // 
	fmt.Println(v == z) // true
	var i = reflect.ValueOf([]interface{}{nil}).Index(0)
	fmt.Println(i)             // 
	fmt.Println(i.Elem() == z) // true
	fmt.Println(i.Elem())      // 
}
```



For a Go value, we can use the `reflect.ValueOf` function to create a `reflect.Value` value representing the Go value, through the help of `interface{}`. The inverse process in similar, we can call the `Interface` method of a `reflect.Value` value to get an `interface{}` value, then type assert on the `interface{}` value to get the Go value represented by (a.k.a., the underlying value of ) the `reflect.Value` value. But please note that, calling the `Interface` method of a `reflect.Value` value which represents a non-exported field causes a panic.

```go
package main

import (
	"fmt"
	"reflect"
	"time"
)

func main() {
	vx := reflect.ValueOf(123)
	vy := reflect.ValueOf("abc")
	vz := reflect.ValueOf([]bool{false, true})
	vt := reflect.ValueOf(time.Time{})

	x := vx.Interface().(int)
	y := vy.Interface().(string)
	z := vz.Interface().([]bool)
	m := vt.MethodByName("IsZero").Interface().(func() bool)
	fmt.Println(x, y, z, m()) // 123 abc [false true] true

	type T struct {x int}
	t := &T{3}
	v := reflect.ValueOf(t).Elem().Field(0)
	fmt.Println(v)             // 3
	fmt.Println(v.Interface()) // panic
}
```

The method `reflect.Value.IsZero` was introduced in Go 1.13. It is used to check whether or not the underlying value of a `reflect.Value` value is a zero value.

Since Go 1.17, [a slice may be converted to an array pointer](https://go101.org/article/container.html#slice-to-array-pointer). However, such a conversion might panic if the length of the pointer base array type is too large. The method `reflect.Value.CanConvert(T reflect.Type)` introduced in Go 1.17 is used to check whether or not a conversion will success.

An example using the `CanConvert` method:

```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	s := reflect.ValueOf([]int{1, 2, 3, 4, 5})
	ts := s.Type()
	t1 := reflect.TypeOf(&[5]int{})
	t2 := reflect.TypeOf(&[6]int{})
	fmt.Println(ts.ConvertibleTo(t1)) // true
	fmt.Println(ts.ConvertibleTo(t2)) // true
	fmt.Println(s.CanConvert(t1))     // true
	fmt.Println(s.CanConvert(t2))     // false
}
```



There are more `reflect.Value` related functions and methods which are not used in above examples, please read the `reflect` package documentation for their usages. In addition, please note that there are [some reflection](https://go101.org/article/details.html#reflect-deep-equal) related [details](https://go101.org/article/details.html#reflect-value-bytes) mentioned in [Go details 101](https://go101.org/article/details.html).