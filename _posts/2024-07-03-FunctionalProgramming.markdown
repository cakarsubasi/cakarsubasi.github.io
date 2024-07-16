---
layout: post
title: "Functional Programming 1"
date: 2024-07-03 10:00:00 +0300
author: "Ark"
permalink: "/functional-programming-1/"
---

This is the first part of the functional programming series, covering some fundamentals of F#, without getting caught up too much. 

Part 1

[Part 2]({% post_url /_posts/2024-07-05-FunctionalProgramming2 %})

The most common ways to run F# code is either using Visual Studio or VS Code with the Ionide extension.  (There is also an Ionide plugin for vim)

Install the .NET SDK and create a .fsx file

To run it, execute `dotnet fsi <filename>`

Alternatively, you can create a project and run it in the project directory.

```
dotnet new console --language F# --output <projectname> 
cd <projectname>
dotnet run
```

```fsharp
printfn "Hello, World!"
```

Use let to bind values to names. Values are immutable by default and cannot be changed.

```fsharp
let hello = "Hello, World!"
printfn "%s" hello // printfn takes a format string which works just like in
                   // C or C#. printfn also appends a newline at the end
printfn $"{hello}" // string interpolation is also supported
```

Functional programming takes its namesake from mathematical functions. Formally, mathematical functions map every element of some set to some arbitrary element in another set. Use let to define functions.

```fsharp
let addone x = x + 1 // addone: int -> int
addone 5 // 6
```

Note that we don't use parentheses in function signatures (unless you wish to annotate types) or when calling functions. We also did not write types in the function signature since the compiler is able to figure them out for us.

By the way, that + is known as an "infix" operator. The function name usually goes to the left but many people would find this a bit awkward in case of arithmetic. You can in fact also write `(+) x 1`

F# is a statically typed language, this means every type of every value is known at compile time. However unlike in C#, where the same function would have to be defined like the following,

```csharp
int addone(int x) {
	return x + 1;
}
```

we almost never have to do this in F. However that does not mean the type constraint does not exist. This does not work for example:

```fsharp
addone 5.0 // expected to have type int but has type float
```

Neither does this:
```fsharp
let twoX x = x + x // int -> int
twoX 3 // 6
twoX 3.0 // expected to have type int but has type float
```

That's because the compiler has constrained the type of `twoX` to `int -> int` based on our initial usage.

Let's write a different function.

```fsharp
let multiply (x, y) = x * y // multiply: int * int -> int
multiply (3, 5) // 15
```

Functions can only take one formal argument, so we put the arguments into a tuple to pass them. In F#, tuple types are indicated with * between elements of the tuple.

But it would be annoying to do this every single time, so enter higher order functions:

```fsharp
let multiply x y = x * y // multiply: int -> int -> int
multiply 3 5 // 15
```

The type signature of multiply appears a bit strange.  `int -> int -> int` makes it seem like the function takes an int and returns a function that takes an int which is in fact exactly what it does.

```fsharp
let multiplyWith3 = multiply 3 // multiplyWith3: int -> int
multiplyWith3 5 // 15
```

Functions are "first class" values in F#, so on higher order functions you can pass fewer than the expected number of arguments and get another function.

Only the last expression inside a function body is returned, so if you wish to have intermediate calculations, you can use the let keyword.

```fsharp
let heron a b c = // float -> float -> float -> float
    let s = (a + b + c) / 2.0
    let area2 = s * (s - a) * (s - b) * (s - c)
    System.Math.Sqrt area2
```

Whitespace is significant in F# (in fact, much more significant than in a language like Python). So expressions in the same level need to have the same indentation in spaces. The compiler may also sometimes complain that certain parts are not indented far enough.

```fsharp
let heron a   // yes this indentation is mandatory
          b   // if you wish to split successive arguments
          c = // to multiple lines, sorry!
    let s = ...
```
### Recursion

Although F# is perfectly capable of mutation, loops, and all the other fancy constructs that imperative programmers use all the time to crash your system, it turns out that mathematics can define many things neatly using recursion. You use let rec to define a recursive function.

```fsharp
let rec factorial x = // factorial: int -> int
    match x with
    | 0 -> 1
    | a -> a * factorial (a - 1)

factorial 5 // 120
```

We see our first match expression here. match is like a switch statement in imperative languages except much more powerful. The expression we give to the match statement is matched against the left side of the match arms and the first time it matches, the expression on the right hand side is returned.

In this case, `0` is our base case which we return 1. The `a` in our recursive case is just a binding name which will match any given expression (so you can use a different name if you want).

This kind of one argument function that immediately matches is so common in F# that there is syntax sugar for it.

```fsharp
let rec factorial = // factorial: int -> int
    function
    | 0 -> 1
    | x -> x * factorial (x - 1)
```

Those of you who are a bit astute are probably screaming that it's not in tail form.

![[functional.png]]

Let's say that you are trying to compute some high value (ignore integer overflow for a moment), our stack is likely going to end up looking like this:

factorial ...99
factorial ...98
factorial ...97
...
(BOOM)
\<Stack Overflow\>

That's because every time we recurse, there is a pending operation `x * ...` left. However, if there are no pending operations, the compiler can convert this to an iterative loop with a state variable.

```fsharp
let factorial x =
    let rec factorial' acc x =
        match x with
        | 0 -> acc
        | x -> factorial' (acc * x) (x - 1)
    factorial' 1 x
```

Here, we moved the result of the addition to an accumulator as a state variable which means that there are no pending operations when we perform recursion.

It is worth noting that you don't have to tail call optimize every single function, only the ones where there is potentially unbounded recursion. As a funny example, the following code crashes the Rust compiler:

```rust
fn main() {
	let x = &&&&&&&&&&& ... (10000 more refs) ... &&&&&&&&0;
}
```
I don't think that is high up the Jira list.
### Lists

While it is nice that we know how to write recursive functions. What if someone gave us a list of things they want an operation done on. In C# and other imperative languages, we'd create a new list and mutate every element. As it turns out, recursion is a very natural way of describing a key data structure in F#, the Linked List (no don't run away).

```fsharp
let rec factorialOfList lst = // int list -> int list
    match lst with
    | [] -> []
    | x :: xs -> factorial x :: factorialOfList xs

let a = [0; 1; 2; 3; 4; 5]
factorialOfList a // [1; 1; 2; 6; 24; 120]
```

Lists are created by putting values in square brackets with semicolons between values.  Inside the match, we have two interesting cases.

`[]` represents the empty list.
`::` is the "cons" operator. It is used for adding an element to the head of the list.

```fsharp
static member (::) // ('a * 'a list) -> 'a list  
```

So a list is either empty, or has an element followed by another list. Neat.

Programmers are however lazy and do not want to create a new function all the time when they want to apply a different operation to elements of a list. So we can factor out factorial (heh).

```fsharp
let rec listMap op lst = // ('a -> 'b) -> 'a list -> 'b list
    match lst with
    | [] -> []
    | x :: xs -> op x :: listMap op xs

let factorialOfList = listMap factorial // int list -> int list
```

Something very strange happens when we do this. The compiler has inferred the type of `listMap` to `('a -> 'b) -> 'a list -> 'b list` as `listMap` is a valid operation for any two types that have a mapping between them. In addition, our `factorialOfLists` function also did not constrain the type of `listMap` (reason for this is complicated but `::` is simply significantly more generic than `(+)`).

This is known as automatic generalization. We have written generic code without ever intending to write generic code. 

Now, what we wrote is just a somewhat unoptimized version of `List.map` but it is good to know how to write map directly as there are instances where we might have more than two cases.

```fsharp
let factorialOfList = List.map factorial
```

There are two more important operations we need to know about. In imperative programming you might have code like this.

```csharp
foreach (var x in ...) 
{
    if (cond(x)) 
    {
        ...
    }
}
```

The analogous operation is filter.

```fsharp
let rec listFilter op lst = // ('a -> bool) -> 'a list -> 'a list
    match lst with
    | [] -> []
    | x :: xs when op x -> x :: listFilter op xs
    | _ :: xs -> listFilter op xs

let cond x = (x % 2) = 1 // = is used for equality, not assignments
let example = [0; 5; 2; 3; 6]
listFilter cond example // [5; 3]
List.filter cond example // optimized std implementation
```

You can of then map whatever operation you were going to do to the list.

The final and most difficult to understand fundamental operation is the fold. A fold collapses every element in a collection to one element (called the state)

```csharp
var acc = ...
foreach (var x in ...) {
    acc = folder(acc, x);
}
return acc;
```

For example, we can use it to add all elements of a list.

```fsharp
let rec listFold folder state lst = 
    // ('State -> 'a -> 'State) -> 'State -> 'a list -> 'State
    match lst with
    | [] -> state
    | x :: xs -> let state' = (folder state x)
                 listFold folder state' xs

let example = [1; 2; 3; 4; 5]
listFold (+) 0 example // (+) has the type 't -> 't -> 't where 't has
                       // (+) as a static member
```

There are several more important adapters.

```fsharp
// zip combines two collections, useful when you need to iterate over
// multiple lists with the same number of elements at the same time
List.zip // 'a list -> 'b list -> ('a * 'b) list
// reduce is like fold but uses the first element of the list as the state
List.reduce // ('a -> 'a -> 'a) -> 'a list -> 'a
// map2 combines zip and map
List.map2 // ('a -> 'b -> 'c) -> 'a list -> 'b list -> 'c list
// collect takes a function that produces a list, and then flattens the lists
List.collect // ('a -> 'b list) -> 'a list -> 'b list


// where is for confused C# programmers
// it is identical to filter
List.where // ('a -> bool) -> 'a list -> 'a list
```

Note that there are many more some of which are fairly advanced and specific.
### Lambda expressions

Lambda expressions scare beginners due to having a scary name and poor support in languages they first learn them in. However they are nothing to be afraid of.

```fsharp
let multiply a b = a * b // int -> int -> int
let multiply = fun a b -> a * b // int -> int -> int
```

The most common use case is with list adapters where you need to define a function but it is short enough and only used in one place that you don't want to have a separate definition of it.

```fsharp
let lst = [(1, 2); (3, 4); (5, 6)]
let flattenPairs = List.collect (fun (a, b) -> [a; b])
// ('a * 'a) list -> 'a list
flattenPairs lst // [1; 2; 3; 4; 5; 6]
```
### Chaining Adapters with |>

Let's get to the part of why functional programming looks so weird to so many people. Let's say you want to zip two lists, take the larger of the two elements in each tuple, sort the remaining elements, and then do a running sum and return the list. It looks something like this.

```fsharp
fst (List.mapFold (fun acc e -> (acc + e, acc + e)) 0.0 (List.sort (List.map (fun (a, b) -> if a > b then a else b) (List.zip lst1 lst2))))
```

If that looks completely unreadable, you're not the only one. Splitting operations to separate lines does not make things much better.
```fsharp
fst // fst takes the first element of a tuple
    (List.mapFold 
                (fun acc e -> (acc + e, acc + e)) 
                0.0 
                (List.sort 
                    (List.map 
                        (fun (a, b) -> if a > b then a else b) 
                        (List.zip lst1 lst2))))
```

The problem is the first operation is actually on the last line, clearly what we need are intermediate products.

```fsharp
let lst = List.zip lst1 lst2
let lst' = List.map (fun (a, b) -> if a > b then a else b) lst
let lst'' = List.sort lst'
let lst''' = List.mapFold (fun acc e -> (acc + e, acc + e)) 0.0 lst''
let lst'''' = fst lst'''
lst''''
```

This helps, but there is still a lot of garbage. Now, one thing you will notice is that the list is always the last argument to all of these functions which is not a coincidence. Enter the infix operator `|>`. The only thing it does is feed the value to its left to the function to its right.

```fsharp
List.zip lst1 lst2 
// same as
lst2 |> List.zip lst1
```

So the above code is equivalent to the below code.

```fsharp
List.zip lst1 lst2
|> List.map (fun (a, b) -> if a > b then a else b)
|> List.sort
|> List.mapFold (fun acc e -> (acc + e, acc + e)) 0.0
|> fst
```

### Sum types

Up to this point, we have only written functions. But to manage anything larger than the most basic programs, we need a way to aggregate data. F# being a fully fledged .NET language supports every type that you can create in C#, however the way types are used in idiomatic F# code can be quite different in F#. The most basic type definition is the type redefinition.

```fsharp
type Handle = string
```
The redefined type is identical to the original type and this is mainly used to either convey intent or to compact large type definitions. We have also seen tuple types.

```fsharp
// ML like syntax
type 'a PairList = ('a * 'a) list 
// C# like syntax
type PairList<'a> = list<'a * 'a>
```

We have seen that generic type parameters are indicated with a quote followed by an identifier. Tuples can also have elements of different types.

```fsharp
type 'a NamedElem = string * 'a
```

In many problems that we model, it is fairly common that there are several "shapes" that our data can take. The most common of these is the absence of a value. Consider the following function:

```fsharp
let rec findElem name (lst: 'a NamedElem list): 'a =
    match lst with
    | [] -> ??? // what to do here
    | (x, value) :: _ when x = name -> value
    | _ :: xs -> findElem name xs
```

Now, if you said null, I presume you are still stuck with Java in 2024 (my condolences). Throwing an exception is an option if you are generally not expecting this function to ever fail (thus making a failure exceptional). But F# offers a better solution.

```fsharp
type 'a Option =
    | None
    | Some of 'a 
```

Thus our function becomes:

```fsharp
let rec findElem name (lst: 'a NamedElem list): Option 'a =
    match lst with
    | [] -> None
    | (x, value) :: _ when x = name -> Some value
    | _ :: xs -> findElem name xs
```

An Option is known as a sum type and one of many sum types defined by default in F#, they are essentially tagged unions in many languages but thanks to pattern matching and the type system, they are super powerful. None and Some are fields in the type and are used to construct instances of it.


### That Monad thing

A monad is a type that defines a flatmap operation (also known as a bind). We do not talk about Monads much in F# for a few reasons.

1. F# does not have typeclasses (typeclasses are like a superpowered version of interfaces).
2. F# allows mutation meaning Monadic interfaces are not as needed.
3. We just don't call Monadic types Monads.
4. F# has computation expressions that are probably the most "monadic" but there are often good reasons to avoid using them


```fsharp
type 't Monad =
    member (>>=): ('t -> 'u Monad) -> 't Monad -> 'u Monad
```

```fsharp
let bind op result = // ('a -> 'b Result) -> 'a Result -> 'b Result
    match result with
        | Ok value -> op value
        | Error err -> Error err
```

Was that so complicated?

### Function Purity

It is difficult to define formally what it means for a function to be pure but informally, a pure function does not mutate any of its input parameters, always returns the same output for the given input parameters, does not throw exceptions and does not modify global state.

All of the functions we have written so far have been pure. Here is an example of an impure function:

```fsharp
let d1 = System.Random.Shared.NextDouble ()
let d2 = System.Random.Shared.NextDouble ()
```

And that makes sense because a random function that returns the same value each time is useless and a random function that requires you to carry around its state with you is inconvenient!