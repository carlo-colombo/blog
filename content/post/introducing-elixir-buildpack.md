---
title: 'Introducing Elixir Buildpack'
date: 2021-02-10T21:55:36+01:00
summary: This is something I am working on (and off) for the last year, it started as a way to play with Elixir and Cloud Native Buildpacks but then I started to use it to create images to deploy on a small [k3s Kubernetes cluster I maintain](https://github.com/carlo-colombo/deployment/blob/master/spec/images.yml#L17-L29).

tags:
    - elixir
    - docker
    - buildpack
---

## TL;DR;

```shell
pack build my-image \
  --path . \
  --builder rg.nl-ams.scw.cloud/carlo-colombo/elixir-builder \
  -e DEBUG=1 \
  -e RELEASE=my-release

docker run -it --rm my-image
```

This is not production ready, use it at your own risk. It seems to work for my use cases but it could break for others.

## Introduction

This is something I am working on (and off) for the last year, it started as a way to play with Elixir and Cloud Native Buildpacks but then I started to use it to create images to deploy on a small [k3s Kubernetes cluster I maintain](https://github.com/carlo-colombo/deployment/blob/master/spec/images.yml#L17-L29).

## Buildpacks

[Buildpacks](https://buildpacks.io/) are a tool to declaratively transform source code into container images, initially developed by Heroku, then adopted (and forked) by Cloudfoundry, are now a Cloud Native Computing incubating project supported by both Pivotal (now VMware) and Heroku and used in multiple PaaS. To build an image with Buildpacks a few component are needed: a platform, a builder and a set of buildpacks.

Platforms as [`pack`](https://github.com/buildpacks/pack) or [`kpack`](https://github.com/pivotal/kpack) are the entry point to create an OCI image, they coordinate (through a lifecycle) the execution of a collection of buildpack packaged in the builder.

The builder define the stack, the images that are going to be used during the build phase and then the run image that will be used as base for the final container. In addition it usually packages buildpacks.

Finally the buildpacks, they can detetect if they can contribuite to the final image or to the build process ([`detect`](https://github.com/carlo-colombo/elixir-buildpacks/blob/master/elixir-runtime/bin/detect)) and then they contribuite ([`build`](https://github.com/carlo-colombo/elixir-buildpacks/blob/master/elixir-runtime/bin/build)) adding dependencies as language runtimes (`erlang`, `node`) or building the application code. The buildpacks can also depend on each other defining the order an of execution.

## Elixir Cloud Native Buildpack

To build an OCI image is possibly not enough to just define a buildpack, what I did was to also define a builder image and a stack, in addition I split the buildpack in `erlang-runtime`, `elixir-runtime`, `mix` and the meta buildpack `elixir` that ties all 3 together.

The runtime buildpacks contribuite erlang and elixir in the build phase, while the mix buildpack assemble a release starting from the source code. Only the final release will be present in the final OCI image, while the other dependencies are not going to be included. This achieves a similar result as multistage docker file where the initial stages provide the build dependencies and build the source code while adding only the built code to the final stage (with the needed runtime dependencies).

Notes: The buildpack I created are not really portable to other stacks as they require to have erlang and elixir available on the build image. I did this to avoid downloading over and over the Erlang and Elixir runtime, but probably a more idiomatic way of achieving it could exist.

## Usage

### Basic

```shell
 pack build my-image \
  --path . \
  --builder rg.nl-ams.scw.cloud/carlo-colombo/elixir-builder
```

### Umbrella projects

```shell
pack build my-image \
  --path . \
  --builder rg.nl-ams.scw.cloud/carlo-colombo/elixir-builder \
  -e RELEASE=my-release
```

### Phoenix applications (minimal)

```shell
 pack build my-image \
  --path . \
  --builder rg.nl-ams.scw.cloud/carlo-colombo/elixir-builder \
  -e SECRET_KEY_BASE=<secret key base>
```

## Example

```shell
#create a new phoenix application
mix phx.new hello_phoenix --no-ecto  --no-webpack --no-install

# start the endpoint in MIX_ENV=prod
echo 'config :hello_phoenix, HelloPhoenixWeb.Endpoint, server: true' >> hello_phoenix/config/prod.secret.exs

# create image
# the openssl command can be replaced with a random string
pack build hello_phoenix \
  --path hello_phoenix \
  --builder rg.nl-ams.scw.cloud/carlo-colombo/elixir-builder \
  -e SECRET_KEY_BASE="$(openssl rand -base64 12)"

#start the application
docker run -it --rm -p 4000:4000 hello_phoenix
```

## Debug

### The Elixir Buildpacks

Adding a `DEBUG` environment variable with any value runs the scripts with `set -x` printing all the commands executed.

### The application (connecting to an iex shell)

```sh
docker exec -it <container> \
    /layers/io.github.carlo-colombo.mix-release/release/bin/hello_phoenix remote
```
