> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [snyk.io](https://snyk.io/blog/choosing-the-best-node-js-docker-image/)

> Snyk's Liran Tal discusses strategies for choosing the best node.js Docker image.

Choosing a Node.js Docker image may seem like a small thing, but image sizes and potential vulnerabilities can have dramatic effects on your CI/CD pipeline and security posture. So, how do you choose the best Node.js Docker image?

It can be easy to miss the potential risks of using `FROM node:latest`, or just `FROM node`(which is an alias for the former). This is even more true if you’re unaware of the overall security risks and sheer file size they introduce to a CI/CD pipeline.

The following is an example of a Node.js Dockerfile that is typically given as a reference in Node.js Docker image tutorials and blog posts — but **this Dockerfile is highly flawed and not recommended**:

```
FROM node
WORKDIR /usr/src/app
COPY . /usr/src/app
RUN npm install
CMD "npm" "start"

```

I have previously outlined and provided a step-by-step guide on [10 best practices to containerize Node.js web applications with Docker](https://snyk.io/blog/10-best-practices-to-containerize-nodejs-web-applications-with-docker/), which builds on and improves the example to achieve a production-ready Node.js Docker image.

For this post, we’ll use the contrived example above as the contents of a `Dockerfile` in order to find an ideal Node.js Docker image.

Your options for a Node.js Docker image
---------------------------------------

There are actually quite a few options you could go for when building your Node.js image. They range from the official Node.js Docker image that is maintained by the core Node.js team, to the specific Node.js image tags that you could choose from within that particular Docker base image, and even other options such as building your Node.js application on top of Google’s `distroless` project, or a bare bones `scratch` image provided by the Docker team.

Out of all of these options, which Node.js Docker image is ideal for you?

Let’s take a look at them one by one to learn more about the benefits and potential risks.

**Author’s note:** Throughout this article, I’ll compare a point-in-time Node.js version, which was last released around June 2022 and refers to Node.js 18.2.0.

### The default node image

Let’s start off with the maintained `node` image. It is officially maintained by the [Node.js Docker team](https://github.com/nodejs/docker-node) and contains several Docker base image tags, which map to different underlying distributions (Debian, Ubuntu, or Alpine) as well as to different versions of the Node.js runtime itself. There are also specific version tags to target CPU architectures such as `amd64` or `arm64x8` (the new Apple M1).

The most common `node` image tags for the Debian distribution, such as `bullseye` or `buster`, are themselves based off of `buildpack-deps`, which are maintained by another team.

What happens when you build your Node.js Docker image based on this default `node` image, with just the `fastify` npm dependency?

```
FROM node

```

Build the image with `docker build --no-cache -f Dockerfile1 -t dockerfile1`, and you get the following:

*   We didn’t specify the Node.js runtime version, so `node` is an alias to `node:latest`, which right now points to Node.js version 18.2.0
*   The Node.js Docker image size is 952MB

What is the dependency and security vulnerability footprint for this latest Node.js image? We can run a Snyk-powered container scan with `docker scan dockerfile1`, which reveals the following:

*   A total of 409 dependencies — these are any open source library that was detected using the operating system package manager, like `curl/libcurl4`, `git/git-man`, or `imagemagick/imagemagick-6-common`.
*   A total of 289 security issues, such as Buffer Overflows, Use After Free errors, Out-of-bounds Write, and more, were found inside those dependencies.
*   The Node.js 18.2.0 runtime version is vulnerable to 7 security issues such as [DNS Rebinding](https://snyk.io/vuln/SNYK-UPSTREAM-NODE-2946423), [HTTP Request Smuggling](https://snyk.io/vuln/SNYK-UPSTREAM-NODE-2946427), and [Configuration Hijacking](https://snyk.io/vuln/SNYK-UPSTREAM-NODE-2946717).

Do you really need `wget`, `git`, or `curl` to be available in the Node.js image for your application? Overall, it’s not a pretty picture — having hundreds of dependencies and tooling in the Node.js Docker image, with hundreds of vulnerabilities counted towards them, the Node.js runtime featuring 7 different security vulnerabilities leaves a lot of room for potential attacks.

Node.js Docker Hub options node:buster vs node:bullseye
-------------------------------------------------------

If you browse the available tags on Node.js Docker Hub repository, you’ll find two options for alternative Node.js image tags — `node:buster` and `node:bullseye`.

Both of these Docker image tags are based on Debian distribution versions. The `buster` image tag maps to Debian 10, which enters its _End of Life_ date in August 2022 through 2024 — so it’s not a great choice. The `bullseye` image tag maps to Debian 11, is referred to as Debian’s current stable release, and has an estimated end of life date of June 2026.

**Author’s note:** Because of this, you’re highly encouraged to move all new and existing Node.js Docker images from `node:buster` image tags to `node:bullseye` or other suitable alternatives.

Let’s build a new Node.js Docker image based on:

```
FROM node:bullseye

```

If you build this Node.js Docker image tag and compare to the above results, you’ll get the exact same size, dependency count, and vulnerabilities found. The reason is that `node`, `node:latest`, and `node:bullseye` all point to the same Node.js image tag being built.

Node.js image tag for slimmer images
------------------------------------

The official Node.js Docker team also maintains an image tag that explicitly targets the tooling needed for a functional Node.js environment and nothing else.

These Node.js image tags are referred to with `slim` image tag variant, such as `node:bullseye-slim`, or Node.js version specific such as `node:14.19.2-slim`.

Let’s build a Node.js `slim` image based on Debian’s current stable release `bullseye`:

```
FROM node:bullseye-slim

```

The image size already decreased dramatically — from close to a gigabyte of container image to an image size of 246MB. Scanning its content also shows a great decline in overall software footprint, with 97 dependencies and only 56 vulnerabilities.

The `node:bullseye-slim` is already a better starting point in terms of container image size and security posture.

An LTS Node.js Docker image
---------------------------

So far, our Node.js Docker images were based off of the current version of Node.js, which is Node.js 18. But according to the [Node.js releases schedule](https://nodejs.org/en/about/releases/), this version doesn’t enter its official `Active LTS` status until October 2022.

What if we always relied on the long-term support (LTS) versions in the Node.js Docker images we’re building? Let’s update the Docker image tag accordingly and build a new Node.j image:

```
FROM node:lts-bullseye-slim

```

The slimmer Node.js LTS version (16.15.0) brings a similar number of dependencies and security vulnerabilities on the image, and a slightly smaller image size of 188MB.

So, as it turns out, while you may have specific requirements to choose between `LTS` and `Current` Node.js runtime versions, none of them significantly impacts the software footprint of the Node.js image.

Is node:alpine a better choice for a Node.js image?
---------------------------------------------------

The Node.js Docker team maintains a `node:alpine` image tag and variants of it to match specific versions of the Alpine Linux distributions with those of the Node.js runtime.

The Alpine Linux project is often cited for its incredibly small image size, which is great on the surface because of its smaller software footprint, but how does it stack up?

```
FROM node:alpine

```

Docker image size is relatively the same as `slim` Node.js images at 178MB, but only 16 operating system dependencies and 2 security vulnerabilities were detected in total. This may indicate that the `alpine` image tag is a good choice for a small image size and vulnerability count.

### Is `node:alpine` the better choice for a Node.js Docker image?

Not really. The Alpine for Node.js image variant might provide an overall small image size and even smaller vulnerabilities count. However, the Alpine project uses [musl](https://www.musl-libc.org/) as the implementation for the C standard library, whereas Debian’s Node.js image tags such as `bullseye` rely on the `glibc` implementation. These differences can account for performance issues, functional bugs, or flat-out application crashes. Itamar Turner-Trauring also [wrote](https://pythonspeed.com/articles/alpine-docker-python/) about why you shouldn’t use Alpine for Python Docker images.

Choosing a Node.js `alpine` image tag means you are effectively choosing an unofficial Node.js runtime. The Node.js Docker team doesn’t officially support container image builds based on Alpine. As such, it has to pull Node.js source code from [unofficial builds](https://unofficial-builds.nodejs.org/) and doesn’t guarantee you won’t run into issues. Some notable, and recent, issues with Node.js `alpine` image tag compatibility are:

*   Yarn being incompatible ([issue #1716](https://github.com/nodejs/docker-node/pull/1716)).
*   If you require `node-gyp` for cross-compilation of native C bindings, then Python, which is a dependency of that process, isn’t available in the Alpine image and you will have to sort it out yourself ([issue #1706](https://github.com/nodejs/docker-node/issues/1706)).

Here are some reasons not to choose Node.js Docker images based on Alpine:

*   The Alpine team manages their own security advisories for software they bundle. This means, if they choose _not_ to fix a security vulnerability with a CVE for `libcurl`, then they will not consider that vulnerable and their advisories will not reflect it. This is one of the reasons vulnerability counts are often lower on Alpine base images.
*   Docker security tooling (such as Trivy or Snyk) will not detect runtime related vulnerabilities in Alpine base images. This is why, despite scanning the Node.js 18.2.0 `alpine` base image tag where Node.js 18.2.0 runtime itself is vulnerable, no security vulnerabilities were reported.
*   The way the Alpine project manages its dependency versions across what it calls `repos` does not guarantee the same versions of installed tooling within that Alpine version tag. An [issue](https://gitlab.alpinelinux.org/alpine/abuild/-/issues/9996) from 2 years ago reveals more details for the curious mind.

Distroless Docker Images for Node.js
------------------------------------

The last comparison item for our benchmark is going to be [Google’s distroless container images](https://github.com/GoogleContainerTools/distroless).

### What is a distroless docker image?

These images are even slimmer than the `slim` Node.js image tag because they only target the application and its runtime dependencies. And so, a distroless Docker image has no container package manager, shell, or other general purpose tooling dependencies, giving them a small size and vulnerability footprint.

Lucky for us, the Distroless project maintains a runtime-specific distroless Docker image for Node.js, identified as `gcr.io/distroless/nodejs-debian11` by its complete namespace and available in Google’s container registry (that’s the `gcr.io` part).

Because the Distroless container images have no software, we can use a Docker multistage workflow to install dependencies for our container and copy them to the distroless images:

```
FROM node:16-bullseye-slim AS build
WORKDIR /usr/src/app
COPY . /usr/src/app
RUN npm install

FROM gcr.io/distroless/nodejs:16
COPY --from=build /usr/src/app /usr/src/app
WORKDIR /usr/src/app
CMD ["server.js"]

```

Building this distroless Docker image results in a 112MB file, which is a significant reduction in file size from both the `slim` and `alpine` image tag variants.

If you’re considering using distroless Docker images, there are some Important considerations to make: 

*   They are based on current stable Debian release versions, meaning they’re up to date with a far out _end of life_ expiration date, which is a great thing.
*   Because they are Debian-based, they rely on the glibc implementation and are less likely to surprise you with issues in production.
*   You will soon enough find out that the Distroless team doesn’t maintain fine-grained Node.js runtime versions. This means you need to rely on the general purpose `nodejs:16` tag that will be frequently updated, or install based on the SHA256 hash of the image at a certain point in time.

Comparison of Node.js Docker image tags
---------------------------------------

We can refer to the following table to summarize our comparison across different Node.js Docker image tags: 

<table><tbody><tr><td>Image tag</td><td>Node.js runtime version</td><td>OS dependencies</td><td>OS security vulnerabilities</td><td>High and Critical vulnerabilities</td><td>Medium vulnerabilities</td><td>Low vulnerabilities</td><td>Node.js runtime&nbsp;vulnerabilities</td><td>Image size</td><td>Yarn available</td></tr><tr><td>node</td><td>18.2.0</td><td>409</td><td>289</td><td>54</td><td>18</td><td>217</td><td>7</td><td>952MB</td><td>Yes</td></tr><tr><td>node:bullseye</td><td>18.2.0</td><td>409</td><td>289</td><td>54</td><td>18</td><td>217</td><td>7</td><td>952MB</td><td>Yes</td></tr><tr><td>node:bullseye-slim</td><td>18.2.0</td><td>97</td><td>56</td><td>4</td><td>8</td><td>44</td><td>7</td><td>246MB</td><td>Yes</td></tr><tr><td>node:lts-bullseye-slim</td><td>16.15.0</td><td>97</td><td>55</td><td>4</td><td>7</td><td>44</td><td>6</td><td>188MB</td><td>Yes</td></tr><tr><td>node:alpine</td><td>18.2.0</td><td>16</td><td>2</td><td>2</td><td>0</td><td>0</td><td>0</td><td>178MB</td><td>Yes</td></tr><tr><td>gcr.io/distroless/nodejs:16</td><td>16.17.0</td><td>9</td><td>11</td><td>0</td><td>0</td><td>11</td><td>0</td><td>112MB</td><td>No</td></tr></tbody></table>

Let’s run through the data and insights we learned through each of the different Node.js image tags and decide which is the most ideal.

### Development-parity

If your choice of which Node.js image tag to use comes down to dev-consistency — meaning that you want to optimize for the exact same environment parity of development and production — then this may already be a lost battle. In most cases, all 3 major operating systems use a different C library implementation. Linux relies on glibc, Alpine relies on musl, and macOS has its own BSD libc implementation.

### Docker image size

Sometimes, size matters. More accurately though, the goal isn’t having the smallest of size but the smallest of software footprint overall. In that case, the `slim` image tags aren’t much different in size compared with their `alpine` counterparts, and they’re both averaging about 200MB for a container image. Granted, the software footprint of `slim` images is still quite high (97 vs `alpine`’s 16).

### Security vulnerabilities

Vulnerabilities are an important concern, and have been the center of many articles on why you should be reducing the size of your container images. However, the semantics of security issues matter a lot.

Leaving out the `node` and `node:bullseye` images due to the larger software footprint and increased security vulnerabilities, we can focus on the smaller set of image types. Comparing between `slim`, `alpine`, and `distroless`, the variation between **high and critical** security vulnerabilities isn’t high in absolute numbers and ranges between 0 and 4 — a manageable risk that could be potentially be irrelevant to your application use-case (and thus ignored).

### Support and resilience

Ensuring that the Node.js Docker team is able to prioritize and address concerns with regard to your container image builds, and that issues are resolved in a timely manner, goes a long way. With anything that isn’t the official, Debian-based image tags, you are essentially unable to cross this off your checklist.

When using the `node` or `node:bullseye-slim` image tags, whether you’re opting for a full operating system image or using the slimmer version of dependencies, you are still getting the latest version of the Node.js runtime. While it is an **even** number (Node.js 18.2.0), at the time of this writing, it still hasn’t been included in the Long Term Support lifecycle, which means it will be bundling new versions of other dependent components such as the latest versions of `npm` itself (which has been known for buggy new behavior and requiring time to stabilize).

The bottom line?
----------------

The most ideal Node.js Docker image would be a slimmed-down version of the operating system, based on a modern Debian OS, with a stable and active Long Term Support version of Node.js.

This comes down to choosing the `node:lts-bullseye-slim` Node.js image tag. I’m in favor of using deterministic image tags, so the slight change I’d make is to use the actual underlying version number instead of the `lts` alias.

**The most ideal Node.js Docker image tag is `node:16.17.0-bullseye-slim`**.

If you work within a mature DevOps team that can support custom base images, my second-best recommendation would be Google’s _distroless_ image tag, because it maintains glibc compatibility for official Node.js runtime versions. This workflow will require some maintenance though, so I’d only advise it if you can support that.

Do more to secure your container images with Snyk
-------------------------------------------------

Snyk Container helps you find and fix container vulnerabilities.