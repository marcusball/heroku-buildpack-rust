# Heroku buildpack for Rust

This is a Heroku buildpack for Rust with support for [cargo][] and [rustup][].  Features include:

- Caching of builds between deployments.
- Automatic updates to the latest stable Rust by default.
- Optional pinning of Rust to a specific version.
- Support for `export` so that other buildpacks can access the Rust toolchain.
- Support for compiling Rust-based extensions for projects written in other languages.

[fode]: https://github.com/ericfode/heroku-buildpack-rust
[cargo]: http://crates.io/
[rustup]: https://www.rustup.rs/

## Intro

This is a fork of [emk's heroku-buildpack-rust](https://github.com/emk/heroku-buildpack-rust) buildpack,
with slight influence from [sgrif's heroku-buildpack-diesel](https://github.com/sgrif/heroku-buildpack-diesel) buildpack. 

This fork was made rather hastily, so for further info check their versions, but I'll try to be responsive and document the changes made. 

## Using this buildpack

To deploy an application to Heroku, we recommend installing the [Heroku CLI][].

If you're creating a new Heroku application, `cd` to the directory containing your code, and run:

```sh
heroku create --buildpack https://github.com/marcusball/heroku-buildpack-rust
```

This will only work if your application has a `Cargo.toml` and uses `git`. If you want to set a particular name for application, see `heroku create --help` first.

To use this as the buildpack for an existing application, run:

```sh
heroku buildpacks:set https://github.com/marcusball/heroku-buildpack-rust
```

You will also need to create a `Procfile` pointing to the release version of your application, and commit it to `git`:

```Procfile
web: ./target/release/hello
```

...where `hello` is the name of your binary.

To deploy your application, run:

```sh
git push heroku master
```

### Running Diesel migrations during the release phase

This will install the diesel CLI at build time and make it available in your dyno. Migrations will run whenever a new version of your app is released. Add the following line to your `RustConfig`

```sh
RUST_INSTALL_DIESEL=1
```

and this one to your `Procfile`

```Procfile
release: ./target/release/diesel migration run
```

[Heroku CLI]: https://devcenter.heroku.com/articles/heroku-command-line

## Specifying which version of Rust to use

By default, your application will be built using the latest stable Rust. Normally, this is pretty safe: New stable Rust releases have excellent backwards compatibility.

But you may wish to use `nightly` Rust or to lock your Rust version to a known-good configuration for more reproducible builds. To specify a specific version of the toolchain, use a [`rust-toolchain`](https://github.com/rust-lang-nursery/rustup.rs#the-toolchain-file) file in the format rustup uses.

Note: if you previously specified a `VERSION` variable in `RustConfig`, that will continue to work, and will override a `rust-toolchain` file.

## Combining with other buildpacks

If you have a project which combines both Rust and another programming language, you can insert this buildpack before your existing one as follows:

```sh
heroku buildpacks:add --index 1 https://github.com/marcusball/heroku-buildpack-rust
```

If you have a valid `Cargo.toml` in your project, this is all you need to do. The Rust buildpack will run first, and your existing buildpack will run second.

But if you only need Rust to build a particular Ruby gem, and you have no top-level `Cargo.toml` file, you'll need to let the buildpack know to skip the build stage.  You can do this by adding the following line to `RustConfig`:

```sh
RUST_SKIP_BUILD=1
```

## Customizing build flags

If you want to change the cargo build command, you can set the `RUST_CARGO_BUILD_FLAGS` variable inside the `RustConfig` file.

```sh
RUST_CARGO_BUILD_FLAGS="--release -p some_package --bin some_exe --bin some_bin_2"
```

The default value of `RUST_CARGO_BUILD_FLAGS` is `--release`.
If the variable is not set in `RustConfig`, the default value will be used to build the project.

## `RustConfig` Options

As noted above, there are several options you can specify in a `RustConfig` file to tweak this buildpack. 

### `RUST_SKIP_BUILD`

Set this to "1" to just install a Rust toolchain and not build `Cargo.toml`.  This is useful if you have a project written in Ruby or Node (for example) that needs to build extension modules using Rust.

Default value: `0`
### `BUILD_PATH`

If your Rust code is not at the root directory of the repository, specify a `BUILD_PATH` to the correct directory.

Default value `""` (the project root)

### RUST_INSTALL_DIESEL

Set this to "1" to install diesel at build time and copy it
into the target directory, next to your app binary. This makes it easy to
# run migrations by adding a release step to your Procfile:
# `release: ./target/release/diesel migration run`

# These flags are passed to `cargo install diesel`, e.g. '--no-default-features --features postgres'
DIESEL_FLAGS=""
# Directory into which the `diesel` binary will be copied after it is built. 
DIESEL_INSTALL_DIR="target/release/"
# Default build flags to pass to `cargo build`.
RUST_CARGO_BUILD_FLAGS="--release"

## Development notes

If you need to tweak this buildpack, the following information may help.

### Testing with Docker

To test changes to the buildpack using the included `docker-compose-test.yml`, run:

```sh
./test_buildpack
```

Then make sure there are no Rust-related *.so files getting linked:

```sh
ldd heroku-rust-cargo-hello/target/release/hello
```

This uses the Docker image `heroku/cedar`, which allows us to test in an official Cedar-like environment.

We also run this test automatically on Travis CI.
