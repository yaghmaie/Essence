---
title: Reuse tests with multiple implementations of a trait in Rust by rstest
description: Never write the same test twice. Use rstest instead to make your tests reusable just like your code!
cover_image: ./reuse_tests_rust_cover.png
tags: 'rust, testing, rstest, tdd'
canonical_url: null
published: true
id: 1752448
date: '2024-02-05T18:34:49Z'
---

Every programming language has its own ways to let us abstract details away, and in [Rust](https://www.rust-lang.org/) it's [traits](https://doc.rust-lang.org/book/ch10-02-traits.html) that give us the ability.
Traits in Rust enable lots of flexibilities such as extending functionalities of a data type or dynamic dispatching.
One of them that we'll be focusing on in this article is expressing common behavior of multiple implementations under the interface defined by a trait.

A perfect example of such implementations is a data store. We expect a data store to have the exact same behavior whether it's storing the data in memory or some advanced highly scalable database in the cloud.

On the one hand, we have traits to help us with coordination between the data stores to implement the common interface defined by a trait. On the other hand, while different implementations share the same trait interface, their internal behavior might differ. Therefore, we need to have a good test coverage in order to ensure that all the different implementations behave consistently from the perspective of their dependents.

To effectively achieve such test coverage without duplicating code or writing error prone boilerplate, we need test fixtures which Rust does not support by default. However, we can use [**rstest**](https://crates.io/crates/rstest) which is a great fixture-based test framework for Rust.

In this article we'll practice test driven development and use **rstest** to accomplish:

* Implementing a data store trait in two different ways
* Maintaining a good test coverage and ensuring different implementations behave the same
* Keeping code and test duplication minimized

TL;DR you can find the code [here on GitLab](https://gitlab.com/yaghmaie/rust-test-reuse).

Now let's get started by creating a new package.

## Getting Started

First of all we need to run a few familiar commands to make a new package for our example and add [**rstest**](https://crates.io/crates/rstest) and [**rstest_reuse**](https://crates.io/crates/rstest_reuse) to the dev dependencies.

**rstest** lets us inject test dependencies (like the data stores we talked about earlier) into different test functions and **rstest_reuse** makes it possible to apply the same test template on multiple test functions.

```sh
$ cargo new rstest_multi_impl_trait 
     Created library `rstest_multi_impl_trait` package
$ cd rstest_multi_impl_trait
$ cargo add rstest rstest_reuse --dev
    Updating crates.io index
      Adding rstest v0.18.2 to dev-dependencies.
             Features:
             + async-timeout
      Adding rstest_reuse v0.6.0 to dev-dependencies.
    Updating crates.io index
```

## Context

Let's say we want to manage some `Fruit`s.

```rust
struct Fruit {
    pub id: usize,
    pub name: String,
}
```

In order to do that, we'll need a `FruitStore` to `add` or `remove` our `Fruit`s for us and also be able to `find` them. Occasionally, we need to know how many `Fruit`s we have in the store . So, the store should be able to `count` them.

```rust
pub trait FruitStore {
    fn count(&self) -> usize;

    fn add(&mut self, fruit: Fruit) -> Result<(), AddError>;

    fn find(&self, id: usize) -> Option<Fruit>;

    fn remove(&mut self, id: usize) -> Result<(), DeleteError>;
}
```

Of course, if we `add` some `Fruit` twice or try to `remove` one that doesn't exist, then we expect an error.

```rust
enum AddError {
    DuplicateId,
}

enum DeleteError {
    IdNotFound,
}
```

## [`VecFruitStore`](https://gitlab.com/yaghmaie/rust-test-reuse/-/commit/62bd47429adb93a25153e8bd4a83536437996cde)

We want to keep things simple and that's why we don't want to use a real database or a cloud storage service to implement our `FruitStore`. Instead we're going to use a `Vec<Fruit>` as our storage backend.

```rust
struct VecFruitStore {
    storage: Vec<Fruit>,
}

impl VecFruitStore {
    fn new() -> Self {
        VecFruitStore {
            storage: Vec::new(),
        }
    }
}
```

And surely `VecFruitStore` has to implement `FruitStore`.

```rust
impl FruitStore for VecFruitStore {
    fn count(&self) -> usize {
        todo!()
    }

    fn add(&mut self, fruit: Fruit) -> Result<(), AddError> {
        todo!()
    }

    fn find(&self, id: usize) -> Option<Fruit> {
        todo!()
    }

    fn remove(&mut self, id: usize) -> Result<(), DeleteError> {
        todo!()
    }
}
```

## [Empty `VecFruitStore`](https://gitlab.com/yaghmaie/rust-test-reuse/-/commit/a688aa9ae2d1d051a963b53af1654e270e5d5446)

It's always a better idea to keep things simple at the start and begin with coding the most simple functions with the most basic inputs. Speaking of data stores, the most simple state of them is surely when they're empty.
Probably the simplest function to test on an empty data store is `count` since it just needs to return `0`. That's where we start then. Needless to say, we're going to write a test before writing any code and follow [**TDD**](https://martinfowler.com/bliki/TestDrivenDevelopment.html) as mentioned before.

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn empty_store_counts_zero() {
        let store = VecFruitStore::new();

        assert_eq!(0, store.count());
    }
}
```

If we run our new test with `cargo test --lib` it fails because `count` is not implemented yet and just includes a placeholder `todo!()`; So, this is our [**Red**](https://martinfowler.com/bliki/TestDrivenDevelopment.html) step:

```text
running 1 test
test tests::empty_store_counts_zero ... FAILED

failures:

---- tests::empty_store_counts_zero stdout ----
thread 'tests::empty_store_counts_zero' panicked at src/lib.rs:41:9:
not yet implemented
```

The easiest change to make the test pass is simply returning `0` from `count` and that's what we do.

```rust
fn count(&self) -> usize {
    0
}
```

```text
running 1 test
test tests::empty_store_counts_zero ... ok
```

The next simple thing to test is trying to `find` a `Fruit` in an empty store. Certainly, an empty store has nothing to `find` for you.

```rust
#[test]
fn empty_store_finds_none() {
    let store = VecFruitStore::new();
    let fruit = store.find(0);

    assert!(fruit.is_none());
}
```

And making it pass

```rust
fn find(&self, id: usize) -> Option<Fruit> {
    None
}
```

```text
running 2 tests
test tests::empty_store_counts_zero ... ok
test tests::empty_store_finds_none ... ok
```

If we `remove` a `Fruit`, we should get a delete error.

```rust
#[test]
fn empty_store_remove_gives_id_not_found_error() {
    let mut store = VecFruitStore::new();
    let result = store.remove(0);

    assert!(matches!(result, Err(DeleteError::IdNotFound)));
}
```

```rust
fn remove(&mut self, id: usize) -> Result<(), DeleteError> {
    Err(DeleteError::IdNotFound)
}
```

```text
running 3 tests
test tests::empty_store_finds_none ... ok
test tests::empty_store_remove_gives_id_not_found_error ... ok
test tests::empty_store_counts_zero ... ok
```

## [Refactoring tests](https://gitlab.com/yaghmaie/rust-test-reuse/-/commit/d55510a5960a014ac7c221b5ca444dd89dde8317)

In the first line of every test we're creating a new `VecFruitStore` right now. This is not ideal because:

1) We have duplication
2) How do we reuse our tests if we had another implementation of `FruitStore`?

To fix these, we need fixture-based tests so it's time for [**rstest**](https://crates.io/crates/rstest) to enter.

```rust
#[cfg(test)]
mod tests {
    use rstest::*;
    ...
```

The next step is to turn our tests from something like:

```rust
#[test]
fn empty_store_counts_zero() {
    let store = VecFruitStore::new();

    assert_eq!(0, store.count());
}
```

To

```rust
#[rstest]
fn empty_store_counts_zero(store: impl FruitStore) {
    assert_eq!(0, store.count());
}
```

Instead of initializing a new `FruitStore` in each test case, let's just receive one from somewhere and focus on the testing part. This way we have a test that doesn't care about what exactly is the `store` that it's evaluating as long is it implements the `FruitStore` trait.

The next question would be: How the tests receive their `store` to run?
Well, [**rstest**](https://crates.io/crates/rstest) makes this is easy. All we need to do is throwing in a fixture function.

```rust
#[fixture]
fn store() -> impl FruitStore {
    VecFruitStore::new()
}
```

Applied to all the tests, the result should be the following:

```rust
#[cfg(test)]
mod tests {
    use crate::{DeleteError, FruitStore, VecFruitStore};
    use rstest::*;

    #[fixture]
    fn store() -> impl FruitStore {
        VecFruitStore::new()
    }

    #[rstest]
    fn empty_store_counts_zero(store: impl FruitStore) {
        assert_eq!(0, store.count());
    }

    #[rstest]
    fn empty_store_finds_none(store: impl FruitStore) {
        let fruit = store.find(0);

        assert!(fruit.is_none());
    }

    #[rstest]
    fn empty_store_remove_gives_id_not_found_error(mut store: impl FruitStore) {
        let result = store.remove(0);

        assert!(matches!(result, Err(DeleteError::IdNotFound)));
    }
}
```

```text
running 3 tests
test tests::empty_store_counts_zero ... ok
test tests::empty_store_remove_gives_id_not_found_error ... ok
test tests::empty_store_finds_none ... ok

```

## [Get ready to add another `FruitStore`](https://gitlab.com/yaghmaie/rust-test-reuse/-/commit/619ad70a5b0161fa4b7b65de67e6da67e50a7bf0)

There's still a catch. If we add another implementation of `FruitStore` like a `HashMapFruitStore` which uses a `HashMap<usize, Fruit>` as storage backend, then how can it be tested by the existing tests? Since the fixture function we wrote earlier only returns a `VecFruitStore`.

Such situations can easily be handled by [**rstest_reuse**](https://crates.io/crates/rstest_reuse). Just remember that you have to add the following code at the **top of the crate** (not the module).

```rust
#[cfg(test)]
use rstest_reuse;
```

First we need to get rid of the fixture function and replace it with a `template` function.

```rust
#[template]
#[rstest]
fn fruit_store(store: impl FruitStore) {}
```

Secondly, we should change the `#[rstest]` on the tests to `#[apply(fruit_store)]`. This way **rstest** can recognize what template has to be applied on which tests.

```rust
#[apply(fruit_store)]
fn empty_store_counts_zero(store: impl FruitStore) {
    assert_eq!(0, store.count());
}
```

Finally, we use `case` on the template to inject our desired `FruitStore`.

```rust
#[template]
#[rstest]
#[case::vec_fruit_store(VecFruitStore::new())]
fn fruit_store(#[case] store: impl FruitStore) {}
```

## [Adding another `FruitStore`](https://gitlab.com/yaghmaie/rust-test-reuse/-/commit/28a1f6a14cdbdeb917f746f564614eec43b7e902)

As you have already guessed adding another `FruitStore` implementation to this test machine is as easy as adding another `case` on the template like `#[case::hashmap_fruit_store(HashMapFruitStore::new())]`.

```rust
#[template]
#[rstest]
#[case::vec_fruit_store(VecFruitStore::new())]
#[case::hashmap_fruit_store(HashMapFruitStore::new())]
fn fruit_store(#[case] store: impl FruitStore) {}
```

But another problem pops up:

```text
running 6 tests
test tests::empty_store_counts_zero::case_1_vec_fruit_store ... ok
test tests::empty_store_counts_zero::case_2_hashmap_fruit_store ... FAILED
test tests::empty_store_finds_none::case_1_vec_fruit_store ... ok
test tests::empty_store_finds_none::case_2_hashmap_fruit_store ... FAILED
test tests::empty_store_remove_gives_id_not_found_error::case_1_vec_fruit_store ... ok
test tests::empty_store_remove_gives_id_not_found_error::case_2_hashmap_fruit_store ... FAILED
```

Imagine if we had 100 test cases, then we had to complete the implementation and get all of them passed in order to get a green build. However, the `case`s also work if on top of the tests. Accordingly, we can add the new `case` one by one on them until all are covered.

```rust
#[apply(fruit_store)]
#[case::hashmap_fruit_store(HashMapFruitStore::new())]
fn empty_store_counts_zero(store: impl FruitStore) {
    assert_eq!(0, store.count());
}
```

## Final words

Nothing can grow our confidence for our work like a great test coverage. Testing should not be something totally separate from our everyday workflow or an afterthought. In this article, we went through an example of implementing a trait of a data store with the same tests with help of [**rstest**](https://crates.io/crates/rstest). I hope you find it useful.

I'm going to leave the code at the current stage so you'd have something to practice on. So please take a copy of this code from [here](https://gitlab.com/yaghmaie/rust-test-reuse) and start adding more test cases and also continue the implementation.
