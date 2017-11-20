# Safety in Rust

## Introduction

Rust is a systems programming language which attempts to be a "safe, concurrent,
and practical language."  Rust uses "Zero-Cost" abstractions to accomplish these
aspirations.  By catching all memory and concurrency pitfalls early on, in the
compilation step by enforcing strong rules on data access and manipulation, as
well as fostering good development choices through idiomatic practices, the
language re-enforces safety without the cost associated with traditional
mechanisms.  The root of the memory safety abstractions revolve around a fairly
simple set of rules.  Rust enforces an ownership system which includes borrowing
to allow for guaranteed memory safety, which allows for all compiled programs
to be free of data race conditions, as well as memory access errors.

## Language Security Features

### Immutability by Default

Variables within Rust by default are immutable.  By defaulting variables to
being immutable, the Rust language forces the developer to really think about
if a variable needs to be modified in place.

### Ownership

Within Rust, variable bindings have a concept of [ownership][ownership] over
what they are bound to.  Logically when an entity is *bound* to a label, or
variable, when the variable goes out of scope, Rust will free the *bound*
resource.

By enforcing this rule very strictly, every time a variable goes out of scope
it's bound resource is deterministically freed.  This protection allows us the
language to guard against common errors such as double free errors which are
common to other languages.


#### Ownership Rules

In addition to the deterministic freeing of bound resources, entities bound
to a variable must follow the following rules:

1. There can be zero or more immutable references
2. There can be exactly one reference which is mutable

This means that you can either have as many immutable references to a variable
as you want, OR you can have exactly one mutable reference ONLY.  These rules
enforced within the compiler allow guaranteed data race condition protections.

Data races occur when the following are true:
1. Two or more pointers to the same resource
2. At least one pointer is written
3. There is no synchronization

As demonstrated by the rules, there can only ever be exactly one pointer to
a resource at any time which is mutable.  This check is performed at compile
time.  Since you cannot have an immutable pointer and a mutable pointer to
the same variable binding at the same time there is no chance of a data race.

### Borrowing

Given that there are rules about only having one mutable pointer to a variable
binding at a time, rust employs a concept of [borrowing][borrowing].
Effectively, when you want to have a block or function manipulate an existing
variable binding rust allows the developer the ability to lend the variable
binding to the new block or function through the use of references.  Example
seen below, pulled from the rust book:

{%highlight rust%}
fn main() {
    let mut x = 5;
    let y = &mut x;    // -+ &mut borrow of x starts here
    *y += 1;           // |
    println!("{}", x); // -+ try to borrow x here, but y is still in scope
}
{%endhighlight%}

The above fails because we have borrowed the reference from x, and given the
bound value to y.  Based on the rules we can only have one mutable reference
so the compiler will fail referencing the fact that we are trying to use x,
even though the reference is borrowed to y.  A more legitimate example would be
the below:

{%highlight rust%}
fn main() {
    let mut x = 5;
    {
        let y = &mut x;    // -+ &mut borrow of x starts here
        *y += 1;           //  |
    }                      // -+ scope of y ends, borrowing ends
    println!("{}", x);     // -+ try to borrow x here, success
}
{%endhighlight%}

Of course the basic ownership rules still apply, (zero or more immutable, only
one mutable reference).  Borrowing tells the compiler to not deterministically
deallocate the resource when it goes out of scope, as it is just borrowed from
the calling scope, and will be deallocated later on.

Borrowing prevents a number of common issues, such as iterator invalidation and
use after free errors.  Iterator invalidation happens when you mutate an array
or vector while you are iterating over said array or vector.  After free errors
happen when you try to access a variable whose binding had been freed.  Since
borrowing doesn't free the variable binding, this is not possible in rust.

### Lifetimes

As mentioned above, in Rust variable bindings all have a [lifetime][lifetime].
Usually this lifetime is the length of the scope where the variable was defined.
But what happens when you need to use variables from different scopes in the
same function call that returns something?  Which scope lifetime do you need to
use?

In Rust you can define explicit lifetimes for function arguments, and struct
parameters, which allow the developer to explicitly tell the compiler that this
variable binding exists for a given lifetime.  Example below:

{%highlight rust%}
fn add(a: &int32, b: &int32) -> &int32 {
    // ...
}

let a = 1;

let sum;
{
    let b = 2;        // -+ `b` comes into scope.
    sum = add(a, b);  //  |
}                     // -+ `b` goes out of scope.
println!("{}", sum);
{%endhighlight%}

In the above example, what if the result of `add` still references `b`?  at our
`println!` we have the potential for a use after free, or dangling pointer
situation.

To allot for these situations Rust employs explicit lifetime mappings for
function inputs and outputs, so Rust can make sure that a lifetime is appropriate
in complicated situations.  Take the below correction:

{%highlight rust%}

fn add<'a,'b>(a: &'a int32, b: &'b int32) -> &'a int32 {
    // ...
}

let a = 1;

let sum;
{
    let b = 2;        // -+ `b` comes into scope.
    sum = add(a, b);  //  |
}                     // -+ `b` goes out of scope.
println!("{}", sum);

{%endhighlight%}

With the above lifetime annotations the compiler can now tell that the second
argument's scope is no factor to the lifetime of the response, which should
be the same as the lifetime of the first parameter.  By explicitly stating how
long the lifetime of the parameters and results should be, the compiler can
enforce the scoping rules appropriately.


## List of Rust Greatness:

* Prevents Iterator Invalidation with Borrowing
* Prevents Use After Free, dangling pointers with Lifetimes and Borrowing
* Prevents All Data Races with Ownership rules
* Prevents Double Free with Scope Deallocation
* Prevents overwriting variables with immutability by default
* Prevents most Memory Leaks by freeing bound resources after scope


[ownership]: https://doc.rust-lang.org/book/first-edition/ownership.html
[borrowing]: https://doc.rust-lang.org/book/first-edition/references-and-borrowing.html
[lifetimes]: https://doc.rust-lang.org/book/first-edition/lifetimes.html

[concurrency]: https://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html
[memory-safety]: http://theburningmonk.com/2015/05/rust-memory-safety-without-gc/
[smart-pointers]: https://pcwalton.github.io/blog/2013/03/18/an-overview-of-memory-management-in-rust/
[so]: https://stackoverflow.com/questions/36136201/how-does-rust-guarantee-memory-safety-and-prevent-segfaults
