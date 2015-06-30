---
layout: post
title: "What is a block chain?"
date: 2015-06-30 20:00:00
categories: block chain
---

Everyone has heard by now of Bitcoin, I am sure.  Bitcoin is a distributed 
peer to peer currency, that is extremely robust, which uses mathematics to 
prove transactions.  I am not going to get into how Bitcoin works, but rather
try to explain the mechanism by which Bitcoin validates transactions, the 
block chain.

## What is a block chain?

A block chain is exactly what it sounds like, a chain of blocks.  Much like a 
chain, all of the blocks are linked together in order.  The block chain starts
at an initialization, or genesis, block and is chained to each block in order
of block validation.  What makes this chain order provable is each block being
added to the block chain is cryptographically hashed with the contents of the 
transaction and the hash of the previous block, thereby linking the previous 
block and the current block in the block chain.  Validation of any given block
can be performed by hashing the previous block with the current block, and then 
hashing the previous block with the block before it, all the way down the chain
until the initialization block.  If any block has been altered, omitted or tampered
with the validation of the entire block chain will fail.

This is a great way to provably show sequence of transactions, and prevent 
malicious tampering of blocks within the chain.

## What is a block?

A block is a grouping of transactions.  When transactions are performed they are
put together in a block.  Each transaction is cryptographically hashed, then 
grouped together as leaves in a binary tree.  These leaves are taken, and 
grouped into pairs, whereby each of their individual hash's are hashed together
forming their parent node.  Each parent node is then hashed with another parent 
node until a root node is created in the tree.  This form of cryptographic hash
tree is called a Merkle Tree.  The Merkle Tree root node is a validation of 
all of the transactions that are within a block.

If any of the transactions within the block are tampered with, the Merkle Root
will no longer validate the block, and the end user will know something isn't 
right with this block.  A provably tamper proof system.

## What is a Transaction?

A transaction consists of, at is primitive, a sender, recipient, and an amount.
Every sender/recipient must have a public/private key generated.  The sender 
creates a transaction to the recipient, and then digitally signs that transaction, 
stating publicly and provably that the sender authorized that transaction to the 
recipient.  This is like signing a contract, except much more provable, as only
someone in control of the sender's private key will be able to perform this task.

When a transaction is created, it is then broadcast to the Bitcoin peer to peer 
network where the transaction is put into a block as explained above, and then 
the block is appended to the block chain, forever more linking it to all previous
and future transactions.

## Who cares?

Well, for one, the virtual currency market cares about validation of 
transactions.  But given this framework, this type of chaining of blocks and
transactions is more widely scoped.  I have heard of a few places that were
looking into using a block chain for voting.  In my opinion this would be a very
good mechanism for voting, except that any vote by anyone would be publicly 
verifiable... Which some people might not want their identity linked to whom
they feel like voting for at a given election.

Basically any immutable, secure and verifiable ledger could benefit from the
work that has been put into the cryptocurrency realm.

