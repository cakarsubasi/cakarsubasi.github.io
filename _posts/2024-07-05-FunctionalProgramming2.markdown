---
layout: post
title: "Functional Programming"
date: 2024-07-05 7:00:00 +0300
author: "Ark"
permalink: "/functional-programming-2/"
---

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
let pspaces =
    let parse (input: string) =
        let remaining = input.TrimStart()
        Ok ((), remaining)
    Parser parse
```

Parsing whitespace cannot fail.

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