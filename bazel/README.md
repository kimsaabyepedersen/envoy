# Building Envoy with Bazel

## Production environments

To build Envoy with Bazel in a production environment, where the [Envoy
dependencies](https://www.envoyproxy.io/docs/envoy/latest/install/building.html#requirements) are typically
independently sourced, the following steps should be followed:

1. Install the latest version of [Bazel](https://bazel.build/versions/master/docs/install.html) in your environment.
2. Configure, build and/or install the [Envoy dependencies](https://www.envoyproxy.io/docs/envoy/latest/install/building.html#requirements).
3. Configure a Bazel [WORKSPACE](https://bazel.build/versions/master/docs/be/workspace.html)
   to point Bazel at the Envoy dependencies. An example is provided in the CI Docker image
   [WORKSPACE](https://github.com/envoyproxy/envoy/blob/master/ci/WORKSPACE) and corresponding
   [BUILD](https://github.com/envoyproxy/envoy/blob/master/ci/prebuilt/BUILD) files.
4. `bazel build --package_path %workspace%:<path to Envoy source tree> //source/exe:envoy-static`
   from the directory containing your WORKSPACE.

## Quick start Bazel build for developers

As a developer convenience, a [WORKSPACE](https://github.com/envoyproxy/envoy/blob/master/WORKSPACE) and
[rules for building a recent
version](https://github.com/envoyproxy/envoy/blob/master/bazel/repositories.bzl) of the various Envoy
dependencies are provided. These are provided as is, they are only suitable for development and
testing purposes. The specific versions of the Envoy dependencies used in this build may not be
up-to-date with the latest security patches. See
[this doc](https://github.com/envoyproxy/envoy/blob/master/bazel/EXTERNAL_DEPS.md#updating-an-external-dependency-version)
for how to update or override dependencies.

1. Install the latest version of [Bazel](https://bazel.build/versions/master/docs/install.html) in your environment.
2. Install external dependencies libtool, cmake, ninja, realpath and curl libraries separately.
On Ubuntu, run the following command:
```
sudo apt-get install \
   libtool \
   cmake \
   realpath \
   clang-format-7 \
   automake \
   ninja-build \
   curl \
   unzip
```

On Fedora (maybe also other red hat distros), run the following:
```
dnf install cmake libtool libstdc++ ninja-build lld patch
```

On macOS, you'll need to install several dependencies. This can be accomplished via [Homebrew](https://brew.sh/):
```
brew install coreutils wget cmake libtool go bazel automake ninja llvm@7
```
_notes_: `coreutils` is used for `realpath`, `gmd5sum` and `gsha256sum`; `llvm@7` is used for `clang-format`

Envoy compiles and passes tests with the version of clang installed by XCode 9.3.0:
Apple LLVM version 9.1.0 (clang-902.0.30).

In order for bazel to be aware of the tools installed by brew, the PATH
variable must be set for bazel builds. This can be accomplished by setting
this in your `$HOME/.bazelrc` file:

```
build --action_env=PATH="/usr/local/bin:/opt/local/bin:/usr/bin:/bin"
```

Alternatively, you can pass `--action_env` on the command line when running
`bazel build`/`bazel test`.

3. Install Golang on your machine. This is required as part of building [BoringSSL](https://boringssl.googlesource.com/boringssl/+/HEAD/BUILDING.md)
and also for [Buildifer](https://github.com/bazelbuild/buildtools) which is used for formatting bazel BUILD files.
4. `go get github.com/bazelbuild/buildtools/buildifier` to install buildifier
5. `bazel build //source/exe:envoy-static` from the Envoy source directory.

## Building Bazel with the CI Docker image

Bazel can also be built with the Docker image used for CI, by installing Docker and executing:

```
./ci/run_envoy_docker.sh './ci/do_ci.sh bazel.dev'
```

See also the [documentation](https://github.com/envoyproxy/envoy/tree/master/ci) for developer use of the
CI Docker image.

## Linking against libc++ on Linux

To link Envoy against libc++, use the following commands:
```
export CC=clang
export CXX=clang++
bazel build --config=libc++ //source/exe:envoy-static
```
Note: this assumes that both: clang compiler and libc++ library are installed in the system,
and that `clang` and `clang++` are available in `$PATH`. On some systems, exports might need
to be changed to versioned binaries, e.g. `CC=clang-7` and `CXX=clang++-7`.

You might also need to ensure libc++ is installed correctly on your system, e.g. on Ubuntu this
might look like `sudo apt-get install libc++abi-7-dev libc++-7-dev`.

## Using a compiler toolchain in a non-standard location

By setting the `CC` and `LD_LIBRARY_PATH` in the environment that Bazel executes from as
appropriate, an arbitrary compiler toolchain and standard library location can be specified. One
slight caveat is that (at the time of writing), Bazel expects the binutils in `$(dirname $CC)` to be
unprefixed, e.g. `as` instead of `x86_64-linux-gnu-as`.

## Supported compiler versions

Though Envoy has been run in production compiled with GCC 4.9 extensively, we now require
GCC >= 5 due to known issues with std::string thread safety and C++14 support. Clang >= 4.0 is also
known to work.

## Clang STL debug symbols

By default Clang drops some debug symbols that are required for pretty printing to work correctly.
More information can be found [here](https://bugs.llvm.org/show_bug.cgi?id=24202). The easy solution
is to set ```--copt=-fno-limit-debug-info``` on the CLI or in your .bazelrc file.

# Testing Envoy with Bazel

All the Envoy tests can be built and run with:

```
bazel test //test/...
```

An individual test target can be run with a more specific Bazel
[label](https://bazel.build/versions/master/docs/build-ref.html#Labels), e.g. to build and run only
the units tests in
[test/common/http/async_client_impl_test.cc](https://github.com/envoyproxy/envoy/blob/master/test/common/http/async_client_impl_test.cc):

```
bazel test //test/common/http:async_client_impl_test
```

To observe more verbose test output:

```
bazel test --test_output=streamed //test/common/http:async_client_impl_test
```

It's also possible to pass into an Envoy test additional command-line args via `--test_arg`. For
example, for extremely verbose test debugging:

```
bazel test --test_output=streamed //test/common/http:async_client_impl_test --test_arg="-l trace"
```

By default, testing exercises both IPv4 and IPv6 address connections. In IPv4 or IPv6 only
environments, set the environment variable ENVOY_IP_TEST_VERSIONS to "v4only" or
"v6only", respectively.

```
bazel test //test/... --test_env=ENVOY_IP_TEST_VERSIONS=v4only
bazel test //test/... --test_env=ENVOY_IP_TEST_VERSIONS=v6only
```

By default, tests are run with the [gperftools](https://github.com/gperftools/gperftools) heap
checker enabled in "normal" mode to detect leaks. For other mode options, see the gperftools
heap checker [documentation](https://gperftools.github.io/gperftools/heap_checker.html). To
disable the heap checker or change the mode, set the HEAPCHECK environment variable:

```
# Disables the heap checker
bazel test //test/... --test_env=HEAPCHECK=
# Changes the heap checker to "minimal" mode
bazel test //test/... --test_env=HEAPCHECK=minimal
```

If you see a leak detected, by default the reported offsets will require `addr2line` interpretation.
You can run under `--config=clang-asan` to have this automatically applied.

Bazel will by default cache successful test results. To force it to rerun tests:

```
bazel test //test/common/http:async_client_impl_test --cache_test_results=no
```

Bazel will by default run all tests inside a sandbox, which disallows access to the
local filesystem. If you need to break out of the sandbox (for example to run under a
local script or tool with [`--run_under`](https://docs.bazel.build/versions/master/user-manual.html#flag--run_under)),
you can run the test with `--strategy=TestRunner=standalone`, e.g.:

```
bazel test //test/common/http:async_client_impl_test --strategy=TestRunner=standalone --run_under=/some/path/foobar.sh
```
# Stack trace symbol resolution

Envoy can produce backtraces on demand and from assertions and other fatal
actions like segfaults. Where supported, stack traces will contain resolved
symbols, though not include line numbers. On systems where absl::Symbolization is
not supported, the stack traces written in the log or to stderr contain addresses rather
than resolved symbols. If the symbols were resolved, the address is also included at
the end of the line.

The `tools/stack_decode.py` script exists to process the output and do additional symbol
resolution including file names and line numbers. It requires the `addr2line` program be
installed and in your path. Any log lines not relevant to the backtrace capability are
passed through the script unchanged (it acts like a filter). File and line information
is appended to the stack trace lines.

The script runs in one of two modes. To process log input from stdin, pass `-s` as the first
argument, followed by the executable file path. You can postprocess a log or pipe the output
of an Envoy process. If you do not specify the `-s` argument it runs the arguments as a child
process. This enables you to run a test with backtrace post processing. Bazel sandboxing must
be disabled by specifying standalone execution. Example command line with
`run_under`:

```
bazel test -c dbg //test/server:backtrace_test
--run_under=`pwd`/tools/stack_decode.py --strategy=TestRunner=standalone
--cache_test_results=no --test_output=all
```

Example using input on stdin:

```
bazel test -c dbg //test/server:backtrace_test --cache_test_results=no --test_output=streamed |& tools/stack_decode.py -s bazel-bin/test/server/backtrace_test
```

You will need to use either a `dbg` build type or the `opt` build type to get file and line
symbol information in the binaries.

By default main.cc will install signal handlers to print backtraces at the
location where a fatal signal occurred. The signal handler will re-raise the
fatal signal with the default handler so a core file will still be dumped after
the stack trace is logged. To inhibit this behavior use
`--define=signal_trace=disabled` on the Bazel command line. No signal handlers will
be installed.

# Running a single Bazel test under GDB

```
tools/bazel-test-gdb //test/common/http:async_client_impl_test -c dbg
```

Without the `-c dbg` Bazel option at the end of the command line the test
binaries will not include debugging symbols and GDB will not be very useful.

# Running Bazel tests requiring privileges

Some tests may require privileges (e.g. CAP_NET_ADMIN) in order to execute. One option is to run
them with elevated privileges, e.g. `sudo test`. However, that may not always be possible,
particularly if the test needs to run in a CI pipeline. `tools/bazel-test-docker.sh` may be used in
such situations to run the tests in a privileged docker container.

The script works by wrapping the test execution in the current repository's circle ci build
container, then executing it either locally or on a remote docker container. In both cases, the
container runs with the `--privileged` flag, allowing it to execute operations which would otherwise
be restricted.

The command line format is:
`tools/bazel-test-docker.sh <bazel-test-target> [optional-flags-to-bazel]`

The script uses two optional environment variables to control its behaviour:

* `RUN_REMOTE=<yes|no>`: chooses whether to run on a remote docker server.
* `LOCAL_MOUNT=<yes|no>`: copy/mount local libraries onto the docker container.

Use `RUN_REMOTE=yes` when you don't want to run against your local docker instance. Note that you
will need to override a few environment variables to set up the remote docker. The list of variables
can be found in the [Documentation](https://docs.docker.com/engine/reference/commandline/cli/).

Use `LOCAL_MOUNT=yes` when you are not building with the envoy build container. This will ensure
that the libraries against which the tests dynmically link will be available and of the correct
version.

## Examples

Running the http integration test in a privileged container:

```bash
tools/bazel-test-docker.sh  //test/integration:integration_test --jobs=4 -c dbg
```

Running the http integration test compiled locally against a privileged remote container:

```bash
setup_remote_docker_variables
RUN_REMOTE=yes MOUNT_LOCAL=yes tools/bazel-test-docker.sh  //test/integration:integration_test \
  --jobs=4 -c dbg
```

# Additional Envoy build and test options

In general, there are 3 [compilation
modes](https://docs.bazel.build/versions/master/user-manual.html#flag--compilation_mode)
that Bazel supports:

* `fastbuild`: `-O0`, aimed at developer speed (default).
* `opt`: `-O2 -DNDEBUG -ggdb3`, for production builds and performance benchmarking.
* `dbg`: `-O0 -ggdb3`, no optimization and debug symbols.

You can use the `-c <compilation_mode>` flag to control this, e.g.

```
bazel build -c opt //source/exe:envoy-static
```

## Sanitizers

To build and run tests with the gcc compiler's [address sanitizer
(ASAN)](https://github.com/google/sanitizers/wiki/AddressSanitizer) and
[undefined behavior
(UBSAN)](https://developers.redhat.com/blog/2014/10/16/gcc-undefined-behavior-sanitizer-ubsan) sanitizer enabled:

```
bazel test -c dbg --config=asan //test/...
```

The ASAN failure stack traces include line numbers as a result of running ASAN with a `dbg` build above. If the
stack trace is not symbolized, try setting the ASAN_SYMBOLIZER_PATH environment variable to point to the
llvm-symbolizer binary (or make sure the llvm-symbolizer is in your $PATH).

If you have clang-5.0 or newer, additional checks are provided with:

```
bazel test -c dbg --config=clang-asan //test/...
```

Similarly, for [thread sanitizer (TSAN)](https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual) testing:

```
bazel test -c dbg --config=clang-tsan //test/...
```

## Log Verbosity

Log verbosity is controlled at runtime in all builds.

## Disabling optional features

The following optional features can be disabled on the Bazel build command-line:

* Hot restart with `--define hot_restart=disabled`
* Google C++ gRPC client with `--define google_grpc=disabled`
* Backtracing on signals with `--define signal_trace=disabled`
* tcmalloc with `--define tcmalloc=disabled`

## Enabling optional features

The following optional features can be enabled on the Bazel build command-line:

* Exported symbols during linking with `--define exported_symbols=enabled`.
  This is useful in cases where you have a lua script that loads shared object libraries, such as
  those installed via luarocks.
* Perf annotation with `--define perf_annotation=enabled` (see
  source/common/common/perf_annotation.h for details).
* BoringSSL can be built in a FIPS-compliant mode with `--define boringssl=fips`
  (see [FIPS 140-2](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/ssl.html#fips-140-2) for details).
* ASSERT() can be configured to log failures and increment a stat counter in a release build with
  `--define log_debug_assert_in_release=enabled`. The default behavior is to compile debug assertions out of
  release builds so that the condition is not evaluated. This option has no effect in debug builds.
* memory-debugging (scribbling over memory after allocation and before freeing) with
  `--define tcmalloc=debug`.

## Disabling extensions

Envoy uses a modular build which allows extensions to be removed if they are not needed or desired.
Extensions that can be removed are contained in
[extensions_build_config.bzl](../source/extensions/extensions_build_config.bzl). Use the following
procedure to customize the extensions for your build:

* The Envoy build assumes that a Bazel repository named `@envoy_build_config` exists which
  contains the file `@envoy_build_config//:extensions_build_config.bzl`. In the default build,
  a synthetic repository is created containing [extensions_build_config.bzl](../source/extensions/extensions_build_config.bzl).
  Thus, the default build has all extensions.
* Start by creating a new Bazel workspace somewhere in the filesystem that your build can access.
  This workspace should contain:
  * Empty WORKSPACE file.
  * Empty BUILD file.
  * A copy of [extensions_build_config.bzl](../source/extensions/extensions_build_config.bzl).
  * Comment out any extensions that you don't want to build in your file copy.

To have your local build use your overridden configuration repository there are two options:

1. Use the [`--override_repository`](https://docs.bazel.build/versions/master/command-line-reference.html)
   CLI option to override the `@envoy_build_config` repo.
2. Use the following snippet in your WORKSPACE before you load the Envoy repository. E.g.,

```
workspace(name = "envoy")

local_repository(
    name = "envoy_build_config",
    # Relative paths are also supported.
    path = "/somewhere/on/filesystem/envoy_build_config",
)

local_repository(
    name = "envoy",
    # Relative paths are also supported.
    path = "/somewhere/on/filesystem/envoy",
)

...
```

## Stats Tunables

The default maximum number of stats in shared memory, and the default
maximum length of a cluster/route config/listener name, can be
overridden at compile-time by defining `ENVOY_DEFAULT_MAX_STATS` and
`ENVOY_DEFAULT_MAX_OBJ_NAME_LENGTH`, respectively, to the desired
value. For example:

```
bazel build --copt=-DENVOY_DEFAULT_MAX_STATS=32768 --copt=-DENVOY_DEFAULT_MAX_OBJ_NAME_LENGTH=150 //source/exe:envoy-static
```

# Release builds

Release builds should be built in `opt` mode, processed with `strip` and have a
`.note.gnu.build-id` section with the Git SHA1 at which the build took place.
They should also ignore any local `.bazelrc` for reproducibility. This can be
achieved with:

```
bazel --bazelrc=/dev/null build -c opt //source/exe:envoy-static.stripped
```

One caveat to note is that the Git SHA1 is truncated to 16 bytes today as a
result of the workaround in place for
https://github.com/bazelbuild/bazel/issues/2805.

# Coverage builds

To generate coverage results, make sure you have
[`gcovr`](https://github.com/gcovr/gcovr) 3.3 in your `PATH` (or set `GCOVR` to
point at it) and are using a GCC toolchain (clang does not work currently, see
https://github.com/envoyproxy/envoy/issues/1000). Then run:

```
test/run_envoy_bazel_coverage.sh
```

The summary results are printed to the standard output and the full coverage
report is available in `generated/coverage/coverage.html`.

Coverage for every PR is available in Circle in the "artifacts" tab of the coverage job. You will
need to navigate down and open "coverage.html" but then you can navigate per normal. NOTE: We
have seen some issues with seeing the artifacts tab. If you can't see it, log out of Circle, and
then log back in and it should start working.

The latest coverage report for master is available
[here](https://s3.amazonaws.com/lyft-envoy/coverage/report-master/coverage.html).

It's also possible to specialize the coverage build to a single test target. This is useful
when doing things like exploring the coverage of a fuzzer over its corpus. This can be done with
the `COVERAGE_TARGET` and `VALIDATE_COVERAGE` environment variables, e.g.:

```
COVERAGE_TARGET=//test/common/common:base64_fuzz_test VALIDATE_COVERAGE=false test/run_envoy_bazel_coverage.sh
```

# Cleaning the build and test artifacts

`bazel clean` will nuke all the build/test artifacts from the Bazel cache for
Envoy proper. To remove the artifacts for the external dependencies run
`bazel clean --expunge`.

If something goes really wrong and none of the above work to resolve a stale build issue, you can
always remove your Bazel cache completely. It is likely located in `~/.cache/bazel`.

# Adding or maintaining Envoy build rules

See the [developer guide for writing Envoy Bazel rules](DEVELOPER.md).

# Bazel performance on (virtual) machines with low resources

If the (virtual) machine that is performing the build is low on memory or CPU
resources, you can override Bazel's default job parallelism determination with
`--jobs=N` to restrict the build to at most `N` simultaneous jobs, e.g.:

```
bazel build --jobs=2 //source/...
```

# Debugging the Bazel build

When trying to understand what Bazel is doing, the `-s` and `--explain` options
are useful. To have Bazel provide verbose output on which commands it is executing:

```
bazel build -s //source/...
```

To have Bazel emit to a text file the rationale for rebuilding a target:

```
bazel build --explain=file.txt //source/...
```

To get more verbose explanations:

```
bazel build --explain=file.txt --verbose_explanations //source/...
```

# Resolving paths in bazel build output

Sometimes it's useful to see real system paths in bazel error message output (vs. symbolic links).
`tools/path_fix.sh` is provided to help with this. See the comments in that file.

# Compilation database

Run `tools/gen_compilation_database.py` to generate
a [JSON Compilation Database](https://clang.llvm.org/docs/JSONCompilationDatabase.html). This could be used
with any tools (e.g. clang-tidy) compatible with the format.

The compilation database could also be used to setup editors with cross reference, code completion.
For example, you can use [You Complete Me](https://valloric.github.io/YouCompleteMe/) or
[cquery](https://github.com/cquery-project/cquery) with supported editors.

# Running clang-format without docker

The easiest way to run the clang-format check/fix commands is to run them via
docker, which helps ensure the right toolchain is set up. However you may prefer
to run clang-format scripts on your workstation directly:
 * It's possible there is a speed advantage
 * Docker itself can sometimes go awry and you then have to deal with that
 * Type-ahead doesn't always work when waiting running a command through docker
To run the tools directly, you must install the correct version of clang. This
may change over time but as of January 2019,
[clang+llvm-7.0.0](http://releases.llvm.org/download.html) works well. You must
also have 'buildifier' installed from the bazel distribution.

Edit the paths shown here to reflect the installation locations on your system:

```shell
export CLANG_FORMAT="$HOME/ext/clang+llvm-7.0.0-x86_64-linux-gnu-ubuntu-16.04/bin/clang-format"
export BUILDIFIER_BIN="/usr/bin/buildifier"
```

Once this is set up, you can run clang-tidy without docker:

```shell
./tools/check_format.py check
./tools/check_spelling.sh check
./tools/check_format.py fix
./tools/check_spelling.sh fix
```

# Advanced caching setup

Setting up an HTTP cache for Bazel output helps optimize Bazel performance and resource usage when
using multiple compilation modes or multiple trees.

## Setup common `envoy_deps`

This step sets up the common `envoy_deps` allowing HTTP or disk cache (described below) to work
across working trees in different paths. Also it allows new working trees to skip dependency
compilation. The drawback is that the cached dependencies won't be updated automatically, so make
sure all your working trees have same (or compatible) dependencies, and run this step periodically
to update them.

Make sure you don't have `--override_repository` in your `.bazelrc` when you run this step.

```
bazel fetch //test/...
cp -LR $(bazel info output_base)/external/envoy_deps ${HOME}/envoy_deps_cache
```

Adding the following parameter to Bazel everytime or persist them in `.bazelrc`, note you will need to expand
the environment variables for `.bazelrc`.

```
--override_repository=envoy_deps=${HOME}/envoy_deps_cache
```

## Setup local cache

You may use any [Remote Caching](https://docs.bazel.build/versions/master/remote-caching.html) backend
as an alternative to this.

This requires Go 1.11+, follow the [instructions](https://golang.org/doc/install#install) to install
if you don't have one. To start the cache, run the following from the root of the Envoy repository (or anywhere else
that the Go toolchain can find the necessary dependencies):

```
go run github.com/buchgr/bazel-remote --dir ${HOME}/bazel_cache --host 127.0.0.1 --port 28080 --max_size 64
```

See [Bazel remote cache](github.com/buchgr/bazel-remote) for more information on the parameters.
The command above will setup a maximum 64 GiB cache at `~/bazel_cache` on port 28080. You might
want to setup a larger cache if you run ASAN builds.

NOTE: Using docker to run remote cache server described in remote cache docs will likely have
slower cache performance on macOS due to slow disk performance on Docker for Mac.

Adding the following parameter to Bazel everytime or persist them in `.bazelrc`.

```
--remote_http_cache=http://127.0.0.1:28080/
```
