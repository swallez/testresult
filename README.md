# Test result

[![CI](https://github.com/wiktor-k/testresult/actions/workflows/ci.yml/badge.svg)](https://github.com/wiktor-k/testresult/actions/workflows/ci.yml)
[![Crates.io](https://img.shields.io/crates/v/testresult)](https://crates.io/crates/testresult)
[![Codecov](https://img.shields.io/codecov/c/gh/wiktor-k/testresult)](https://app.codecov.io/gh/wiktor-k/testresult)


Provides `TestResult` type that can be used in tests to avoid
`unwrap`s but at the same time to have precise stacktraces with the
point of failure clearly written.

Consider the following code. It uses `unwrap` so the test failure
stacktrace will informative. Unfortunately it's not as concise as it
could be:

```rust
#[test]
fn it_works() {
   // ...
   std::fs::File::open("this-file-does-not-exist").unwrap();
   // ...
}
```

Improved version of this code uses `Result` and the `?` operator:

```rust
#[test]
fn it_works() -> Result<(), Box<dyn std::error::Error>> {
   // ...
   std::fs::File::open("this-file-does-not-exist")?;
   // ...
   Ok(())
}
```

Running the following code with `RUST_BACKTRACE=1 cargo test` shows
the following stacktrace:

```text
---- tests::it_works stdout ----
thread 'tests::it_works' panicked at 'assertion failed: `(left == right)`
  left: `1`,
  ...
   4: test::assert_test_result
             at /rustc/4b91a6ea7258a947e59c6522cd5898e7c0a6a88f/library/test/src/lib.rs:184:5
   5: testresult::tests::it_works::{{closure}}
             at ./src/lib.rs:52:5
   6: core::ops::function::FnOnce::call_once
             at /rustc/4b91a6ea7258a947e59c6522cd5898e7c0a6a88f/library/core/src/ops/function.rs:248:5
  ...
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

Unfortunately even though the test function location is recorded, the
exact line where the test failure occurred is not present in the
backtrace.

Let's adjust the test result type to use `TestResult`. This is the
only change compared to previous example:

```rust
#[test]
fn it_works() -> TestResult {
   // ...
   std::fs::File::open("this-file-does-not-exist")?;
   // ...
   Ok(())
}
```

Running it again with `RUST_BACKTRACE=1 cargo test` shows more details:

```text
---- tests::it_works stdout ----
thread 'tests::it_works' panicked at 'Error: No such file or directory (os error 2)', src/lib.rs:10:9
  ...
   3: <core::result::Result<T,F> as core::ops::try_trait::FromResidual<core::result::Result<core::convert::Infallible,E>>>::from_residual
             at /rustc/4b91a6ea7258a947e59c6522cd5898e7c0a6a88f/library/core/src/result.rs:2125:27
   4: testresult::tests::it_works
             at ./src/lib.rs:53:9     // <-----
   5: testresult::tests::it_works::{{closure}}
             at ./src/lib.rs:52:5
   6: core::ops::function::FnOnce::call_once
             at /rustc/4b91a6ea7258a947e59c6522cd5898e7c0a6a88f/library/core/src/ops/function.rs:248:5
  ...             
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

The advantages of using `TestResult`:
  - exact failure line is present in the backtrace,
  - the underlying error message is present in the test failure,
  - the signature of the test result is simpler.
