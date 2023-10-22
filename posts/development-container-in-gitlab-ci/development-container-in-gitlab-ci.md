---
title: Development container & Gitlab CI - Build better and faster
description: Improve build quality without compromising speed with a development container in Gitlab CI
cover_image: ./docker_gitlab_run.png
tags: 'docker, gitlab, container, pipeline'
canonical_url: null
published: false
---

One of the benefits of an automated build pipeline is reproducible outcomes. This alone has several other benefits such as more reasonable results, and less surprises. Ideally we want the exact same stuff every time we run the pipeline for a specific version of our code regardless of where we are executing it.

It is encouraged to version control everything that affects the final outcome. Like the code, third party dependencies, all the OS packages required to build our software, the pipeline itself, etc. Additionally, each run has to be isolated to avoid interference of all the side effects caused by the other runs or pipelines.

This approach clearly increases our confidence in the result. But, one of it's biggest caveats is much slower build times. Definitely, it is exhaustive to prepare the environment and commence the build from scratch each time. So, the challenge is to have the predictable outcomes while having it fast.

In this post we want to exercise this challenge using [Gitlab CI](https://docs.gitlab.com/ee/ci/) and [Docker](https://www.docker.com/).

## The build environment

The environment that we run our build is the base layer of all the other operations. The compiler, packaging tools, operating system and all that is installed on it are all examples of what makes that environment. Tools like Docker make it a lot easy to prepare our desired environment as a [container](https://www.docker.com/resources/what-container/). We can put together all the bits and pieces with a [Dockerfile](https://docs.docker.com/engine/reference/builder/), or pick a precooked one (like Node.js 20 on Debian 10) off [Docker Hub](https://hub.docker.com/).

We want to defined our pipeline with Gitlab CI in a way that this container is efficiently created with each check-in and used through the rest of the steps.

## Context

To give it a bit of a context, we will write a program in [Rust](https://www.rust-lang.org/) which calls a [gRPC](https://grpc.io) API and in the CI pipeline we will build and test it. We are going to start with an empty project and develop it step by step.

You can see the taken steps and resulting pipelines all in [**Development Container repository**](https://gitlab.com/yaghmaie/development-container). Each commit represents a single step and it's changes. Also take a look at the [**Pipelines**](https://gitlab.com/yaghmaie/development-container/-/pipelines) ran during this exercise.

### Step 1: [Hello, World](https://gitlab.com/yaghmaie/development-container/-/commit/e9e9d066b5eaba3815ad65f83c68ea667aff6c05)

First, we create a new project with `cargo new grpccaller` and we add a small hello world test in [`main.rs`](https://gitlab.com/yaghmaie/development-container/-/blob/e9e9d066b5eaba3815ad65f83c68ea667aff6c05/src/main.rs). We add a single stage to run `cargo test` in our [pipeline](https://gitlab.com/yaghmaie/development-container/-/blob/e9e9d066b5eaba3815ad65f83c68ea667aff6c05/.gitlab-ci.yml).

At this point we just want to make sure that our CI server is able to pick up jobs, and we are able to run some unit tests in the pipeline to get feedback on our changes to the code. Keeping the progress in tiny steps helps to catch issues earlier when they are not too complex.

```yml
# .gitlab-ci.yml

stages:
  - test

cargo:test:
  stage: test
  image: rust:alpine3.18
  script:
    - cargo test
```

There is no need to have the **Development Container** yet. We can use the existing [official Rust docker image](https://hub.docker.com/_/rust) off the shelf.

### Step 2: [Add gRPC dependencies](https://gitlab.com/yaghmaie/development-container/-/commit/f972575a11ca76ff1faa52f5ffccf9a1fa491aee)

Now we add the dependencies we need for implementing the gRPC client to [`Cargo.toml`](https://gitlab.com/yaghmaie/development-container/-/blob/f972575a11ca76ff1faa52f5ffccf9a1fa491aee/Cargo.toml) then push it so the CI server run tests for us and make sure all is good. But in software development nothing always goes as planned.

```toml
# Cargo.toml

[dependencies]
prost = "0.12.1"
tokio = { version = "1.33.0", features = ["macros", "rt-multi-thread"] }
tonic = "0.10.2"

[build-dependencies]
tonic-build = "0.10.2"
```

Looks like `cargo` cannot finish the build because of some packages missing from the [official Rust docker image](https://hub.docker.com/_/rust).

```plain
cannot find crti.o: No such file or directory
```

### Step 3: [Install required packages for build](https://gitlab.com/yaghmaie/development-container/-/commit/e87efc7d65e43ad553c2e964a0abedd6f66d1102)

So, as the next step we will easily add some lines to the [CI file](https://gitlab.com/yaghmaie/development-container/-/blob/e87efc7d65e43ad553c2e964a0abedd6f66d1102/.gitlab-ci.yml) and fix it.

```yml
# .gitlab-ci.yml

stages:
  - test

cargo:test:
  stage: test
  image: rust:alpine3.18
  script:
    - apk add musl-dev
    - apk add protoc
    - cargo test
```

You may have noticed what we are doing is not the best. Every time the pipeline runs it installs `musl-dev` and `protoc`, fetches all the dependencies, and compiles the whole thing from start. It can be a lot more efficient but we rather make it work first then worry about making it right and fast.

### Step 4: [Call grpcbin SayHello](https://gitlab.com/yaghmaie/development-container/-/commit/d06e1940e8f86688cd7a8400dc711ff2ca9f5478)

We want to call the [grpcbin](https://github.com/moul/grpcbin) hello service to say hello. We will use the instance hosted by [k6](https://k6.io/) at <https://grpcbin.test.k6.io/>.

We add [`helloworld.proto`](https://gitlab.com/yaghmaie/development-container/-/blob/d06e1940e8f86688cd7a8400dc711ff2ca9f5478/proto/helloworld.proto), a [`build.rs`](https://gitlab.com/yaghmaie/development-container/-/blob/d06e1940e8f86688cd7a8400dc711ff2ca9f5478/build.rs), and apply changes to [`main.rs`](https://gitlab.com/yaghmaie/development-container/-/blob/d06e1940e8f86688cd7a8400dc711ff2ca9f5478/src/main.rs) to make it work.

```rust
// main.rs

fn hello_world() -> &'static str {
    "Hello, world"
pub mod hello {
    tonic::include_proto!("hello");
}

fn main() {
    println!("{}", hello_world());
use hello::hello_service_client::HelloServiceClient;
use hello::HelloRequest;

async fn hello_world() -> Result<String, Box<dyn std::error::Error>> {
    let mut client = HelloServiceClient::connect("http://grpcbin.test.k6.io:9000").await?;

    let request = tonic::Request::new(HelloRequest {
        greeting: Some("world".into()),
    });

    let response = client.say_hello(request).await?;

    Ok(response.into_inner().reply)
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("{}", hello_world().await?);

    Ok(())
}

#[cfg(test)]
mod tests {
    use crate::hello_world;

    #[test]
    fn test_hello_world() {
        assert_eq!(hello_world(), "Hello, world");
    #[tokio::test]
    async fn test_hello_world() {
        assert_eq!(hello_world().await.unwrap(), "hello world");
    }
}

```

### Step 5: [Cache installed dependencies & compilations](https://gitlab.com/yaghmaie/development-container/-/commit/04768d1ac3b718be6d0a215bf160b16864d98f1e)

With this setup, `cargo` has to download all the 3-rd party dependencies and compile them each time we run our pipeline. This is because the runs are isolated and cannot interfere with each other. But unlike code, the dependencies not often change. Therefore, it would be a great win if we could reuse that effort.

We can easily achieve it by utilizing [Gitlab CI's cache](https://docs.gitlab.com/ee/ci/caching/). [**The Cargo Book**](https://doc.rust-lang.org/cargo/index.html) lists the [files and directories](https://doc.rust-lang.org/cargo/guide/cargo-home.html#caching-the-cargo-home-in-ci) we can cache to get the most efficient experience while installing dependencies. There is also a [build cache](https://doc.rust-lang.org/cargo/guide/build-cache.html) (`/target` directory) which is output of a build and we can safely store it in our CI cache to speed up our compilation time.

There is a limitation with caching in Gitlab CI which is the paths we define to be cached must be within the project directory. It means we cannot save the files in [**Cargo Home**](https://doc.rust-lang.org/cargo/guide/cargo-home.html) unless we change it's path to somewhere within our project's directory. Thus, we need to set the `CARGO_HOME` environment variable to something like `${CI_PROJECT_DIR}/.cargo`. If you are wondering what `${CI_PROJECT_DIR}` is, then you need to take a look at [Gitlab CI's predefined variables](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html).

Finally we end up here:

```yml
# .gitlab-ci.yml

stages:
  - test

variables:
  CARGO_HOME: ${CI_PROJECT_DIR}/.cargo

cargo:test:
  stage: test
  image: rust:alpine3.18
  cache:
    policy: pull-push
    paths:
      - .cargo/.crates.toml
      - .cargo/.crates2.json
      - .cargo/bin/
      - .cargo/registry/index/
      - .cargo/registry/cache/
      - .cargo/git/db/
      - .cargo/bin
      - target/
    when: on_success
  script:
    - apk add musl-dev
    - apk add protoc
    - cargo test

```

On the first run after checking in these changes it got [longer than 2 minutes](https://gitlab.com/yaghmaie/development-container/-/pipelines/1044765415) for the pipeline to finish. But, the second run took [52 seconds](https://gitlab.com/yaghmaie/development-container/-/pipelines/1044768619) which indicates a build with cache is at least twice faster in this case.

### Step 6: [Make development container](https://gitlab.com/yaghmaie/development-container/-/commit/7a4f81f0d0de2401ed18626da30e24b11d3cdd32)

At this point we have a working feature and we improved caching in our pipeline. But right now, whenever we want to run `cargo` we have to install OS packages (`musl-dev` & `protoc`) first. This is far from ideal since there should be no need to install them each time a job in the pipeline uses `cargo`. It is not possible to use cache in this case because the installed files are outside project directory and there is no good way to bring them in.

A **Development Container** can solve this problem for us. Instead of doing our operations in the official rust container, we can make a new image based on it that suits our specific needs.

First thing we need is a *Dockerfile*. So, we move those two lines out of the test stage and bring them into:

```dockerfile
# Development.Dockerfile

FROM rust:alpine3.18

RUN apk add --no-cache musl-dev=1.2.4-r2
RUN apk add --no-cache protoc=3.21.12-r2
```

More specific package versions are encouraged for more predictable outcomes.

We will build an image from this file and push it to our project's [Container Registry](https://docs.gitlab.com/ee/user/packages/container_registry/) at the first stage (*prepare*) of the pipeline and will use it in the rest of the stages when needed.

You might ask if building a container on every run is wasteful or not? Of course it could be. But, using `docker build` we can [specify external cache sources](https://docs.docker.com/engine/reference/commandline/build/#cache-from) with `--cache-from` that makes the process really efficient. Containers' layered architecture makes not just building but also storing them very efficient. You can learn more about how it works [here](https://docs.docker.com/build/cache/).

We also need to decide on naming and tagging our development containers. Fortunately, Gitlab CI has a variable for the registry image (`$CI_REGISTRY_IMAGE`) we can use and define new variables:

```yml
# .gitlab-ci.yml

---
variables:
  DEVELOPMENT_IMAGE: $CI_REGISTRY_IMAGE/development
  DEVELOPMENT_IMAGE_MAIN_BRANCH_TAG: $DEVELOPMENT_IMAGE:main
  DEVELOPMENT_IMAGE_CURRENT_BRANCH_TAG: $DEVELOPMENT_IMAGE:$CI_COMMIT_BRANCH
  DEVELOPMENT_IMAGE_TAG: $DEVELOPMENT_IMAGE:$CI_COMMIT_SHORT_SHA
---
```

For you to have a better understanding, in my case these can translate into the following:

* DEVELOPMENT_IMAGE: registry.gitlab.com/yaghmaie/development-container/development
* DEVELOPMENT_IMAGE_MAIN_BRANCH_TAG: registry.gitlab.com/yaghmaie/development-container/development:main
* DEVELOPMENT_IMAGE_CURRENT_BRANCH_TAG: registry.gitlab.com/yaghmaie/development-container/development:some-other-branch
* DEVELOPMENT_IMAGE_TAG: registry.gitlab.com/yaghmaie/development-container/development:7a4f81f0

We want to use `DEVELOPMENT_IMAGE_MAIN_BRANCH_TAG` and `DEVELOPMENT_IMAGE_CURRENT_BRANCH_TAG` as cache sources to build `DEVELOPMENT_IMAGE_TAG` and update `DEVELOPMENT_IMAGE_CURRENT_BRANCH_TAG`. We will use `DEVELOPMENT_IMAGE_TAG` in the rest of the stages of our pipeline.

This is how to build the image the way described:

```bash
docker build \
  --cache-from $DEVELOPMENT_IMAGE_MAIN_BRANCH_TAG \
  --cache-from $DEVELOPMENT_IMAGE_CURRENT_BRANCH_TAG \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  --file Development.Dockerfile \
  --tag $DEVELOPMENT_IMAGE_TAG \
  --tag $DEVELOPMENT_IMAGE_CURRENT_BRANCH_TAG \
```

By setting `BUILDKIT_INLINE_CACHE` build argument, we write cache metadata in the container so it can be used as cache source for later builds.

After the build is done, it is pushed to the container registry:

```bash
docker push --all-tags $DEVELOPMENT_IMAGE
```

The final `.gitlab-ci.yml` will be:

```yml

stages:
  - prepare
  - test

variables:
  CARGO_HOME: ${CI_PROJECT_DIR}/.cargo
  DEVELOPMENT_IMAGE: $CI_REGISTRY_IMAGE/development
  DEVELOPMENT_IMAGE_MAIN_BRANCH_TAG: $DEVELOPMENT_IMAGE:main
  DEVELOPMENT_IMAGE_CURRENT_BRANCH_TAG: $DEVELOPMENT_IMAGE:$CI_COMMIT_BRANCH
  DEVELOPMENT_IMAGE_TAG: $DEVELOPMENT_IMAGE:$CI_COMMIT_SHORT_SHA

docker:build:development:
  stage: prepare
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - |
      docker build \
        --cache-from $DEVELOPMENT_IMAGE_MAIN_BRANCH_TAG \
        --cache-from $DEVELOPMENT_IMAGE_CURRENT_BRANCH_TAG \
        --build-arg BUILDKIT_INLINE_CACHE=1 \
        --file Development.Dockerfile \
        --tag $DEVELOPMENT_IMAGE_TAG \
        --tag $DEVELOPMENT_IMAGE_CURRENT_BRANCH_TAG \
        .
    - docker push --all-tags $DEVELOPMENT_IMAGE

cargo:test:
  stage: test
  image: $DEVELOPMENT_IMAGE_TAG
  cache:
    policy: pull-push
    paths:
      - .cargo/.crates.toml
      - .cargo/.crates2.json
      - .cargo/bin/
      - .cargo/registry/index/
      - .cargo/registry/cache/
      - .cargo/git/db/
      - .cargo/bin
      - target/
    when: on_success
  script:
    - cargo test

```

If you want more details about this last change, please go ahead and take a look at [it's commit](https://gitlab.com/yaghmaie/development-container/-/commit/7a4f81f0d0de2401ed18626da30e24b11d3cdd32).

## Conclusion

Despite all the advancement with development tools, keeping a CI pipeline fast while reliable can be challenging. Working on a CI pipeline without tests and builds is sort of meaningless. So, in this post we started with a "hello, world" rust project and progressed into calling a gRPC API, then making development container for builds and tests in the CI. All the files and related date for this exercise can be found at [**Development Container repository**](https://gitlab.com/yaghmaie/development-container).
