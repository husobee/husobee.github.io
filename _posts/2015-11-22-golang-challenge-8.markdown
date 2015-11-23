---
layout: post
title: "sudoku solver"
date: 2015-11-22 08:00:00
categories: golang challenge sudoku
---

## sudoku solver

This weekend I decided to take the [golang challenge #8][gochal8] which is to implement a 
sudoku solver.  Sudoku is a fairly straight forward puzzle. Given a 9x9 matrix,
with some positions in the matrix filled out, your task is to solve the remainder
of the matrix with the following rules:

1. Each position in the matrix must contain 1-9
2. Rows in the matrix must have unique numbers
3. Columns in the matrix must have unique numbers
4. "Unit Squares" consisting of 3x3 sub matrices must have unique numbers

The challenge specified the "input" format of a puzzle to be solved as below:

{% highlight text %}
1 _ 3 _ _ 6 _ 8 _
_ 5 _ _ 8 _ 1 2 _
7 _ 9 1 _ 3 _ 5 6
_ 3 _ _ 6 7 _ 9 _
5 _ 7 8 _ _ _ 3 _
8 _ 1 _ 3 _ 5 _ 7
_ 4 _ _ 7 8 _ 1 _
6 _ 8 _ _ 2 _ 4 _
_ 1 2 _ 4 5 _ 7 8
{% endhighlight %}

You can see from the above input sample, the input consists of single digit
unsigned integers, underscores, spaces and newline/linefeed ascii characters.  
The required output of the application is the filled in completed puzzle, OR
something informing the user that the puzzle isn't solvable.

## implementation process

Of course, I started this challenge with the inputs/outputs.  I figured, what 
better way to have some data to play with, if I am able to correctly parse the 
stdin input and write to the stdout output.  This brought me to create a custom 
scanner in golang.

The `bufio` package in the stdlib for golang is pretty neat, as it will allow 
the implementer the ability to make custom `Split` functions in order to create
your own scanner.  Below is my [puzzleScanSplit][scansplit] function that I feed into the 
bufio scanner, to parse and validate the input is correct:

{% highlight go %}

// puzzleScanSplit - This is the customer scanner split function used to both
// parse and validate the stdin representation of the puzzle
func puzzleScanSplit(data []byte, atEOF bool) (advance int, token []byte, err error) {
	// based on scanlines, we will validate each line at a time
	advance, token, err = bufio.ScanLines(data, atEOF)
	if err == nil && token != nil {
		if len(token) != maxInputRowLength {
			// line length is incorrect, error
			err = ErrParseInvalidLineLength
			return
		}
		// check that each line is correct format
		for i, b := range token {
			if isEvenNumber(i) {
				// even, should be either a Number or Blank
				if !isNumber(b) && !isBlank(b) {
					//error
					err = ErrParseInvalidCharacter
					return
				}
			} else {
				// odd, should be space
				if !isSpace(b) {
					err = ErrParseInvalidCharacter
					return
				}
			}
		}
	}
	return
}
{% endhighlight %}

In the above code, you can see we are basically extending the `bufio.ScanLines` 
split function for the `bufio.Scanner` and performing validation checks on the
input, for example, making sure there is either a number or underscore on even 
characters, as well as making sure spaces are in the right place, and the row
length is correct.

This function is used in [ParsePuzzle][parse] to create a new puzzle from an `io.Reader`
which in our applications case is `os.Stdin`, but awesomely, for unit tests, we 
are using `strings.NewReaders` as our `io.Reader` of choice.

`io.Reader` in golang is an interface, so I felt like this ParsePuzzle signature
would be really flexible.  In this snipit you can see we are actually creating a
"Puzzle" type:

{% highlight go %}
	// scan one row at a tim
	for scanner.Scan() {
		if rowCount > 8 {
			// we have exceeded the allowable number of rows, report invalid
			// row count
			return p, ErrParseInvalidRowCount
		}
		// grab the token bytes
		token := scanner.Bytes()
		posCount := 0
		for i := 0; i < len(token); i += 2 {
			// since we have already validated the correctness
			// of the puzzle input, we will skip to every other
			// value from the line
			var value uint8
			if token[i] != underscore {
				// if the value is not an underscore, set to
				// the number value of the ascii token
				value, _ = asciiToNumber(token[i])
			}
			// populate the value in the matrix
			p.p[rowCount][posCount] = value
			posCount++
		}
		rowCount++
	}

{% endhighlight %}

in the above we are using our scanner with our custom splitter
on the reader, and validating the number of rows is correct.  We are then
taking the ascii value from the scanner, and converting it into it's 
uint8 representation.  Underscores are replaced with unit8 0 values.

## output

Part of the requirement was to output a fully populated solution, so to those
ends I went about making a `Dump(io.Writer)` method on the Puzzle struct as
seen [here][dump].  What I really enjoy about golang is what interfaces bring
to the table.  Since I am taking a writer as my function input, I could care
less about it's implementation, I know for sure it will have a `Write` method,
which I can use to write to it.  Much like for the inputs, I am able to use 
`os.Stdout` in the application, and in unit tests I can use `bytes.Buffer` or 
any number of things that implement the writer interface.  Pretty sweet.

## teh sudoku

So, now for the actual solving of the sudoku.... well I am partial to backtracking,
what can I say, I do it a lot ;)

Backtracking is a means to solve puzzles where you make guesses, and see how they
will pan out in the long run.  As you can see the implementation of a simple 
sudoku solver is literally the [smallest segment][solver] of code in the entire project.

Walking through this code, we perform the following steps:

* Iterate over all the rows
  * Iterate over all the columns
    * Iterate over all the possible values, if the position we are on is blank
    * Check if that possible value is allowable ( check row, column, unit square for uniqueness)
    * If allowable, start this whole process over again, with that position filled in
    * If no possible values are allowed, this is a dead branch, return to previous caller

Backtracking is merely a depth first traversal of a tree, but what I love about
recursion, is that the application stack (the function calls) builds the tree, 
and the logic within the function tells you which branches you can skip, and prune.

## results

My solver works for solving sudoku.  Below is what I have been getting when 
running benchmark tests for the solve on a simple sudoku puzzle:

{% highlight bash %}
[husobee@localhost sudoku]$ go test -bench Solve 
PASS
BenchmarkSolvePuzzle       50000             62066 ns/op
BenchmarkSolveHardPuzzle              50          31588436 ns/op
ok      github.com/husobee/sudoku       5.344s
{% endhighlight %}

[here][husosudoku] is the source code of this implementation.

[gochal8]: http://golang-challenge.com/go-challenge8/
[scansplit]: https://github.com/husobee/sudoku/blob/master/utils.go#L51
[parse]: https://github.com/husobee/sudoku/blob/master/puzzle.go#L45
[dump]: https://github.com/husobee/sudoku/blob/master/puzzle.go#L24-L42
[solver]: https://github.com/husobee/sudoku/blob/master/puzzle.go#L150-L191
[husosudoku]: https://github.com/husobee/sudoku
