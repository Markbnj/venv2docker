# venv2docker

venv2docker is a bash shell script which packages a python virtualenv into
a docker container. The script supports numerous options for controlling
the content of the image at build time, as well as the behavior of the
container at runtime.

## Why?

It's a good question! Physically migrating your virtualenv into a docker
container is probably not a good way to proceed for production deployments,
where most engineers will wante to freeze pip requirements, recreate the
environment inside a clean image, and then clone the code repository into
it. That gives you a declarative recipe for recreating the runtime
environment, as well as confidence that what was committed to the repo is what
is running in the container.

But, not everyone is running CI/CD in a production stack. Someone may instead
be building a simple Wordpress site in a virtualenv at home, and still wish to
take advantage of the ease of deployment that a containerized application
offers. That use-case, and also just wanting to dig into virtualenv and
understand it better, were enough justification for me to create this program.

## Pre-release

venv2docker is early, pre-release code. The only branch at this time is
master. The script has only been tested in ubuntu 15.10 and debian jessie,
running on bash with python 2.7.5+. I would like to make it more portable,
and there is a lot of work still to be done there. In short anything can
change, and certain things such as the package structure are very likely
to.

## Prerequisites

venv2docker requires docker to be installed, of course. If you don't have it
you can [get it here](https://docs.docker.com/linux/step_one/).

A working python and [virtualenv](https://virtualenv.readthedocs.org/en/latest/)
installation is also required.

## Installation

Just grab the repository and either run the script right in the bin directory
or put it on the path. The script is standalone and you can copy it anywhere
you like.

## Use

When you run venv2docker it first finds and verifies either the current active
virtualenv or the one specified by name on the command line. It then packs
up the project files and python dependencies and stages them in a build
folder under the project path (.venv2docker by default). Once the files are in
place the script generates a dockerfile and runs the `docker build` command
against it to create the image. After the build completes the generated
dockerfile and archives remain in the build folder until the next build, or
until `venv2docker --clean` is run.

Pretty much everything else involves the script options which control how
the dockerfile is generated and thus what the content and behavior of the
resulting image is.

### Examples:

`venv2docker`

This command will build an image from the active virtualenv. If no
virtualenv is active the program will print an error and exit. All other
options will be set to their defaults (see below). The resulting image
will have the same name as the virtualenv and be tagged 'latest'.

`venv2docker my_test_env`

Identical to the command above except that it operates on the `my_test_env`
virtual environment, whether or not it is the active virtualenv.

`venv2docker --name=my_test_image my_test_env`

Like the command above this one operates on a specific named virtualenv,
however the resulting image will be named `my_test_image`.

`venv2docker --name=my_repo/my_test_image --tag=beta my_test_env`

This command adds a repository path to the image name, and a tag. The
resulting image will be `my_repo/my_test_image:beta`. A repository
path is required to push the image to the docker hub manually, or using
the `-p/--push` option.