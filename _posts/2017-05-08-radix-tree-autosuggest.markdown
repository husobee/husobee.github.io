---
layout: post
title: "Simple Auto-complete"
date: 2017-05-08 08:00:00
categories: auto-complete radix
---

A month or two back I decided I wanted to implement a simple auto-suggest
micro-service.  I had always admired the auto-suggest and wanted to know how to
make one.

## Structure

I decided that the best way to make an autosuggestion data structure was to
start with a data structure that works off of prefixes.  This was chosen because
as you type, you should get suggestions, so we will want to figure out what the
user wants based on a prefix of the string they are wanting to get.

Luckily for us, we can use a prefix trie, or radix tree, to accomplish this.
Much in the same way my [golang url router][vestigo] works.  Below you can see
the brunt of the auto-suggest engine

```go
go func() {
	for {
		select {
		case r := <-retrieve:
			// retrieve case
			var result = result{
				Terms: []Term{},
				Err:   nil,
			}
			// walk the tree beginning at the prefix we are given
			// and report back all the terms which are useful
			tree.WalkPrefix(r.Key, func(s string, v interface{}) bool {
				result.Terms = append(result.Terms, Term{s, v})
				return false
			})
			r.Result <- result
		case t := <-insert:
			// insertion case
			if _, ok := tree.Insert(t.Key, t.Value); ok {
				t.Err <- nil
			}
			t.Err <- errors.New("failed to insert into structure")
		case <-quit:
			// quit our gardener
			return
		}
	}
}()
```

When we get a message on the retrieve channel, all we need to do is walk that
given prefix.  This will result in a sub-tree which contains all possible matches
that you have in your radix tree for that given prefix.  And quite quickly too.

Insertion is fairly trivial as well using [go-radix][go-radix] library.  Looking
at the performance we can see that the insertion and retrieval of random uuids
comes out to about 80K ns/op for insertion and 20K ns/op for retrieval, which is
very good:

```text
$ go test ./... -bench Benchmark
BenchmarkInsertion-8       20000             81414 ns/op
BenchmarkRetrieval-8       50000             20613 ns/op
PASS
ok      github.com/husobee/suggest/data 20.512s
```

## Improvements

I don't know about you, but I am a terrible speller.  I constantly embarrass
myself with my spelling mistakes.  In this type of auto-suggest, using a plain
prefix tree, you have to always be correct as you type.  This is because, if the
prefix doesn't match, i.e. misspelled, you will not be able to find the sub-tree
you want to traverse.

Enter BK Trees.  BK trees take the Levenshtein Distance of a word to another 
word into account, and creates a sub tree of closest matching words.  This is
handy for words that are some number of letters different.

To improve the auto-suggest algorithm above, we could use a [bktree][bktree] to 
calculate the difference of a word and then feed all of the n-closest matches 
back into the radix tree implementation.  This will work pretty well for finding
closely misspelled words.

Hope this was helpful.

[github]: https://github.com/husobee/suggest
[vestigo]: https://github.com/husobee/vestigo
[go-radix]: https://github.com/armon/go-radix
[bktree]: https://github.com/gansidui/bktree
