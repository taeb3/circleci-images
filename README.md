**Deprecation Notice:** the legacy Convenience Images (the images in this repo) are now deprecated. They have been replaced by the next-gen Convenience Images. We will no longer be accepting PRs for features or improvements to these images. Instead, please migrate over to the new images. You can view and ask questions about the deprecation on [CircleCI Discuss](https://discuss.circleci.com/t/legacy-convenience-image-deprecation/41034). You can browse next-gen Convenience Images and what's available via the [CircleCI Developer Hub](https://circleci.com/developer/images). You can find the repos for next-gen images [here](https://github.com/CircleCI-Public?q=cimg-&type=&language=&sort=).

---

# CircleCI Images [![CircleCI Build Status](https://circleci.com/gh/circleci/circleci-images.svg?style=shield)](https://circleci.com/gh/circleci/circleci-images) [![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](https://raw.githubusercontent.com/circleci/circleci-docs/master/LICENSE) [![CircleCI Community](https://img.shields.io/badge/community-CircleCI%20Discuss-343434.svg)](https://discuss.circleci.com)

## Stay informed about CircleCI image changes/announcements
As part of regular maintenance, changes are occasionally made to various images, from updating images' contents, to changing how image variants are tagged. With the exception of bugfixes or security patches, these changes will always be announced in advance. Changes are posted in the Announcements section of CircleCI Discuss; relevant posts will always have a `convenience-images` tag:

- https://discuss.circleci.com/c/announcements
- https://discuss.circleci.com/tags/convenience-images

By creating a Discuss account, you can subscribe to these posts, in order to receive notifications via email:

https://discuss.circleci.com

## Overview
A set of convenience images that work better in context of CI. This repo contains the official set of images that CircleCI maintains. It contains language as well as services images:

* Language images (e.g. `ruby`, `python`, `node`) are images targeted for common programming languages with the common tools pre-installed. They primarily extend the [official images](#official-images) and install additional tools (e.g. browsers) that we find very useful in context of CI.
* Service images (e.g. `mongo`, `postgres`) are images that have the services pre-configured with development/CI mode. They also primarily extend the corresponding [official images](#official-images) but with sensible development/CI defaults (e.g. disable production checks, default to nojournal to speed up tests)

## Official images
We extend [Docker Official Repositories](https://docs.docker.com/docker-hub/official_repos/) in order to start with the same consistent set of images.

This allows us to make things more standardized. From our scripts for checking for updates, the type of OS on the base image, and so forth. We can recommend using `apt-get install` rather than documenting various constraints depending on which stack you're using.

The official images on Docker Hub are curated by Docker as their way to provide convenience images which address both development and production needs. Since Docker sponsors a dedicated team responsible for reviewing each of the official images, we can take advantage of the community maintaining them independently without trying to track all of the sources and building automations for each one. For now we can take a shortcut, without building this infrastructure.

Finally, our convenience images are augmenting these official images, by adding some missing packages, that we install ourselves for common dependencies shared for the CI environment.

All of the official images on Docker Hub have an "_" for the username, for example:
https://hub.docker.com/_/ruby

You can view all of the officially supported images here:
https://hub.docker.com/explore/

CircleCI supported images are here:
https://hub.docker.com/r/circleci/

To view the Dockerfiles for CircleCI images, visit the [CircleCI-Public/circleci-dockerfiles](https://github.com/circleci-public/circleci-dockerfiles) repository.

# How to add a bundle with images
A bundle is a top-level subfolder in this repository (e.g. `postgres`).

For the image Dockerfiles, we use a WIP templating mechanism. Each bundle should contain a `generate-images` script for generating the Dockerfiles. You can use [`postgres/generate-images`](postgres/generate-images) and [`node/generate-images`](node/generate-images) for inspiration. The pattern is executable script of the following sample:


```bash
#!/bin/bash

# the base image we should be tracking. It must be a Dockerhub official repo
BASE_REPO=node

# Specify the variants we need to publish.  Language stacks should have a
# `browsers` variant to have an image with firefox/chrome pre-installed
VARIANTS=(browsers)

# By default, we don't build the alpine images, since they are typically not dev friendly
# and makes our experience inconsistent.
# However, it's reasonable for services to include the alpine image (e.g. psql)
#
# uncomment for services

#INCLUDE_ALPINE=true

# if the image needs some basic customizations, you can embed the Dockerfile
# customizations by setting $IMAGE_CUSTOMIZATIONS. Like the following
#

IMAGE_CUSTOMIZATIONS='
RUN apt-get update && apt-get install -y node
'

# boilerplate
source ../shared/images/generate.sh
```

By default, the script uses `./shared/images/Dockerfile-basic.template` template which is most appropriate for language based images. Language image variants (e.g. `-browsers` images that have language images with browsers installed) use the `./shared/images/Dockerfile-${variant}.template`.

Service image should have their own template. The template can be kept in `<bundle-name>/resources/Dockerfile-basic.template`—like [`./mongo/resources/Dockerfile-basic.template`](./mongo/resources/Dockerfile-basic.template).

To build all images—push a commit with `[build-images]` text appearing in the commit message.

Also, add the bundle name to in Makefile `BUNDLES` field.


## Development
*This section is a work in progress*

This project's build system is managed with `Make`. It generates Dockerfiles and builds images based off of upstream Docker Library base images, variant images that upstream may have, variant images that CircleCI provides, as well as tools common for testing projects in various programming languages.

### Generate Dockerfiles
Use `make <base-image>/generate_images` to generate of all of the Dockerfiles for a specific base image. For example, to generate all of the Dockerfils for the `circleci/golang` image, you would run `make golang/generate_images`.

Use `make images` to generate Dockerfiles for **every** base image we have. This process is fairly fast with decent Internet connection.

Once generated, Dockerfiles for each image will be in the images folder for that base image (e.g. ./python/images/).

### Build images

#### Build a single Dockerfile
You can build a single image, with a throway name and the `docker build` command.

`docker build -t <throw-away-img-name> <directory-containing-dockerfile>`

Here's how you would build the "regular" Go v1.11 CircleCI image:

`docker build -t test/golang:latest golang/images/1.11.0/`

The image name and tag (`test/golang:latest`) can be whatever name you want, it's just for local dev and can be deleted.

#### Build all Dockerfiles for a base image
Use `make <base-image>/publish_images` to `docker build` all of the Dockerfiles for that base image. There's a couple things to note here:

1. Each base image has **a lot** of images. Building them will end up taking several GBs of disk space, and can take quite a while to run. Make sure you want to do this before you do it.
1. The build script will also try to run `docker push`. If you don't work for CircleCI, this will fail and that's okay. It's safe to ignore.

#### Build all Dockerfiles for every base image
Don't do it. If you have an Ultrabook laptop, it won't be happy. If you really want to do it, look at the `Makefile`.

### Testing
There is automated testing for every variant of every image!

Tests run with [dgoss](https://github.com/aelsabbahy/goss/tree/master/extras/dgoss).

Tests are platform-specific (e.g., PHP and Android have their own distinct sets of tests)—see the `goss.yaml` file in each image directory.

The testing logic is currently in `shared/images/build.sh`, as it is nestled between the existing automated `docker build` and `docker push` functionality. In short, for a particular variant of an image, we make a copy of its Dockerfile, append some Dockerfile syntax to add a custom entrypoint that allows us to execute tests against the running container, build a special, temporary version of the image (added time is insignificant, as most of the Dockerfile steps are cached), and run the tests.

A particular variant is pushed to Docker Hub only if tests pass; if not, the same process restarts for the next variant, etc.

Tests run twice: once for `stdout`, and again, with JUnit formatting, for `store_test_results` (after some post-processing [due to how goss outputs JUnit XML](https://github.com/aelsabbahy/goss/blob/master/outputs/junit.go)...). With test runtimes at essentially zero seconds, running everything twice has a negligible effect on job runtime.

By default, a given branch will push images to the [`ccitest` Docker Hub org](https://hub.docker.com/r/ccitest), with the branch name appended to all image tags. Once a given branch is merged to staging, staging will then push images to the [`ccistaging` Docker Hub org](https://hub.docker.com/r/ccistaging). Finally, we can rebuild those images on the [`circleci` Docker Hub org](https://hub.docker.com/r/circleci) by merging changes from `staging` into the `master` branch. See below for note on forked pull requests.

#### Remaining work
- Tests are very bare-bones right now and could use image-specific additions—please add things to each image's `goss.yml` file and the existing logic will take care of the rest!
- The testing code is spread across the repository and is a bit confusing; some refactoring would help

## Limitations
* We welcome and appreciate contributions (issues, PRs) from the community! However, for security reasons, forked pull requests will not push Dockerfiles to [CircleCI-Public/circleci-dockerfiles](https://github.com/CircleCI-Public/circleci-dockerfiles), example images to [CircleCI-Public/example-images](https://github.com/CircleCI-Public/example-images), or images to Docker Hub. All Dockerfiles, example image Dockerfiles/READMEs, and images will generate/build in forked PR jobs as expected; however, nothing will be pushed to external CircleCI resources.
* The template language is a WIP—it only supports `{{BASE_IMAGE}}` template. We should extend this.
* We cannot support Oracle JDK for licensing reasons. See [Oracle's Binary Code License Agreement for the Java SE Platform](http://oracle.com/technetwork/java/javase/terms/license/index.html) and [Stack Exchange: Is there no Oracle JDK for docker?](https://devops.stackexchange.com/questions/433/is-there-no-oracle-jdk-for-docker) for details.


## Licensing & Usage

The `circleci-images` repository is licensed under The MIT License.
See [LICENSE](https://github.com/circleci/circleci-images/blob/master/LICENSE) for the full license text.

The Docker images generated from this repository are designed to work with the CircleCI `docker` executor.
While you may use the images anywhere Docker is available, your experience may vary.
