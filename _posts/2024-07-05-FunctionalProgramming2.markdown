---
layout: post
title: "Functional Programming 2"
date: 2024-07-05 7:00:00 +0300
author: "Ark"
permalink: "/functional-programming-2/"
---

This is part 2 of the functional programming series illustrating an example library and application.

[Part 1]({% post_url /_posts/2024-07-03-FunctionalProgramming %})

Part 2

Inspired by the brilliant library that is FParsec as well as just seeing there being a page for parser combinators in F# for Fun and Profit, I may as well write my own.

Let's start by discussing what the parsing problem is. Given a grammar, decide if a given input string is a valid sentence in the grammar (and if it is, return an abstract syntax tree representation of it). The parsing problem is undecidable in the most general case known as recursively enumerable grammars, but we won't be dealing with those today. It is pretty likely that you have run into a situation once or twice when you had to parse some arbitrary text file and ran into some issues.

![[1920px-Chomsky-hierarchy.svg.png]]

However, it doesn't have to be that way and I will illustrate that using JSON. JSON is pretty special because it combines two properties..
1. It is dead simple. The entire language is 6 railroad diagrams and we will ignore 1 of them.
2. It is "too powerful" for regular expressions.

### Regular Expressions

Regex is interesting in that it is very powerful for some tasks to the point that many find it almost unruly to use but regex is also very weak in the sense that it can only parse regular grammars which JSON is not part of ([dead simple proof in stack exchange](https://cstheory.stackexchange.com/a/4017)).

For the solution provided here, we will use absolutely no regular expressions, but of course you're free to use them if you'd like.

### What is a Parser Combinator?

To solve the parsing problem, powerful tools such as Lex and Yacc were created. These are parser _generators_. Typically, they take a description of a lexical specification and grammar in a declarative way and then write a parser for you. While this metaprogramming has its advantages, it is also overly complicated, however there is a simpler way.

Let's say that we want to parse a single char, we might consider writing something like this

```fsharp
parseChar: char -> string -> Result<char * string, Errors>
```

We take the character we are looking for as input, as well as the input string. If we match the character, we return the character and a new string with the character removed. We can wrap the part that will be common in all parsers into its own type.

```fsharp
// Combinator.fs
module Combinator

type ParsingError = string list
type 'a Parser = Parser of (string -> Result<'a * string, ParsingError>)
```

Now we can write our first parser that parses a single given character:

```fsharp
// Combinator.fs
let pchar c = // char -> char Parser
    let parse (input: string) =
        if input.Length = 0 then
            Error [$"Expected {c}, got empty string"]
        else if input.[0] = c then
            Ok (c, input.[1..])
        else
            let actual = input.[0]
            let remaining = input.[1..]
            Error [$"pchar: Expected {c}\n got {actual}\n remaining: {remaining}"]
    Parser parse
```

Phew, that's a lot of code, and a lot of it is handling edge cases. But I will promise it will pay off soon. Parsing a given string turns out to be quite similar and we can use the StartsWith function within the String class.

```fsharp
// Combinator.fs
let pstring (s: string) = // string -> string Parser
    let parse (input: string) =
        if s.Length > input.Length then
            Error [$"pstring: Expected {s}\ngot {input}"]
        else if input.StartsWith(s) then
            Ok (s, input.[s.Length..])
        else
            let actual = input.[0..s.Length-1]
            let remaining = input.[input.Length..]
            Error
	            [$"pstring: Expected {s}\n got {actual} \n remaining: {remaining}"]
    Parser parse
```

Now, the only other thing we need is a way to combine parsers. But we will get there in a bit.

### Parsing JSON

```fsharp
// JSON.fs
module JSON
open Combinator

type JValue =
    | JObject of Map<string, JValue>
    | JArray of JValue list
    | JString of string
    | JNumber of float
    | JBool of bool
    | JNull

type JSON = Json of JValue
```

JSON is a dead simple data exchange standard discovered (allegedly) by Doug Crackford. It defines 6 types of JSON Values. An array type which is a finite list of JSON values delimited with \[\], an object type, which is a key value store with string keys delimited with \{\}, a string type that is delimited with quotes ("), a number type, a boolean type which can be true or false, and a null value that can only be null. You can find the lexical details in its [website](https://www.json.org/json-en.html).

Let's start with the simplest type, the null.

```fsharp
// JSON.fs
let parseNull = // string Parser
    pstring "null"
```

Unfortunately, we need a way to get this to parse as a JValue. We need to essentially be able to take the value stored in the parser state and convert it to another value.

We can do that with the following function.

```fsharp
// Combinator.fs
let wrap constructor parser = // ('a -> 'b) -> 'a Parser -> 'b Parser
    let wrap' input =
        match parser with Parser func -> func input
        |> Result.bind
	        (fun (parsed, remaining) -> Ok (constructor parsed, remaining))
    Parser wrap'
```

If you look carefully at the inferred function signature, we have essentially wrote a map operation for the parser. We once again need an inner function to return (you can write it in lambda form to avoid naming it but personally I think this is fine). One thing you will notice is that nearly all our functions in our library file will be like this while everything will be neat and tidy in the JSON parser.

```fsharp
// JSON.fs
let parseNull = // JValue Parser
    pstring "null" |> wrap (fun _ -> JNull)
```

There is one more problem. All values in the JSON standard are allowed to be followed with meaningless whitespace. Whitespace can be meaningful in many languages but we will take the lazy route this time.

```fsharp
// Combinator.fs
let pspaces = // () Parser
    let parse (input: string) =
        let remaining = input.TrimStart()
        Ok ((), remaining)
    Parser parse
```

Parsing whitespace cannot fail for our case, so we unconditionally return Ok.

Now we need to write a function to do the combination. The and operator runs both parsers in succession

```fsharp
// Combinator.fs
let parserAnd p1 p2 = // 'a Parser -> 'b Parser -> ('a * 'b) Parser
    let parse input =
        let result = pget p1 input
        match result with
            Ok (value, remaining) ->
                match pget p2 remaining with
                    | Ok (value2, remaining2) -> Ok ((value, value2), remaining2)
                    | Error err -> Error err
            | Error err -> Error err
    Parser parse
```

This combinator takes two parsers, and returns a parser that succeeds if both parsers succeed and returns their output as a tuple. Now, we can properly parse null.

One problem you'll notice is that the behavior of `parserAnd` is confusing when used alongside the pipe operator |>.

```fsharp
pA |> parserAnd pB // applies pB first, then pA
```

In addition, a lot of the times when we subsequently apply parsers, we are only interested in the result of one of them. Now, there's a really neat way of solving both of these problems in a very clean way, but let's just define a few helper functions first and move on.

```fsharp
// Combinator.fs
let parserAnd2 p2 p1 = parserAnd p1 p2

let pfst p = // ('a * 'b) parser -> 'a Parser
    let parse input =
        match pget p input with
            | Ok ((a, b), remaining) -> Ok(a, remaining)
            | Error err -> Error err
    Parser parse

let psnd p = // ('a * 'b) parser -> 'b Parser
    let parse input =
        match pget p input with
            | Ok ((a, b), remaining) -> Ok(b, remaining)
            | Error err -> Error err
    Parser parse
```

```fsharp
// JSON.fs
let parseNull = // JValue Parser
    parserAnd (pstring "null") pspaces |> wrap (fun _ -> JNull)
```

Next, let's parse booleans. With booleans, we have two potential states, "true" and "false"

```fsharp
// JSON.fs
let parseBool = // JValue Parser
    parseTrue = pstring "true" |> parserAnd2 pspaces |> wrap (fun _ -> JBool true)
    |> parserOr <|
    (pstring "false" |> parserAnd2 pspaces |> wrap (fun _ -> JBool false))
```

You will notice the somewhat lesser used brother of the forward pipe operator |>, the reverse pipe operator <|. It's fairly vestigial in this case but allows messing with the order of arguments a bit. The first line becomes the first argument to parserOr and the bottommost line becomes the second, the parantheses are required.

### Infix operators using static members

The concept of an operator is a bit difficult to explain in mathematics (mainly in the sense of how an operator is different from a function) but pretty much all of you should be familiar with the topic of operator precedence. When we do math, powers have precedence over multiplication and division, and those have precedence over addition and subtraction.

\[2 + 4 * 3]\

This is known as infix notation. Programming languages mostly respect this convention, including F#. The problem is programming languages typically have far more operators to worry about. There's of course equality and inequality which are not operators in mathematics but are operators producing truth values in most programming languages, bitwise operators, and pretty much every language has at least one or two very unique operators.

Now, most people only have so much working memory and mentally parsing huge expressions is often difficult so parantheses are used to make everything clear (even if they may not be strictly necessary)


Infix operators must be defined as static members of a type. One limitation of F# is that you cannot reference names defined after some point and our combinator functions rely on our type. However, what we can do is define extensions to existing types.

I will steal the operator names from FParsec (in case you wish to try that library).

```fsharp
// Combinator.fs
type 'a Parser = Parser of (string -> Result<'a * string, ParsingError>)
// ...
// definitions for parserAnd, parserOr, pfst, psnd etc...
// ...
type 'a Parser with
    /// first apply the parser to the left, then apply the parser to the right
    /// return the result of both parsers in a tuple
    static member (.>>.) (p1, p2) =
        parserAnd p1 p2

    /// first apply the parser to the left, then apply the parser to the right
    /// return the result of the left parser
    static member (.>>) (p1, p2) =
        parserAnd p1 p2 |> pfst

    /// first apply the parser to the left, then apply the parser to the right
    /// return the result of the right parser
    static member (>>.) (p1, p2) =
        parserAnd p1 p2 |> psnd

    /// first apply the parser to the left, if it fails, apply the parser to the right
    static member (<|>) (p1, p2) =
        parserOr p1 p2
```

This allows very terse yet unambiguous chaining of operations.

### Conditional Parsing

Next up are strings. Now, strings do have some subtle lexical details we need to worry about if you want to write a fully compliant JSON parser, but we will ignore them for now. A string starts with a quotation mark, is followed by any unicode codepoint except for a backslash, quote or control characters. We'll pretend the escape sequences don't exist (you can consider it a challenge to write a parser covering them).

First for a single char.

```fsharp
// Combinator.fs
let pcond cond = // (char -> bool) -> char Parser
    let parse (input: string) =
        if input.Length = 0 then
            Error ["Unexpected end of sequence"]
        else if cond input.[0] then
            Ok(input.[0], input.[1..])
        else
            Error [ $"Unexpected \"{input.[0]}\"" ]
    Parser parse
```

However, this one is not too useful, we need essentially a takeWhile version of this which is the following.

```fsharp
// Combinator.fs
let pcondWhile cond = // (char -> bool) -> string Parser
    let parse (input: string) =
        let matched = 
            input.ToCharArray() 
            |> Array.takeWhile cond 
            |> Array.map string 
            |> String.concat "" 

        if matched.Length > 0 then
            Ok (matched, input.[matched.Length..])
        else
            Error [$"Matched no characters"]
    Parser parse
```

And now we can write our parseString function.

```fsharp
// JSON.fs
let parseString =
    let cond c =
        not (Char.IsControl(c) || c = '\"')
    let psep = pchar '\"'

    psep >>. (pcondWhile cond) .>> psep .>> pspaces
    |> wrap (fun str -> JString str)
```

### Top level and parsing arrays

The remaining things to write are arrays and objects as well as the top level parsers. We'll write the top level parsers first since they don't do any new things.

```fsharp
// JSON.fs
let parseValue = // JValue Parser
    parseNull
    <|> parseBool
    <|> parseString
    <|> parseArray  // TODO
    <|> parseObject // TODO

let public parseJson = // Json Parser
    pspaces >>. parseValue
    |> wrap (fun jvalue -> Json jvalue)
```

We need to first apply pspaces since JSON can have arbitrary whitespace at the beginning.

We'll do array before object since it is simpler. It is often the case that we have to parse grammars of the following form:

```
elements ::= ∅
          |  element "," elements
```

That's exactly what a JSON array is (minus the required square brackets). First we need a zero or more parser. We'll also write a simple utility function which we'll use for the brackets.

Since there is no limit on recursion depth, pmany has to use only tail calls.

```fsharp
// Combinator.fs

// parse 0 or more with pvalue separated by psep
let pmany (psep: Parser<'a>) (pvalue: Parser<'b>): Parser<'b list> =
    let rec pmany' input lst =
        match pget pvalue input with
            | Ok (inside, remaining) -> 
                match pget psep remaining with
                    | Error _ -> inside :: lst, remaining 
                    | Ok (_, remaining) ->
                        pmany' remaining (inside :: lst)                 
            | Error _ -> [], input
    let parse input =
        let (lstrev, remaining) = pmany' input []
        Ok (List.rev lstrev, remaining)
    Parser parse

let pbetween (pleft: 'dc1 Parser) (pright: 'dc2 Parser) (pcenter: 'a Parser) =
    pleft >>. pcenter .>> pright
```

With these, it is really easy to define the parseArray function.

```fsharp
// JSON.fs
let parseArray = // JValue Parser
    let pleft = pchar '[' >>. pspaces
    let pright = pchar ']' >>. pspaces
    let psep = pchar ',' >>. pspaces
    pbetween pleft pright (pmany psep parseValue)
    |> wrap (fun pvalues -> JArray pvalues)
```

And parseObject is only a slightly more complicated version of this.

```fsharp
let parseObject' =
    let pleft  = pchar '{' >>. pspaces
    let pright = pchar '}' >>. pspaces
    let psep   = pchar ',' >>. pspaces
    let psep2  = pchar ':' >>. pspaces

    let pkey = parseString |> wrap (fun elem -> match elem with JString str -> str | _ -> failwith "parseObject: typeError")

    let pkeyvalue = pkey .>> psep2 .>>. parseValue

    pbetween pleft pright (pmany psep pkeyvalue)

    |> wrap (fun pvalues -> JObject (Map.ofList pvalues))
```

However, we have a bit of a problem. 

```fsharp
let parseValue =
    // ...
    parseArray // parse array is not defined

let parseArray =
    // ...
    parseValue
```

Using the and keyword does not solve this.

```fsharp
let parseValue = // parseValue evaluates parseArray
    // ...
    parseArray

and parseArray = // parseArray evaluates parseValue
    // ...
    parseValue
```

Since F# allows recursive function definitions to be recursive, but not values. We need a form of indirection to make this work.

### Recursive definitions and RefCell

I will use more or less the same solution that FParsec uses for our problem since this gives us a good opportunity to talk about reference cells and mutation.

A reference cell is a mutable location in memory. In F#, when you pass values to a function, that value will not change when you next use it even if it is marked as mutable. A reference cell opts out of that guarantee and allows arbitrary mutation. This somehow [sounds familiar](https://doc.rust-lang.org/stable/std/cell/index.html).

Reference cells are unwieldly to use and can make it very difficult to reason what your code does, so it is best to use them sparingly. Lazy initialization is one area where we need them.

Reference cells are created with the ref function and can be read from by accessing the Value field. 

```fsharp
let a = ref 0

let printa () =
    printfn "%d" a.Value

printa () // 0

a.Value <- 42

printa () // 42
```

With this knowledge, we create the following function. ForwardDeclare takes no arguments and returns two values. First, a reference cell that initially contains a dummy parser that crashes. And a second parser that just calls passes its input to the parser inside the reference cell.

```fsharp
// Combinator.fs
let forwardDeclare () = // unit -> ref<Parser<'a>> * Parser<'a>
    let pfail = Parser (fun _ -> failwith "This parser always fails")
    let pref = ref pfail
    let pforward = Parser (fun input -> prun (pref.Value) input) 
    pref, pforward
```

This function is overloaded on its output values only and F# is able to deduce the types of the outputs based on their later usage.

```fsharp
// JSON.fs
let pobjectref, parseObject  = forwardDeclare()
let parrayref, parseArray = forwardDeclare()

let parseValue =
    // ...
    <|> parseArray
    <|> parseObject // no longer depends on the later values

let parseObjectActual =
    // ...
pobjectref.Value <- parseObjectActual

let parseArrayActual =
    // ...
parrayref.Value <- parseArrayActual
```

And that's it.

### Challenge: Parse JSON Numbers

Hint: Parse the number as text and use the built in Double.Parse in the standard library. Note that parsing floats is locale sensitive in .NET so you will need to use the following form.

```fsharp
let culture = Globalization.CultureInfo.InvariantCulture
let toFloat = (fun (str: string) -> Double.Parse(str, culture) |> JNumber)
```

Hint: Consider writing an additional parser combinator for regex.

```fsharp
let pregex regex = // string -> string Parser
    // ...
```

Alternatively, you can try parsing it using only parser adapter chains.

### Notes on the implementation

For simplicity, I've represented the parser state purely using `string` but in practice, you most likely want to store line and column information as well (for better error messages) or even go one step further and add an explicit generic type parameter for the parser state like FParsec did so library consumers can pass state around more easily.

Additionally, string is not the most efficient type to represent the state of the input stream because they are always in memory, and each operation on a string creates a new string (even though we never modify the strings). Passing around slices and a single string would be significantly faster and have much lower GC pressure especially on much more complicated grammars on big files.

Error handling has also been an afterthought in this implementation with us simply adding up a bunch of non machine readable strings side by side.
