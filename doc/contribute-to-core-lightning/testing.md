---
title: "Testing"
slug: "testing"
excerpt: "Understand the testing and code review practices."
hidden: false
createdAt: "2022-12-09T09:58:21.295Z"
updatedAt: "2023-07-13T05:21:39.220Z"
---
# Testing

Tests are run with: `make check [flags]` where the pertinent flags are:

```shell
DEVELOPER=[0|1] - developer mode increases test coverage
VALGRIND=[0|1]  - detects memory leaks during test execution but adds a significant delay
PYTEST_PAR=n    - runs pytests in parallel
```

A modern desktop can build and run through all the tests in a couple of minutes with:

```shell Shell
make -j12 full-check PYTEST_PAR=24 DEVELOPER=1 VALGRIND=0
```

Adjust `-j` and `PYTEST_PAR` accordingly for your hardware.

There are four kinds of tests:

- **source tests** - run by `make check-source`, looks for whitespace, header order, and checks formatted quotes from BOLTs if BOLTDIR exists.

- **unit tests** - standalone programs that can be run individually. You can also run all of the unit tests with `make check-units`. They are `run-*.c` files in test/ subdirectories used to test routines inside C source files.

  You should insert the lines when implementing a unit test:

  `/* AUTOGENERATED MOCKS START */`

  `/* AUTOGENERATED MOCKS END */`

  and `make update-mocks` will automatically generate stub functions which will allow you to link (and conveniently crash if they're called).

- **blackbox tests** - These tests setup a mini-regtest environment and test lightningd as a whole.  They can be run individually:

  `PYTHONPATH=contrib/pylightning:contrib/pyln-client:contrib/pyln-testing:contrib/pyln-proto py.test -v tests/`

  You can also append `-k TESTNAME` to run a single test.  Environment variables `DEBUG_SUBD=<subdaemon>` and `TIMEOUT=<seconds>` can be useful for debugging subdaemons on individual tests.

- **pylightning tests** - will check contrib pylightning for codestyle and run the tests in `contrib/pylightning/tests` afterwards:

  `make check-python`

Our Github Actions instance (see `.github/workflows/*.yml`) runs all these for each pull request.

#### Additional Environment Variables

```text
TEST_CHECK_DBSTMTS=[0|1]            - When running blackbox tests, this will
                                      load a plugin that logs all compiled
                                      and expanded database statements.
                                      Note: Only SQLite3.
TEST_DB_PROVIDER=[sqlite3|postgres] - Selects the database to use when running
                                      blackbox tests.
EXPERIMENTAL_DUAL_FUND=[0|1]        - Enable dual-funding tests.
EXPERIMENTAL_SPLICING=[0|1]         - Enable splicing tests.
```

#### Troubleshooting

##### Valgrind complains about code we don't control

Sometimes `valgrind` will complain about code we do not control ourselves, either because it's in a library we use or it's a false positive. There are generally three ways to address these issues (in descending order of preference):

1. Add a suppression for the one specific call that is causing the issue. Upon finding an issue `valgrind` is instructed in the testing framework to print filters that'd match the issue. These can be added to the suppressions file under `tests/valgrind-suppressions.txt` in order to explicitly skip reporting these in future. This is preferred over the other solutions since it only disables reporting selectively for things that were manually checked. See the [valgrind docs](https://valgrind.org/docs/manual/manual-core.html#manual-core.suppress) for details.
2. Add the process that `valgrind` is complaining about to the `--trace-children-skip` argument in `pyln-testing`. This is used in cases of full binaries not being under our control, such as the `python3` interpreter used in tests that run plugins. Do not use this for binaries that are compiled from our code, as it tends to mask real issues.
3. Mark the test as skipped if running under `valgrind`. It's mostly used to skip tests that otherwise would take considerably too long to test on CI. We discourage this for suppressions, since it is a very blunt tool.

# Fuzz testing

Core Lightning currently supports coverage-guided fuzz testing using [LLVM's libfuzzer](https://www.llvm.org/docs/LibFuzzer.html) when built with `clang`.

The goal of fuzzing is to generate mutated -and often unexpected- inputs (`seed`s) to pass to (parts of) a program (`target`) in order to make sure the codepaths used:

- do not crash
- are valid (if combined with sanitizers)  
  The generated seeds can be stored and form a `corpus`, which we try to optimise (don't store two seeds that lead to the same codepath).

For more info about fuzzing see [here](https://github.com/google/fuzzing/tree/master/docs), and for more about `libfuzzer` in particular see [here](https://www.llvm.org/docs/LibFuzzer.html).

## Build the fuzz targets

In order to build the Core Lightning binaries with code coverage you will need a recent [clang](http://clang.llvm.org/). The more recent the compiler version the better.

Then you'll need to enable support at configuration time. You likely want to enable a few sanitizers for bug detections as well as experimental features for an extended coverage (not required though).

```shell
./configure --enable-developer --enable-address-sanitizer --enable-ub-sanitizer --enable-fuzzing --disable-valgrind CC=clang && make
```

The targets will be built in `tests/fuzz/` as `fuzz-` binaries, with their best known seed corpora stored in `tests/fuzz/corpora/`.

You can run the fuzz targets on their seed corpora to check for regressions:

```shell
make check-fuzz
```

## Run one or more target(s)

You can run each target independently. Pass `-help=1` to see available options, for example:

```
./tests/fuzz/fuzz-addr -help=1
```

Otherwise, you can use the Python runner to either run the targets against a given seed corpus:

```
./tests/fuzz/run.py fuzz_corpus -j2
```

Or extend this corpus:

```
./tests/fuzz/run.py fuzz_corpus -j2 --generate --runs 12345
```

The latter will run all targets two by two `12345` times.

If you want to contribute new seeds, be sure to merge your corpus with the main one:

```
./tests/fuzz/run.py my_locally_extended_fuzz_corpus -j2 --generate --runs 12345
./tests/fuzz/run.py tests/fuzz/corpora --merge_dir my_locally_extended_fuzz_corpus
```

## Improve seed corpora

If you find coverage increasing inputs while fuzzing, please create a pull request to add them into `tests/fuzz/corpora`. Be sure to minimize any additions to the corpora first.

### Example

Here's an example workflow to contribute new inputs for the `fuzz-addr` target.

Create a directory for newly found corpus inputs and begin fuzzing:

```shell
mkdir -p local_corpora/fuzz-addr
./tests/fuzz/fuzz-addr -jobs=4 local_corpora/fuzz-addr tests/fuzz/corpora/fuzz-addr/
```

After some time, libFuzzer may find some potential coverage increasing inputs and save them in `local_corpora/fuzz-addr`. We can then merge them into the seed corpora in `tests/fuzz/corpora`:

```shell
./tests/fuzz/run.py tests/fuzz/corpora --merge_dir local_corpora
```

This will copy over any inputs that improve the coverage of the existing corpus. If any new inputs were added, create a pull request to improve the upstream seed corpus:

```shell
git add tests/fuzz/corpora/fuzz-addr/*
git commit
...
```

## Write new fuzzing targets

In order to write a new target:

- include the `libfuzz.h` header
- fill two functions: `init()` for static stuff and `run()` which will be called repeatedly with mutated data.
- read about [what makes a good fuzz target](https://github.com/google/fuzzing/blob/master/docs/good-fuzz-target.md).

A simple example is [`fuzz-addr`](https://github.com/ElementsProject/lightning/blob/master/tests/fuzz/fuzz-addr.c). It setups the chainparams and context (wally, tmpctx, ..) in `init()` then bruteforces the bech32 encoder in `run()`.