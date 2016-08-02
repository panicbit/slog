# slog-rs - Structured, composable logging for [Rust][rust]

<p align="center">
  <a href="https://travis-ci.org/dpc/slog-rs">
      <img src="https://img.shields.io/travis/dpc/slog-rs/master.svg?style=flat-square" alt="Travis CI Build Status">
  </a>
  <a href="https://crates.io/crates/slog">
      <img src="http://meritbadge.herokuapp.com/slog?style=flat-square" alt="crates.io">
  </a>
  <a href="https://gitter.im/dpc/dpc">
      <img src="https://img.shields.io/badge/GITTER-join%20chat-green.svg?style=flat-square" alt="Gitter Chat">
  </a>
  <br>
  <strong><a href="//dpc.github.io/slog-rs/">Documentation</a></strong>
</p>

## Introduction

Inspired by [log15] for Go.

### Features

* easy to use
* hierarchical loggers
* lazily evaluated values
* modular and extensible
	* small core create with multiple addon crates (`./crates/`) - compile only
	what you're actually using
* standard `log` crate logging support (`slog-stdlog` crate)
* drains & output formatting
	* filtering
	* multiple outputs
	* asynchronous IO writing
	* terminal output, with color support (`slog-term` crate)
	* Json (`slog-json` crate)
		* Bunyan (`slog-bunyan` crate)
	* syslog (`slog-syslog` crate)
	* support for first class custom ones

### Advantages over `log` crate

* **composable** and **extensible** logging drains - You can easily log messages
  to different destinations, in different formats, using different filtering
  rules. Building new drains and new output formats is very easy.
* drains are **run-time swappable** - Send a signal to your program and start
  logging everything to a file for run-time production debugging. Send another
  one when you're done to return to normal operation. Build your own flexible
  scenarios easily.
* **non-global** - Hierarchical loggers carry information about context of
  logging. When logging an error condition, you want to know which resource was
  being handled, on which instance of your service, using which source code
  build, talking with what peer, etc. In standard `log` you would have to repeat
  this information in every log statement. In `slog` it will happen
  automatically.
* both **human and machine readable** - By keeping the key-value data format,
  meaning of logging data is preserved. Dump your logging to a JSON file, and
  send it to your data-mining system for further analysis.
* **lazy evaluation** and **asynchronous IO** included

### Terminal output example

![slog-rs terminal output](http://i.imgur.com/IUe80gU.png)

## Using & help

### Code snippet

``` rust
fn main() {
    // Create a new drain hierarchy, for the need of your program.
    // Choose from collection of existing drains, or write your own
    // `struct`-s implementing `Drain` trait.
    let drain = slog_term::async_stderr();

    // `AtomicSwitch` is a drain that wraps other drain and allows to change
    // it atomically in runtime.
    let ctrl = AtomicSwitchCtrl::new(drain);
    let drain = ctrl.drain();

    // Turn a drain into new group of loggers, sharing that drain.
    //
    // Note `o!` macro for more natural `OwnedKeyValue` sequence building.
    let root = drain.into_logger(o!("version" => VERSION, "build-id" => "8dfljdf"));

    // Build logging context as data becomes available.
    //
    // Create child loggers from existing ones. Children clone `key: value`
    // pairs from their parents.
    let log = root.new(o!("child" => 1));

    // Closures can be used for values that change at runtime.
    // Data captured by the closure needs to be `Send+Sync`.
    let counter = Arc::new(AtomicUsize::new(0));
    let log = log.new(o!("counter" => {
        let counter = counter.clone();
        // Note the `move` to capture `counter`,
        // and unfortunate `|_ : &_|` that helps
        // current `rustc` limitations. In the future,
        // a `|_|` could work.
        move |_ : &RecordInfo| { counter.load(SeqCst)}
    }));

    // Loggers  can be cloned, passed between threads and stored without hassle.
    let join = thread::spawn({
        let log = log.clone();
        move || {

            info!(log, "before-fetch-add"); // counter == 0
            counter.fetch_add(1, SeqCst);
            info!(log, "after-fetch-add"); // counter == 1

            // `AtomicSwitch` drain can swap it's interior atomically (race-free).
            ctrl.set(
                // drains are composable and reusable
                drain::filter_level(
                    Level::Info,
                    drain::async_stream(
                        std::io::stderr(),
                        // multiple outputs formats are supported
                        slog_json::new(),
                    ),
                ),
            );

            // Closures can be used for lazy evaluation:
            // This `slow_fib` won't be evaluated, as the current drain discards
            // "trace" level logging records.
            debug!(log, "debug", "lazy-closure" => |_ : &RecordInfo| slow_fib(40));

            info!(log, "subthread", "stage" => "start");
            thread::sleep(Duration::new(1, 0));
            info!(log, "subthread", "stage" => "end");
        }
    });

    join.join().unwrap();
}
```

See `examples/features.rs` for full code.


Read [Documentation](//dpc.github.io/slog-rs/) for details and features.

If you want to say hi, or need help use [#dpc gitter.im][dpc gitter].

To report a bug or ask for features use [github issues][issues].

[rust]: http://rust-lang.org
[dpc gitter]: https://gitter.im/dpc/dpc
[issues]: //github.com/dpc/slog-rs/issues
[log15]: //github.com/inconshreveable/log15

### Building & running

If you need to install Rust (come on, you should have done that long time ago!), use [rustup][rustup].

[rustup]: https://www.rustup.rs

#### In your project

In Cargo.toml:

```
[dependencies]
slog = "*"
```

In your `main.rs`:

```
#[macro_use]
extern crate slog;
```