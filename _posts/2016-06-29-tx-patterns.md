---
layout: post
title: Asynchronous transactional patterns 
date: 2016-06-29
type: post
published: true
status: publish
comments: true
categories:
- Technology
tags:
- databases
meta:
  _edit_last: '1'
author:
  login: jake
  email: jakewins@gmail.com
  display_name: jake
  first_name: ''
  last_name: ''
redirect_to: 'https://tech.davis-hansson.com/p/tx-patterns/'
---

Transactions are not very hip anymore - so unhip, in fact, that people started building databases without them.
Alas, as people who decided to try those databases found out, transactions remain a fundamental aspect of applications that don't break horrifically. 

Except, it turns out that even if you *have* transactions, it's surprisingly easy to shoot yourself in the foot.

<!--more-->

## 0. Overview

This post is about using transactions in asynchronous APIs, specifically when using Promises in javascript. However, for illustratitive purposes, it'll first go through anti-patterns and safe patterns of use in blocking code.

Best of all, I'll do this using a test harness that verifies a given pattern is safe. I want to use the word "proves", but I don't feel comfortable enough in my knowledge of academia to use that word. Lets just say that if Mike Burns were still my friend, he'd be proud.

The full source code backing this post can be found on [Github](https://github.com/jakewins/transactional-patterns).

## 1. The Problem

Imagine we have a use case like this:

    begin transaction
    if domain logic is true:
      commit transaction
    else:
      rollback transaction

The implementation must commit or roll back exactly once - but never do both - and if domain logic is true, it must commit. The problem is that any of the actions may fail, the pattern we choose needs to handle those failures without violating the rules.

To test this, we wrote a tiny test harness that can take a pattern, and tell you if it will violate any of the rules.
We do this first by definining all the available actions, and their possible outcomes. For the blocking case, one action is `begin()`, and it has two possible outcomes - either it succeeds or it throws an exception:

```javascript
let begin = action('begin', FAIL, SUCCEED);

function FAIL() {
  throw new Error("Induced failure.");
}

function SUCCEED() {

}
```

Then we define a set of valid outcomes, or valid sequences of actions. If the code pattern we're testing invokes actions in a sequence we haven't explicitly said is valid, the pattern fails the test.

For the blocking use case, one of the rules is that if domain logic fails, it must roll back. We define this by writing it up as a valid sequence of actions:

```javascript
let valid_sequences = [
  [[begin, SUCCEED], [business_logic, FAIL], [rollback]],
];
```

Then we give the available actions, the list of valid sequences and the pattern to test to the harness, and it will tell us if any combination of action outcomes causes the pattern to perform an invalid sequence.

```javascript
test_correctness(actions, valid_sequences, pattern)
```

If a scenario is found where the pattern fails validation, we get a description like this:

    Pattern failed validation
      Scenario: begin -> SUCCEED
                business logic -> FAIL 
                commit -> FAIL
                rollback -> FAIL
      Sequence: begin, business logic

You can find the source code for the tester [here](https://github.com/jakewins/transactional-patterns/blob/master/src/harness.js#L10).

## 2. Blocking patterns

Remembering our use case:

    begin transaction
    if domain logic is true:
      commit transaction
    else:
      rollback transaction

### A naive approach

Lets try just implementing the use case verbatim.

```javascript
begin();
if( business_logic() ) {
  commit();
} else {
  rollback();
}
```

Unfortunately, this doesn't work, says our test. Four different cases cause bad outcomes, all variants of the same case:

    Pattern failed validation
      Scenario: begin -> SUCCEED
                business logic -> FAIL
                commit -> FAIL
                rollback -> FAIL
      Sequence: begin, business logic

Note how the "Sequence" section above is just `begin, business logic`, no commit or rollback. In the `Scenario` section, we can see that the `business logic` action is set to fail. 

Looking at the code, if the business logic throws an exception, the code never calls commit or rollback, leading to a leaked transaction.

### Handling exceptions

Fine, so we add a try/catch then.

```javascript
begin();
try {
  if( business_logic() ) {
    commit();
  } else {
    rollback();
  }
} catch( e ) {
  rollback();
}
```

Again, no dice, four cases had bad outcomes. Each is one of two variants:

      Pattern failed validation
        Scenario: begin -> SUCCEED
                  business logic -> RETURN_TRUE
                  commit -> FAIL
                  rollback -> FAIL

        Sequence: begin, business logic, commit, rollback

      Pattern failed validation
        Scenario: begin -> SUCCEED
                  business logic -> RETURN_FALSE
                  commit -> FAIL
                  rollback -> FAIL
        Sequence: begin, business logic, rollback, rollback


That is - either this pattern calls rollback twice, or it calls rollback after it's called commit. 
This is better, but with most databases the second call to rollback would've caused an error, meaning this pattern does not elegantly handle failure and could make it hard to find the root cause. 

### A correct pattern

One common and correct approach is this:

```javascript
success = false;
begin();
try {
  if( business_logic() ) {
    success = true;
  }
} finally {
  if(success) {
    commit();
  } else {
    rollback();
  }
}
```

This passes all permutations without breaking the rules, meaning it won't leak transactions and it won't obfuscate errors by causing secondary errors. However, it is a bit verbose. 

### A correct, and less verbose, pattern

This is why Neo's legacy Java API, and now all our blocking Driver APIs, don't expose `commit` and `rollback` on the transaction interface. Instead, the pattern we just looked at is enshrined in the API.

```javascript
let tx = begin();
try {
  if( business_logic() ) {
    tx.success();
  }
} finally {
  tx.finish();
}
```

Or, if you are using a language with context handlers:

```python
with begin() as tx:
  if business_logic():
    tx.success()
```

```java
try(Transaction tx = begin()) {
  if(business_logic()) {
    tx.success();
  }
}
```

## 3. Asynchronous patterns

The asynchronous case more complex. Each action may now fail in *two* ways: it can either immediately throw an exception, or the promise it returns can be rejected. 

We first tried the pattern another vendor recommends in their documentation:

```javascript
begin()
  .then(business_logic)
  .then((outcome) => {
    if(outcome) {
      commit();
    } else {
      throw Error("Exception to trigger rollback.")
    }
  })
  .catch((error) => rollback());
```

Unfortunately, this suffers from the "secondary failure" problem; if `commit()` fails, this pattern will then trigger `rollback()`, which will obscure the first failure:

    Pattern failed validation
      Scenario: begin -> SUCCEED
                commit -> THROW
                rollback -> SUCCEED
                business logic -> RETURN_TRUE
      Sequence: begin, business logic, commit, rollback

### A correct pattern

The Promise API seems unclear about whether it supports `finally`. Some implementations do, others don't. In any case, one way to solve our whoes is to leverage a Promise API that supports `finally`. Then we can run an adapted version of our blocking pattern:

```javascript
let success = false;
begin()
  .then(business_logic)
  .then((outcome) => {
    if(outcome) {
      success = true;
    }
  })
  .finally(() => {
    if(success) {
      commit();
    } else {
      rollback();
    }
  });
```

If you don't have `finally`, you can achieve the same thing by chaining `catch` and `then`:

```javascript
let success = false, error;
begin()
  .then(business_logic)
  .then((outcome) => {
    if(outcome) {
      success = true;
    }
  })
  .catch((e) => error = e)
  .then(() => {
    if(success) {
      commit(); 
    } else { 
      rollback();
    }

    if(error) { throw error; }
  });
```

Phew. That's getting mighty long.

### A correct, and less verbose, pattern

Part of designing APIs like this is to make the right thing be the obvious thing to do. While the patterns above are correct, they are not obvious - in fact, they are horrendously obscure and brittle.

Javascripts excellent support for closures can be used almost the same way the `try-with-resources` or `with` examples I showed from Java and Python further up. Scratching our heads a bit, it seems there's at least one pattern where that can be safely used with Promises:

```javascript
transactionally((tx) => {
  return business_logic().then((outcome) => {
    if(outcome) {
      tx.success();
    }
  });
});
```

If you're interested in the `transactionally` function, it's defined [here](https://github.com/jakewins/transactional-patterns/blob/master/src/cases/promise/patterns.js#L74).

This has one important caveat: the user *must* have that return in there, or the `transactionally` sugar can't know when to wrap the transaction up. However, we can guard against this by mandating that something is returned.

It's not quite as terse as the blocking code examples, but it's reasonably simple, and rather hard to use incorrectly. 

In order to support more advanced and specialized use cases, we'll keep the raw `commit` and `rollback` functions available in the Javascript driver today, and instead propose to augment the driver with this pattern, and to make this the default in our documentation.

## Finally

All the code for this is available [here](https://github.com/jakewins/transactional-patterns).
If you think you've got a pattern that's easier, faster, smarter - clone the repo and see if it passes the test :)

P.S.: If anyone knows how to make the stupid "related posts" thing in Jekyll render posts that are actually related, let me know.
