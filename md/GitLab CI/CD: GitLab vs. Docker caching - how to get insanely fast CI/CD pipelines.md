> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.augmentedmind.de](https://www.augmentedmind.de/2022/06/12/gitlab-vs-docker-caching-pipelines/)

> Learn how Docker-based caching can speed up your GitLab CI/CD pipelines even more than GitLab's built......

Using a Node.js example project I demonstrate how Docker-based caching can speed up your GitLab CI/CD pipelines even more than GitLab’s built-in caching mechanism. I explain how each approach works, and what the technical prerequisites are. I also list tools that support you with setting up a Docker-based CI pipeline.

Introduction
------------

**Caching of files between CI jobs is a very common approach to speed up CI pipelines. Examples** for such files **are:**

*   A **dependency cache folder**, where the dependency/package manager stores previously downloaded packages. E.g. `/root/.npm` for NPM, or `/root/.cache` for Python’s `pip`. Given that this folder is cached, a CI job that needs to install dependencies finishes faster, because it no longer needs to re-download those dependencies already in the cache.
*   **Installed dependencies**, e.g. the `node_modules` folder for a NPM-based project, or the `venv` folder for a Python project. If this folder is cached, commands such as “`npm install`” or “`pip install -r requirements.txt -U`” complete faster, because the package manager (here: `npm` or `pip`) can omit upgrading/replacing/downloading packages that are already installed.
    *   **Note: be careful with actually caching such dependency folders, because there is a risk for unintended side effects,** possibly resulting in _non-reproducible_ builds. For instance, in CI jobs that deal with Node.js, you should use “`npm ci`” instead of “`npm install`“, to detect drift between the `package.json` and `package-lock.json` file. However, “`npm ci`” does not make use of an existing `node_modules` folder, but deletes that folder (if it exists) right away and recreates a new one, to ensure that you get a _clean_ state of your dependencies. For Python, the caveat is that the `venv` folder might contain more dependencies than you need, because `pip` does not uninstall those dependencies that were already installed (from earlier jobs or other branches), but are not declared in the `requirements.txt` file. Having superfluous dependencies installed can be problematic, due to Python’s dynamic nature, where libraries try to load other libraries (failing silently), changing their behavior in case of success.
*   **Intermediate build artifacts**: these are byproducts created while compiling/transpiling an application. E.g. object files generated when compiling a C++ application, or the `node_modules/.cache` folder when bundling JavaScript/TypeScript code with `webpack`. If you cache them, a follow-up compilation will complete much faster, because the compilers/transpilers intelligently detect these files, and can therefore skip many (if not all) compilation steps.

**Generally, with GitLab CI/CD there are two caching mechanisms you can use (also in combination):**

*   **GitLab CI/CD caching**, where you define a **`cache`** key for the jobs in your `.gitlab-ci.yml` file
*   **Docker build cache** (using the BuildKit backend)

**In this article, I will explain both approaches, followed by recommendations regarding which approach to use, depending on the types of GitLab _runners_ you use.**

**Throughout the article, I will reference a GitLab [demo project](https://gitlab.com/MShekow/gitlab-vs-docker-caching) that illustrates both approaches.**

GitLab CI/CD caching
--------------------

**The general approach of GitLab’s built-in caching mechanism is as follows:**

*   **Right before a job starts, the GitLab runner tries to retrieve a zip archive that contains the cache from some location** (more details below) **and extracts this zip file to the current project directory**
*   **At the end of a job, the GitLab runner zips all those files/folders specified in the CI job’s** `**cache.paths**` (in your `.gitlab-ci.yml` file), **and stores the zip in some location**

**The concrete cache zip storage location depends on how the runners are configured by the administrator: there are two different configuration modes:**

*   **_Local_ cache (default for self-hosted runners)**:
    *   The GitLab runner manages a local _cache folder_ (see [docs](https://docs.gitlab.com/ee/ci/caching/#where-the-caches-are-stored)) that contains all the caches as zip files. If the GitLab runner is installed “natively” (e.g. with `apt-get` on a Debian/Ubuntu VM), this folder is just a normal directory on the host. If the runner is operated within a Docker container, the cache folder is inside a _Docker volume_ managed by the GitLab runner.
    *   Whenever a new CI job is sent to the runner, the runner tries to find the matching zip archive in the local cache folder, and if it finds one, it is extracted to the job directory (the extraction takes some time)
    *   At the end of a job, the runner zips the files/dirs specified by the “`cache`” key in the `.gitlab-ci.yml` again, and moves the zip file to the local cache folder
*   **_Distributed_ cache** ([docs](https://docs.gitlab.com/runner/configuration/speed_up_job_execution.html#use-a-distributed-cache), **the default for the SaaS runners on GitLab.com**):
    *   The admins set up a central cloud-based storage server (e.g. S3-based) that stores the cache zip files for all projects. The runners are configured to use this cloud storage.
    *   Whenever a new CI job is sent to the runner, the runner tries to download the matching zip archive from the cloud storage, and if the download was successful, the zip file is extracted to the job directory (downloading and extracting takes some time)
    *   At the end of a job, the runner zips the files/dirs specified by the “`cache`” key in the `.gitlab-ci.yml` again, and uploads the zip file to the cloud storage (which takes time)

**In general, there are a few tricks you can apply when using GitLab’s caching:**

*   Set the GitLab variables `**FF_USE_FAST_ZIP**` and related variables to speed up the zipping process ([docs](https://docs.gitlab.com/ee/ci/runners/configure_runners.html#artifact-and-cache-settings))
*   Disable uploading/updating the cache in those CI jobs that only need to _read_ the cache, by setting a cache `**policy**` ([docs](https://docs.gitlab.com/ee/ci/yaml/#cachepolicy))
*   Avoid _cache trashing_ by intelligently picking the cache `**key**`, e.g. file content of `package-lock.json` when caching the `node_modules` folder ([docs](https://docs.gitlab.com/ee/ci/yaml/#cachekeyfiles))
*   Use _multiple_, more fine-grained `**cache**` definitions per job: each cache is smaller, and can therefore be also retrieved more efficiently. E.g. _build cache_ vs. _test cache_ vs. _dependency cache_ ([docs](https://docs.gitlab.com/ee/ci/caching/index.html#use-multiple-caches))

**Take a look at the** `**gitlab-caching**` **branch in [this demo repository](https://gitlab.com/MShekow/gitlab-vs-docker-caching) to see a concrete example for a Node.js application.**

Docker-based caching using Docker’s build cache
-----------------------------------------------

**The Docker build cache** (when using the BuildKit build engine) is essentially a large collection of _locally_-stored binary files (managed by the Docker and BuildKit daemons) that **can be used for two purposes**:

*   **_Image layer_ caching**: contains all the (intermediate) image layers (corresponding to the statements in your `Dockerfile`) as binary files, together with meta-data (the _cache key_) that helps the build engine to decide whether a cached layer can be used during a build, or whether that layer needs to be completely rebuilt. For `COPY` statements, Docker’s cache invalidation algorithm considers the (recursive) hashes of the files/folders you are `COPY`ing into the image from the build context.
*   **_Directory_ caching**. An example would be “`RUN --mount=type=cache,target=/root/.cache,id=pip **pip install -r requirements.txt**`“. This is useful to temporarily mount _dependency cache folders_ (discussed in the _Introduction_) of package managers, such as `pip`, `npm` or `apt`/`yum`/etc., which are shared between consecutive image builds, because the _source_ of the mount point is actually on the _host_.

Docker build cache persistency warning

**The remainder of this article assumes that you install the GitLab runner on a _fixed/static_ fleet of machines, to actually see any speed-ups.** Here is why: the Docker build cache is a _local_ cache, managed by the Docker daemon on the host where the deamon is installed. For static machines, this local cache is persistent, in the same way the build cache is persistent on your developer laptop, where building a particular image for the second time is much faster than building it the first time.

However, if you use GitLab.com’s shared SaaS runners, you must declare a **`service: [docker:dind]`** in the `.gitlab-ci.yml`, which uses Docker-in-Docker, creating a _temporary_ Docker daemon for each CI job. The local build cache of that temporary daemon always starts out being _empty_. Consequently, the speed-up effects vanish, because the temporary Daemon’s local build cache is not persistent!

**The basic idea why Docker’s build cache speeds up CI jobs is this: instead of putting statements in the** `**script**` **section of your** `**.gitlab-ci.yml**` **file, you put them into a** `**Dockerfile**` **– even if you don’t need or want a Docker image in the end. The** `**Dockerfile**` **is actually a _multi-stage_** `**Dockerfile**` **(see [docs](https://docs.docker.com/develop/develop-images/multistage-build/)). Inside the** `**Dockerfile**`**, you define several _targets_ / stages (e.g. “install dependencies”, “build”, or “test”). In each of the CI jobs, you then run** `**docker build**` **for a specific target. For instance, the CI job that runs the tests then runs docker build for the “test” target.**

Take a look at the `**docker-caching**` branch in [this demo repository](https://gitlab.com/MShekow/gitlab-vs-docker-caching) to see the same Node.js example application, ported to using the Docker-based approach. Note that you should carefully craft the `.dockerignore` file to maximize the caching efficiency. Otherwise, you might end up with _always-changing_ files in your Docker _build context_ (e.g. the `.git` folder), which invalidates the Docker build cache when you have statements such as “`**COPY . .**`” in your `Dockerfile`.

Comparison of both approaches
-----------------------------

**Understanding each approach works best by looking at a concrete example. Visit the [demo repository](https://gitlab.com/MShekow/gitlab-vs-docker-caching) and look at the two branches** `**docker-caching**` **and** `**gitlab-caching**` **to understand how each approach works.** If you switch to the pipeline view in that GitLab project, you can see the execution times of each branch. **I ran two pipelines per branch, where the first pipeline needs to start from scratch (empty cache) and the second pipeline has everything cached already.**

**As you can see, the speed-up effect of GitLab caching is rougly 2x, whereas the speed-ups of Docker-based caching are roughly 6x.**

### Why and when Docker-based caching is faster

Assuming that you use a _static_ number of GitLab runner machines with Docker daemon installed (whose Daemon socket is mounted into the CI job containers, see [docs](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-docker-socket-binding)), **using Docker’s build cache (over GitLab CI/CDs built-in caching) is faster for the following reasons**:

*   **Assuming that a cachable artifact (e.g. an image layer) is already present in Docker’s _local_ build cache, the runner can use it _instantly_.** You are not wasting time with compressing/decompressing/downloading cache zip files, which would happen with _GitLab_‘s caching mechanism.
*   **When using _GitLab_‘s caching, you must treat the cache as _unreliable_.** **Consequently, you _always_ have to run the commands that ensure that the content of the cached folder(s) is up-to-date again** (such as “`yarn install`“). Given a filled cache, such commands execute faster (compared to running them against an _empty_ cache), but they still take some time. Exemplary, in the demo project, `**yarn install**` takes 2-3 seconds for a perfectly filled cache. However, **with _Docker_‘s layer caching, the cached layers are downloaded in a _reliable_ way, and thus you need to run such commands that populate the cache (such as “**`**yarn install**`**“) _only once_.**
*   **Docker’s image layer caching often avoids that the commands that achieve the actual goal of a CI job** (e.g. building or testing your application) **need to be executed at all, in case they are already cached**. Take a look at the [build job log](https://gitlab.com/MShekow/gitlab-vs-docker-caching/-/jobs/2398310487), which reveals that the `script` section completed in 6 seconds, because every command (including `RUN yarn build`) was cached. In contrast, with _GitLab_ caching, the commands (such as “`yarn build`” in the example project) are _always_ executed.

### When to use GitLab’s CI/CD caching

**There are of course also a few reasons why you might want to prefer the traditional GitLab caching mechanism:**

*   As indicated in the above box (_Docker build cache persistency warning_), you should prefer GitLab’s caching mechanism over Docker-based caching **whenever you use dynamically-provisioned runners that don’t have access to their own persistent Docker daemon**.
*   Your runners require access to a Docker daemon. Best speeds are achieved when you control your own fleet of (few) runners. In [this article](https://www.augmentedmind.de/2022/04/17/operate-self-hosted-gitlab-runner/) I explain how to set up such runners. If you cannot set this up, then this approach is not for you.
*   With Docker-based caching, **there are a few other minor issues that might annoy you**:
    *   **Exposing files as GitLab artifacts that were built inside a container is more complex. See [here](https://gitlab.com/MShekow/gitlab-vs-docker-caching/-/blob/docker-caching/.gitlab-ci.yml#L50-L55) for a workaround.**
    *   **The output of the command is polluted due to BuildKit**. The output does not only contain the output of your actual `Dockerfile` statements, but also all other kinds of output of the BuildKit daemon. This makes it a bit more difficult to read the CI job log, or diagnosing failing jobs.

Tool support for Docker-based builds
------------------------------------

**If you decide to use Docker-based builds, moving commands from the `script` section of the `.gitlab-ci.yml` file to a `Dockerfile` with multiple stages, you might as well use dedicated tooling. There are tools (requiring Docker) that help you define CI pipelines (and their jobs) in a CI-vendor independent language, and they also let you run the entire pipeline locally on your development machine. I found the following tools in this area:**

*   [**Earthly**](https://docs.earthly.dev/): instead of a `Dockerfile` you write an `Earthfile` instead, which is a blend of `Dockerfile` and `Makefile`
*   [**toast**](https://github.com/stepchowfun/toast): uses a YAML file in which you declare tasks (with optional dependencies between tasks), which looks somewhat similar to `gitlab-ci.yml` files
*   [**Dagger**](https://dagger.io/): uses the CUE language to define pipelines

Conclusion
----------

As I’ve illustrated, Docker-based caching can achieve much better speed-up effects, compared to GitLab’s built-in caching (6x vs. 2x). However, these speed-ups come at a operative cost: you need to maintain a (static) fleet of (virtual) machines that have runners installed (which have a _local_ on-disk cache), and keep them up-to-date over time. The fact that the number of runners is _static_ also means that _if_ your pipeline workload _varies_ strongly, you may have to “over-provision” your GitLab runner fleet, resulting in higher per-month server costs, compared to _dynamic_ scaling approaches (e.g. using GitLab’s [SaaS runners](https://docs.gitlab.com/ee/ci/runners/), dynamic runner scaling [on AWS](https://docs.gitlab.com/runner/configuration/runner_autoscale_aws/), or using the [Kubernetes executor](https://docs.gitlab.com/runner/install/kubernetes.html)), where short-lived CI job execution environments (e.g. VMs or Kubernetes pods) only cost you money while they are running. The downsides of the dynamic approach is that costs are more difficult to predict, and that the execution environments are short-lived: this makes your jobs slower because they must always download/upload cache contents over the network (which is slower than using a local disk), and because a Docker daemon (in case you need one) must always be dynamically created (via DinD) each time a job runs.

In general, as with any optimization measure, you should not rely on what other people (including me) say. **Always measure yourself how big the effect of the optimization (here: caching) is.** For instance, you may discover that caching large folders (such as _dependency cache_ folders like “`node_modules`“) is not worth it if your runners have a very fast internet connection anyway, where re-downloading packages does not take long. This is especially true if you use GitLab’s SaaS runners, which have to download the cache from the (distributed) cloud storage anyway.