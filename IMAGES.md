# CircleCI Convenience Images

CircleCI provides a set of language/stack images that CircleCI has found to be useful for development and use in CI.

Docker Hub provides many official images targeting languages and stacks (e.g. [Node](https://hub.docker.com/r/_/node), [Ruby](https://hub.docker.com/r/_/ruby/), [Python](https://hub.docker.com/r/_/python/), [PHP](https://hub.docker.com/r/_/php), etc).  While these images provide a good starting point and aimed for use in production case, CircleCI sees value in augment them further to provide tools making them better equipped for CI use where users desire more utility tools installed.

# Goals of images

* Quick adoption - have the very common CI tools pre-installed
* Ease Docker learning curve for users new to Docker and CircleCI
* Allow transitioning to custom images later
* Add sensible default configurations for CircleCI environment (e.g. sensible default memory limit for Java images)

# Non-Goals

* Install everything!
* Target polyglot applications.  However, if we find that a community use another language tools frequently, we may install it.  (e.g. node/yarn/hamlc/cass may be installed for the Rails related images, but not for Golang images).

# Trade offs

When goals are in conflict, use the following rules of thumb:

* Helping new users onboard >> not breaking existing customers who didn't lock their versions
  * Newly released images should have the latest tools with best practices
  * e.g. images should have the latest non-vulnerable git, ssh, ruby, etc
  * Onboarding process should encourage locking down docker versions or create their own images; and how to test new images
  * Obviously, don't try make significant changes unnecessarily - as they impact docs and shared knowledge about images

* Following Docker official images >> establishing our best practice
  * Docker official language/service images currently represent the widely adopted community practices and their flaws/quarks are easier to find and work-around and get more eyes and contributes to get things up to date.
  * If we diverge a lot, users will have a harder time figuring out our image flaws (which we will ultimately do), it will be harder for us to maintain as new images come out, and it will be harder for users to fork when they want to customize their own image
  * e.g. we should track the official node images in Docker Hub in release and use the same npm/yarn version installed there.  If a new version of node/yarn is released, wait until it's released in Docker Hub (which happens almost momentarily)
  * e.g. if old versions of node don't have yarn - we shouldn't install it in our convenience image.  This is also in line with favoring new users over supporting legacy software.
  * e.g. if official images use mutable tags and `ruby:2.4.1` is updated with newer git/ssh/etc, our `circleci/ruby:2.4.1` needs to be updated to match the underlying updates too

# Practices

All CircleCI convenience images must do the following:

* When appropriate, extend the language official images provided by Docker Hub and track them
* Install common development tools (e.g. `git`, `ca-certificates`, `ssh`, `tar`, `curl`, `wget`, `nc`, `ifconfig`).  Pretty much that what [`buildpack-deps` image](https://hub.docker.com/_/buildpack-deps/) contain
  * Versions of these tools may change between versions - and we aim to install the latest available.  If user needs to pin any of them, they are encouraged to lock the docker version or use the Dockerfile to build their custom image
* Default to using a non-root user, namely `circleci` - which has passwordless sudo
* Images should be published in Docker Hub as `circleci/<name>:<version>`, e.g. `circleci/node:7.3.2`
* Images should be rebuilt built in frequent basis (at least once a week), and upon notice of new upstream releases (e.g. new node version released)
  * This is done to automatically track changes in official images
  * Official images and upstream repos are frequently updated to include security fixes and maintenance updates.  Our images should get these security updates applied
  * There isn't a good mechanism to track changes in official images, so periodic changes to rebuild images are the best we can

# Attitude about images and lifecycle

* Convenience images should be designed with new docker and CircleCI users in mind, to try to get them to green faster
* Helping new users >> not breaking existing customers
  * Obviously, we should avoid trying to break existing users as much as possible.


# Notes about language/stack images

The tools that are installed on the image should have the following patterns:

* Install Commonly used development tools used, e.g. `ssh`, `tar`, `curl`, `wget`, etc - and any tools that we recommend for use in our documentation (e.g. `dockerize`, `jq`)
* Install Language specific common tools, e.g. `npm` for node images, `pip` for python images, `bundle` for ruby images
* When in doubt, don't pre-install it in the basic image
* Installed tools should be good about backward compatibility and we should install the latest tools as much as possible.  Avoid installing software where users would significantly care about pinning.
  * GOOD: `ssh`, `curl`, `nc` -- pretty much everyone the latest version and vast majority wouldn't pin ssh
  * BAD: database clients!  Users typically want to pin which postgres/rabbitmq client they use - avoid installing it in base image
  * FUZZY: There are some fuzzy cases.  These should be dealt with in case-by-case basis and it's a judgement call.  If they are very commonly used and generally users use the latest, install the latest - but educate users on how to opt out.  For example:
    * Language package managers (e.g. `npm`, `pip`).  Users pretty much expect these tools to be installed.  We should install the official maintained software at any given time and educate users how to downgrade or create their own image to pin down the version if they wish.
    * Browsers.  If users opt in to use the images with browsers pre-installed, they should expect to use the latest released stable browser.  Official browsers are good at backwards compatibility, and with browsers auto-updating they serve as canaries for breakages to come.  Yet, this can easily introduce irreproducability in builds (e.g. due to browser being incompatible with Selenium version, chrome is out of sync with chromedriver).  In such cases, we should install the latest in the images and educate users about the problem by encouraging them to pin the image to a known docker image after being onboarded successfully.

* Images can have multiple variants.  The basic image should install the tools needed for the basic use of that stack.  Each basic image is supplemented with variant to address common uses in the stack.  For example, we publish `browser` variant with common browsers installed for users with web testing needs.

* No language specific version managers should be used in the official CircleCI images.  No rvm/rbenv, pyenv, nvm, etc should be installed.  Each image should install a single version.

* [EXPERIMENTAL] If a community frequently uses polyglot tools, we can publish a variant with the related tools.  For example, the ruby web community frequently uses javascript/node as part of their builds, e.g. for asset pipelines, frontend creation.  This is a pretty common pattern and we are unclear what the solution is.  Thus, we are proposing the following line in the sand implementation that we think is the 80/20 solution:
  * Hypothesis:
    * this pattern of use is NOT bi-directional, e.g. ruby users use node tools; but node users don't use ruby tool.
    * Most frequently users need the latest version of the other language, e.g. Ruby projects mostly care about what version of Ruby they use, but are OK with the latest stable node version.
  * Experimental solution:
    * We publish a variant image under the primary image (e.g. `ruby:2.4.1-node`) that has the exact version of ruby installed and the latest stable node.
    * Users who got their builds green, can pin the entire image
    * Users who need a specific version of Node will be guided on how to pin the version by modifying the `ruby:2.4.1-node` Dockerfile so they have their desired image.


* For now and until getting clearer signal from users, the following tools should NOT be installed:
  * Editors and tools useful in ssh.  When we add ssh capability in 2.0, we can re-examine
  * deployment tools, e.g. `awscli`, `ansible`, `heroku`.  We may consider adding them, but in the first iteration, we are erring in the side of not including them
  * Docker tools e.g. `docker`, `docker-compose`.  It's not clear users are always happy with the latest version.  We can provide recipes for installing the right docker version.


# Notes about Services images

* Use official images as much as possible!  Also these should be as much as possible a thin layer of the official images - only configuration files or environment variable changes if possible.
* Configure them for CI environment and for speed, if the behavior doesn't change the visible behavior of common tests.  Can disable any production startup checks, journaling (e.g. used for failover and crash handling), use in-memory storage if possible, etc.
* Provide sensible default for username/password if users frequently need to configure them (e.g. `MYSQL_ALLOW_EMPTY_PASSWORD`)
