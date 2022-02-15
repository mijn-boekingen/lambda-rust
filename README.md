# [AWS Lambda](https://aws.amazon.com/lambda/) [Rust](https://www.rust-lang.org/) ARM64 Docker Builder [![Build Status](https://github.com/mijn-boekingen/lambda-rust/workflows/Main/badge.svg)](https://github.com/mijn-boekingen/lambda-rust/actions)

## 🤔 About

This docker image extends AWS Lambda `provided.al2-arm64` runtime environment, and installs [rustup](https://rustup.rs/) and the *stable* rust toolchain.

This provides a build environment, consistent with your target execution environment for predictable results.

## 📦 Install

Tags for this docker image follow the naming convention `ghcr.io/mijn-boekingen/lambda-rust:{version}-rust-{rust-stable-version}`
where `{rust-stable-version}` is a stable version of rust.

You can find a list of available docker tags [here](https://github.com/mijn-boekingen/lambda-rust/pkgs/container/lambda-rust)

> 💡 If you don't find the version you're looking for, please [open a new github issue](https://github.com/mijn-boekingen/lambda-rust/issues/new?title=I%27m%20looking%20for%20version%20xxx) to publish one

You can also depend directly on `ghcr.io/mijn-boekingen/lambda-rust:latest-arm64` for the most recently published version.

## 🤸 Usage

The default docker entrypoint will build a packaged release optimized version of your Rust artifact under `target/lambda/release` to
isolate the lambda specific build artifacts from your host-local build artifacts.

> **⚠️ Note:** you can switch from the `release` profile to a custom profile like `dev` by providing a `PROFILE` environment variable set to the name of the desired profile. i.e. `-e PROFILE=dev` in your docker run
>
> **⚠️ Note:** you can include debug symbols in optimized release build binaries by setting `DEBUGINFO`. By default, debug symbols will be stripped from the release binary and set aside in a separate .debug file.

You will want to volume mount `/code` to the directory containing your cargo project.

You can pass additional flags to `cargo`, the Rust build tool, by setting the `CARGO_FLAGS` docker env variable.

Unzipped `boostrap` and `boostrap.debug` files are always available under `target/lambda/${PROFILE}/output/${BIN}` dir.
If you want only them and don't need a `.zip` archive (e.g. for when running lambdas locally) pass `-e PACKAGE=false` flag.
More on that in [local testing](#-local-testing).

A typical docker run might look like the following.

```sh
$ docker run --rm \
    -u "$(id -u)":"$(id -g)" \
    -v ${PWD}:/code \
    -v ${HOME}/.cargo/registry:/cargo/registry \
    -v ${HOME}/.cargo/git:/cargo/git \
    ghcr.io/mijn-boekingen/lambda-rust:latest-arm64
```

> 💡 The -v (volume mount) flags for `/cargo/{registry,git}` are optional but when supplied, provides a much faster turn around when doing iterative development

Note that `-u "$(id -u)":$(id -g)` argument is crucial for the container to produce artifacts
owned by the current host user, otherwise you won't be able to `rm -rf target/lambda`
or run `cargo update`, because the container will write artifacts owned by `root` docker user
to `target/lambda` and `./cargo/{registry,git}` dirs which will break your dev and/or ci environment.

You should also ensure that you do have `${HOME}/.cargo/{registry,git}` dirs created
on your host machine, otherwise docker will create them automatically and assign `root` user
as an owner for these dirs which is unfortunate...

If you are using Windows, the command above may need to be modified to include
a `BIN` environment variable set to the name of the binary to be build and packaged

```diff
$ docker run --rm \
    -u $(id -u):$(id -g) \
+   -e BIN={your-binary-name} \
    -v ${PWD}:/code \
    -v ${HOME}/.cargo/registry:/cargo/registry \
    -v ${HOME}/.cargo/git:/cargo/git \
    ghcr.io/mijn-boekingen/lambda-rust:latest-arm64
```

For more custom codebases, the '-w' argument can be used to override the working directory.
This can be especially useful when using path dependencies for local crates.

```diff
$ docker run --rm \
    -u $(id -u):$(id -g) \
    -v ${PWD}/lambdas/mylambda:/code/lambdas/mylambda \
    -v ${PWD}/libs/mylib:/code/libs/mylib \
    -v ${HOME}/.cargo/registry:/cargo/registry \
    -v ${HOME}/.cargo/git:/cargo/git \
+   -w /code/lambdas/mylambda \
    ghcr.io/mijn-boekingen/lambda-rust:latest-arm64
```

## ⚓ Using hooks

You can leverage hooks provided in the image to customize certain parts of the build process.
Hooks are shell scripts that are invoked if they exist, so you can customize the process. The following hooks exist:

* `install`: run before `cargo build` - useful for installing native dependencies on the lambda environment
* `build`: run after `cargo build`, but before packaging the executable into a zip - useful when modifying the executable after compilation
* `package`: run after packaging the executable into a zip - useful for adding extra files into the zip file

The hooks' names are predefined and must be placed in a directory `.lambda-rust` in the project root.

You can take a look at an example [here](./tests/test-func-with-hooks).

## 🔬 Local testing

Once you've built a Rust lambda function artifact, the `provided.al2-arm64` runtime expects deployments of that artifact to be named "**bootstrap**".
The `lambda-rust` docker image builds a zip file, named after the binary, containing your binary file renamed to "bootstrap" for you,
but zip file creation is unnecessary for local development.

In order to prevent the creation of an intermediate `.zip` artifact when testing your lambdas locally, pass `-e PACKAGE=false` during the build.
After that the necessary output (not zipped) is available under `target/lambda/{profile}/output/{your-lambda-binary-name}` dir.
You will see both `bootstrap` and `bootstrap.debug` files there.

> **⚠️ Note:** `PACKAGE=false` prevents `package` hook from running.

You can then invoke this bootstrap executable for the `provided.al2-arm64` AWS lambda runtime with a one off container.

```sh
# Build your function skipping the zip creation step
# You may pass `-e PROFILE=dev` to build using dev profile, but here we use `release`
docker run \
    -u $(id -u):$(id -g) \
    -e PACKAGE=false \
    -e BIN={your-binary-name} \
    -v ${PWD}:/code \
    -v ${HOME}/.cargo/registry:/cargo/registry \
    -v ${HOME}/.cargo/git:/cargo/git \
    ghcr.io/mijn-boekingen/lambda-rust:latest-arm64

# Build a container with your binary as the runtime

$ docker build -t mylambda -f- . <<EOF
FROM public.ecr.aws/lambda/provided:al2-arm64
COPY bootstrap /var/runtime
CMD [ "function.handler" ]
EOF

# start a container based on your image
$ docker run \
    --name lambda \
    --rm \
    -p 9000:8080 \
    -d mylambda

# provide an event payload (in event.json" by http POST to the container

$ curl -X POST \
    -H "Content-Type: application/json" \
    -d "@event.json" \
    "http://localhost:9000/2015-03-31/functions/function/invocations"

# Stop the container
$ docker container stop lambda
```

You may submit multiple events to the same container.

## 🤸🤸 Usage via cargo aws-lambda subcommand

A third party cargo subcommand exists to compile your code into a zip file and deploy it. This comes with only
rust and docker as dependencies.

Setup

```sh
$ cargo install cargo-aws-lambda
```

To compile and deploy in your project directory

```sh
$ cargo aws-lambda {your aws function's full ARN} {your-binary-name}
```

To list all options

```sh
$ cargo aws-lambda --help
```

More instructions can be found [here](https://github.com/vvilhonen/cargo-aws-lambda).

Doug Tangren ([softprops](https://github.com/softprops)) 2020, Alexander Zaitsev ([zamazan4ik](https://github.com/zamazan4ik)) 2021
