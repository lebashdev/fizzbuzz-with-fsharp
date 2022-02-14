# Introduction

F# is a wonderful language.  One thing I appreciate about it as a programmer is its expressiveness.  In this article, I'll attempt to demonstrate this by (ab)using various language features to implement a classic FizzBuzz algorithm, and try to look the pros and cons of each solution.

# FizzBuzz in C#

Before we dive into F#, we should review a fairly classic implementation in C#.

```csharp
using System;

for (var x = 1; x <= 100; x++)
{
	if ((x % 15) == 0) { Console.WriteLine("FizzBuzz"); }
	else if ((x % 3) == 0) { Console.WriteLine("Fizz"); }
	else if ((x % 5) == 0) { Console.WriteLine("Buzz"); }
	else { Console.WriteLine(x); }
}
```

Because I'm targeting .NET 6, which supports top-level statements, I don't need to wrap this code in a class.  There are a few reference implementations of this.  This one is IMO clear and concise, and it solves the standard definition of the problem.


# FizzBuzz in F#

Let's take that C# implementation, and port it over to F#.

```fsharp
let mutable x = 1

while x <= 100 do
    if x % 15 = 0 then printfn "FizzBuzz"
    else if x % 3 = 0 then printfn "Fizz"
    else if x % 5 = 0 then printfn "Buzz"
    else printfn "%i" x
    x <- x + 1
```

They look pretty similar.  The main difference is how the loop is implemented.  F# does not support the `for(;;)` construct.  So we fake it with a `while` loop, and a mutable variable.  With every iteration of the loop, we replace the value of `x` by `x + 1`.

Even though that's a perfectly valid program, that solution is not completely idiomatic.  In F#, the default is to make everything immutable.

## Recusion and immutability

We can eliminate that mutable field, but we then need to change how we loop over each value.  We can use a recursive function to achieve that.

```fsharp
let rec solve x =
    if x % 15 = 0 then printfn "FizzBuzz"
    else if x % 3 = 0 then printfn "Fizz"
    else if x % 5 = 0 then printfn "Buzz"
    else printfn "%i" x
    if x < 100 then f (x + 1) else ignore x

solve 1
```

Here the `while` loop is replaced by a recursive function.  We initiate the loop by calling the value with `1`.  Then the function keeps invoking itself as long as it is given an `x` value which is less than 100, incrementing the value of the `x` parameter with each recursion.  The `ignore` function is used to interrupt the loop by, well, not doing anything with the current value of `x`.

This example introduces the syntax for defining functions:

```fsharp
let f x = <some logic>
```

And the syntax to invoke them:

```fsharp
f <some value>
```

Also, `x` is no longer mutable in this example.  With each recursion, a new value is created in memory, and passed to function.  The old value is not replaced.

## A more intuitive looping construct

The example above is fine, but it will look a bit esoteric to those of us more familiar with iterative looping.

While recursion is an important concept in functional programming, it is possible to rewrite our algorithm in a way that both eliminates mutability and leverages an iterative loop.

```fsharp
for x in [1..100] do
    if x % 15 = 0 then printfn "FizzBuzz"
    else if x % 3 = 0 then printfn "Fizz"
    else if x % 5 = 0 then printfn "Buzz"
    else printfn "%i" x
```

Here again, the conditional logic inside the loop remains more or less unchanged.  The simplification comes from the use of the `for..in..do` construct, which puts us back in familiar territory.

This implementation introduces some new language features:

- Lists, which are F#'s native way of representing finite collections
- Ranges (`<start>..<end>`)
- `for` loops

In plain English, this implementation first produces an immutable list of numbers from 1 to 100, then applies the FizzBuzz logic to each value in that list.

## Introducing laziness

There is a small inefficiency in the previous implementation.  It allocates the entire collection and maintains it in memory for as long as the algorithm is running.  In our case, the list is very small, so there is no real concern about wasting memory, but when collections become very large, whether we should buffer a large amount of data becomes a legitimate question.

The FizzBuzz algorithm only needs to look at each number in the range once, and it keeps moving forward.  It's a good candidate for sequences.

```fsharp
for x in seq { 1..100 } do
    if x % 15 = 0 then printfn "FizzBuzz"
    else if x % 3 = 0 then printfn "Fizz"
    else if x % 5 = 0 then printfn "Buzz"
    else printfn "%i" x
```

Compared to the previous example, the only difference is the use of `seq { 1..100 }` instead of `[1..100]`, which transforms our finite range into a lazily computed sequence.

Sequences are a powerful construct, and they are not specific to F#.  C# supports them in the form of `IEnumerable<T>`, which is the basis for extremely popular features such as LINQ.  The basic idea behind sequences is that they are not static data, but computation expressions.  Instead of producing all the values in a collection and storing them in memory at once, we incrementally produce them any time the consuming logic needs to pull an item from the sequence.

Now that we've reviewed several ways to implement the looping aspect of the algorithm, let's move our focus back to the conditional logic at its core.

## Inroducing pattern matching

IMHO, one of F#'s best features is pattern matching.  Its syntax has been very stable since version 1.0, which is a testament to how well it has been designed.  C# progressively introduced pattern matching over the years, but IMO it's not nearly as elegant as in F#.

```fsharp
for x in seq { 1..100 } do
    match x with
    | x when x % 15 = 0 -> printfn "FizzBuzz"
    | x when x % 3 = 0 -> printfn "Fizz"
    | x when x % 5 = 0 -> printfn "Buzz"
    | _ -> printfn "%i" x
```

The algorithm did not fundamentally change here.  We have just replaced the `if..else` construct by the `match...with` construct.  Notice how the logic inside each match expression is pretty much identical to what we had before.  It is fine, but there is no real gain in terms of conciseness.

Another thing we introduce here is the wildcard pattern (`_`) which will match anything that wasn't matched by the more specific patterns.

## Deconstructing values

This version addresses the verbosity of the previous implementation.

```fsharp
for x in seq { 1..100 } do
    match (x % 3, x % 5) with
    | (0, 0) -> printfn "FizzBuzz"
    | (0, _) -> printfn "Fizz"
    | (_, 0) -> printfn "Buzz"
    | _ -> printfn "%i" x
```

Instead of what basically equates to classic `if` conditions, we rely on a feature called destructuring, which can be combined with pattern-matching.  We create a pair of values, group them into a tuple, and then try to match it with various combinations of shapes **and** contents, until there is a match.  This form of pattern-matching is very common in F#.  Destructuring is not limited to tuples and applies to a variety of data structures.

## Deconstructing our solution

The implementation above would be my preferred implementation.  I find it to be a good example of idiomatic F#.  It is concise but not cryptic.  It showcases many good features of the language.

For the sake of continuing our journey through F#'s language features, let's introduce some complexity, as an excuse to explore additional concepts.

```fsharp
let fizzBuzz x =
    match (x % 3, x % 5) with
    | (0, 0) -> "FizzBuzz"
    | (0, _) -> "Fizz"
    | (_, 0) -> "Buzz"
    | _ -> string x

let print = printfn "%s"

seq { 1..100 }
|> Seq.map fizzBuzz
|> Seq.iter print
```

So far our code was 100% impure, as it produced side-effects by printing directly to the standard output.  The essence of functional programming is composition, i.e., the ability to produce complex behavior by combining many simple functions.  Because there are issues associated with writing "impure" functions, it is generally a good idea to separate pure logic from impure logic.  Pure logic is good because it makes things more predictable.  Impure logic may produce sneaky side-effects that cannot be formally modeled as inputs mapping to outputs.

The implementation above splits the algorithm into two parts.  The `fizzBuzz` function is pure.  It takes a value as its input, and produces a value without causing any side-effects.  It doesn't matter how many times you call it.  It will always return the same value.  The `print` function, however, is impure.  It does not return anything, and just writes a string to the standard output.

The pure and impure logic are combined into a pipeline in the final portion of the program, where we introduce the pipe (`|>`) operator, which takes the result of an expression and feeds it to the next.  Only the last line in that pipeline is impure.  The very last thing that program does is applying its side-effects.

## Formal composition

Pipes are one common way to assemble complex workflows.  Another way leverages formal composition.  The defintion of formal composition is pretty simple.  Given two functions `f :: A -> B` and `g :: B -> C`, we can compose a new function `g . f` of type is `A -> C`.  Composition glues two function together, hiding what happens in the middle.

```fsharp
let fizzBuzz x =
    match (x % 3, x % 5) with
    | (0, 0) -> "FizzBuzz"
    | (0, _) -> "Fizz"
    | (_, 0) -> "Buzz"
    | _ -> string x

let print = printfn "%s"
let solveThenPrint = fizzBuzz >> print

seq { 1..100 } |> Seq.iter solveThenPrint
```

This new example is similar to the one before, but rather han applying `fizzBuzz` and `print` separately, we combine them with the composition operator `>>` and apply the resulting function to each element of the sequence.

Note that `fizzBuzz` is still a pure function, but because it is combined with the `print` function, which is impure, the resulting function is impure.

Composition is the essence of functional programming.  There are many ways to apply it in our example.  As long as something logically forms a function, it can be composed.  Here is an alternative to our previous example:

```fsharp
let fizzBuzz x =
    match (x % 3, x % 5) with
    | (0, 0) -> "FizzBuzz"
    | (0, _) -> "Fizz"
    | (_, 0) -> "Buzz"
    | _ -> string x

let print = printfn "%s"

let fizzBuzzMany = Seq.map fizzBuzz
let printMany = Seq.iter print
let solveThenPrint = fizzBuzzMany >> printMany

seq { 1..100 } |> solveThenPrint
```

Rather than composing the functions we apply to the discrete values, we compose the functions we apply to the whole sequence.

## Introducing discriminated unions

No F# article would be complete without mentioning discriminated unions.  It's yet-another powerful feature of the language, which allows us to define a type representing various shapes of data.  For something like FizzBuzz, we can describe the various possible outcome of the algorithm with a single union type.

```fsharp
type Output = FizzBuzz | Fizz | Buzz | Other of int

let fizzBuzz x =
    match (x % 3, x % 5) with
    | (0, 0) -> FizzBuzz
    | (0, _) -> Fizz
    | (_, 0) -> Buzz
    | _ -> Other x

let printOutput x =
    match x with
    | Fizz -> printfn "Fizz"
    | Buzz -> printfn "Buzz"
    | FizzBuzz -> printfn "FizzBuzz"
    | Other x -> printfn "%i" x

let transform = Seq.map fizzBuzz
let print = Seq.iter printOutput
let solve = transform >> print

seq { 1..100 } |> solve
```

This new example decouples the logical representation of what the algorithm produces from their concrete representation in the standard output.  That implementation is definitely not efficient.  It's just there to demonstrate how we can produce and consume union types, and how they work well with pattern-matching.  Union types have lots of practical applications, from handling errors, to describing recursive data structures.

## Computing all possible values

So far we've used a very simple sequence (`1..100`).  As explained earlier, it represents a computation which yields values upon request, and not static data, which saves us from allocating all the values in memory at once.  This particular sequence represents a finite range, but we can theoretically implement infinite sequences.  As an example, we could create a sequence representing all the FizzBuzz solutions for all the positive integers.

```fsharp
let allFizzBuzzValues = seq {
    let mutable x = 1
    while true do
        if x % 15 = 0 then yield "FizzBuzz"
        else if x % 3 = 0 then yield "Fizz"
        else if x % 5 = 0 then yield "Buzz"
        else yield string x
        x <- x + 1
    }
```

As you can see, there is nothing to exit the loop within the expression.  In theory, it represents a gigantic number of values, yet it consumes very little memory.  It is made possible by the fact that such expression is lazily evaluated.

Here is how we could consume that sequence and only print the values within the 10-19 range.

```fsharp
allFizzBuzzValues
|> Seq.skip 9
|> Seq.take 10
|> Seq.iter (printfn "%s")
```

We are back to using mutable fields though.  Here is one way to fix it, by using `Seq.unfold` which takes a seed and produces new values from it.

```fsharp
let allFizzBuzzValues =
    Seq.unfold (
        fun x ->
            if x % 15 = 0 then Some ("FizzBuzz", x + 1)
            else if x % 3 = 0 then Some ("Fizz", x + 1)
            else if x % 5 = 0 then Some ("Buzz", x + 1)
            else Some (string x, x + 1)) 1
```

It essentially replaces the `while` loop and keeps yielding values as long as the function passed to it returns `Some` value.  `Some of 'a` is part of the `Option<'a>` discriminated union we've mentioned earlier, and indicates the presence of a value, as opposed to `None`, which can be seen as some sort of type-safe `null`.

```fsharp
type Option<'a> =
   | Some of 'a
   | None
```

An interesting point here is that using `Seq.unfold` makes the algorithm significantly faster than the version using the mutable field.  F#'s standard library is heavily optimized, and allows us to use functional constructs without paying a heavy price.  That being said, computing sequences this way is a cool trick, but it does not scale well.  The ranged sequence (`seq { x..y }`) would still be a much more efficient way of producing our input data.

## Decomposing and recomposing

Assuming that we want to pursue the infinite sequence approach, one inefficiency in our previous version is that we apply the FizzBuzz calculation within the infinite sequence generation, even if we end up discarding the value with `Seq.skip`.  That seems like a waste of CPU cycles.  It makes sense to decompose the logic to improve it.

First, we isolate the sequence generator, and have it only produce the input values, rather than the resulting FizzBuzz values.

```fsharp
let positiveIntegers = Seq.unfold (fun x -> Some (x, x + 1)) 1
```

Then, we wrap the transformation logic in a simple function which takes an integer input, and produces a string.

```fsharp
let fizzBuzz x =
    if x % 15 = 0 then"FizzBuzz"
    else if x % 3 = 0 then "Fizz"
    else if x % 5 = 0 then "Buzz"
    else string x
```

And we glue everything together into a pipeline.

```fsharp
positiveIntegers
|> Seq.skip 0
|> Seq.take 100
|> Seq.map fizzBuzz
|> Seq.iter (printfn "%s")
```

With this updated version, the fizzbuzz logic is only applied to the integers that have been pulled from the sequence.  Skipped integers are never transformed.

## Unnecessary complexity

To wrap this up, we can make our code unnecessarily complex.  This will be an opportunity to introduce record types.

Instead of hard-coding the parameters of our algorithm, we can use a declarative approach and encode them as data.  Let's create a definition of what that data should look like.

```fsharp
type Case = { DivisibleBy: int; Output: string }
```

We have created a type named `Case`, which describes an object with two fields.  Now we can declare the cases from the FizzBuzz problem.

```fsharp
let cases =
    [
        { DivisibleBy = 15; Output = "FizzBuzz" };
        { DivisibleBy = 5; Output = "Buzz" };
        { DivisibleBy = 3; Output = "Fizz" };
    ]
```

The last piece is a function which refers to that data instead of the hard-coded `x % 15`, `x % 3`, etc. cases.

```fsharp
let fizzBuzz x =
    let rec inner rest =
        // We can pattern-match lists and records
        match rest with
        | [] -> string x
        | { DivisibleBy = div; Output = output } :: tail ->
            match x % div with
            | 0 -> output
            | _ -> inner tail
    
    inner cases
```

It is completely overkill, but it shows how records can be consumed by F# logic.  We use recursion to loop through the cases, then we use deconstruction/pattern matching, first to check if there are more cases to apply, and then to match the `x` value to one of the `Output` properties.
