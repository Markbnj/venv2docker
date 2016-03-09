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
Not to mention that it's a pretty good demo project: in one file you have
concerns of python, docker, and bash scripting. What's not to like?

## Pre-release

venv2docker is early. It has been tested on a very limited number of platforms
(ubuntu 15.10 and debian jessie), against a very limited number of
virtualenvs, and only using python 2.7+. I am also not really a very good bash
programmer, so feel free to laugh at my hacks and curse my bugs, but please do also
open an issue here so I can correct them and learn from you.

Having said that, there is not much risk in your trying this script for the
following reasons:

1. The script does not write to your existing virtualenv.
2. The script does not write to your project directory other than the venv2docker
build folder (which you can change, see "Options" below).
3. The script makes all file-level changes in /tmp.
4. All other writes take place inside the docker image and the host system docker registry.
5. It cleans up after itself.

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

```
venv2docker [OPTION]... [VIRTUALENV]

Create docker images built from an existing virtualenv environment.

If VIRTUALENV is not supplied and a virtualenv is currently active
then that venv will be used.

Example:
    venv2docker --name=myrepo/test testenv

Creates a docker image from the 'testenv' virtualenv and commits
it to myrepo/test.

Options:
    --apt=PACKAGES          Use apt-get to install PACKAGES at build.
    --apt=FILE              Install each package in FILE at build.
    --args=ARGS             Pass ARGS to entrypoint command line.
    -b, --base=BASE         Use BASE as the base image.
    --bin_path=PATH         Install the project folder to PATH in the image.
    -c, --clean             Perform build clean step and exit
.   -d, --debug             Print diagnostic information after build.
    --dockerdir=DIR         Use DIR for build files (defaults to .venv2docker).
    --entrypoint=COMMAND    Execute COMMAND as entrypoint at startup.
    -e, --env=ENV           Set variables in ENV inside image.
    --lib_path=PATH         Install the venv directories to PATH in the image.
    --maintainer=NAME       Use NAME as the maintainer string in the image.
    -n, --name=NAME         Use NAME as the committed image name.
    --no-dangling           Remove an existing image before building.
    --no-log                Don't log docker build output.
    --paths=PATHS           Add PATHS to system path.
    --pip=FILE              Install each package in FILE at build.
    --ports=PORTS           Expose PORTS in image.
    -p, --push              Push the image to the Docker hub after building.
    --pypaths=PATHS         Add PATHS to python path.
    -r, --run               Launch the image after building.
    -s, --skip-image        Generate the dockerfile but skip building.
    -t, --tag=TAG           Use TAG as the tag for the built image.
    -u, --user=USER         Change to USER before running entrypoint.
    -w, --workdir=DIR       Change to DIR before running entrypoint.
```

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

## Relocation and --relocatable

Relocating a virtualenv is what we're doing when we move it into a docker
container, i.e. we are lifting it out of one file system and moving it into
another. From the documentation:

> Normally environments are tied to a specific path. That means that you cannot move an environment around or copy it to another computer.

How are virtualenvs tied to a specific path? The python interpreter itself is
fine with being moved around. It assumes that the directory above its directory
is the 'base' for the installation, and expects all the system libs to hang
off of that. So as far as the interpreter is concerned activating a virtualenv
is a simple matter of making sure the right interpreter (the one in `.virtualenvs/<venv_name>/bin`)
is the first one in the path. The venv2docker script takes care of this by setting
PATH in the image such that the venv python interpreter is first.

The second way that a venv is tied to a specific path is through the "hash-bang"
declaration inserted into installed library scripts in `.virtualenvs/<venv_name>/bin`
and also found in some wheel-related files under `.virtualenvs/<venv_name>/lib` and
`.virtualenvs/<venv_name>/local/lib`. The hash-bang points at the virtualenv python
interpreter using an absolute path (i.e. `/home/mark/.virtualenvs/<venv_name>/bin/python`
in my case). There is an experimental flag, `--relocatable` that aims to solve this
issue by using relative paths.

I didn't want to have to depend on that flag having been set when a venv was created,
so I considered two other potential solutions: 1) install a symlink in the image
to redirect file operations to the right place; or 2) patch the files. I felt that
(1) was messy and not very elegant, while (2) was more invasive, but nevertheless
a pretty straightforward and low-risk operation using sed, so I decided to go that
way.

You do not need to set --relocatable on your virtualenvs to use this tool, and it
hasn't been tested on venvs created that way. It _should_ work, but if you run
into issues with it, or with any other aspect of relocating the env into a docker
container please open an issue here so I can follow up on it.

## Architecture and base image compatibility

Very often the only thing you care about when deploying docker containers is that
the host machine is running a compatible version of the linux kernel. Not so with
virtualenvs. The constraints are a little tighter. The main reason for this is that
virtualenv doesn't replicate your whole system python installation. It's reason
for existing is to isolate installed dependencies. So instead of copying all the
stable system python libraries into the virtualenv it symlinks to them.

What this means is that if you try to move a virtualenv to an architecture in which
the system python install is either missing, or installed to a different location
than the platform on which you built the venv, it will fail to run correctly. If you
start the interpreter in the image and see messages about missing platform-dependent or
-independent libraries, or missing `site.py` then the image into which you are
moving the venv either doesn't have python installed, or has it somewhere else.

## Running the image

## Entrypoints and passing arguments

## Ports

## Debugging techniques

## Examples:

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

`venv2docker --base=ubuntu:15.10 --name=my_repo/my_test_image --tag=beta my_test_env`

The default base image for builds is debian jessie, however you can use the
`-b/--base` option to set any base image you want to use by name.

`venv2docker --base=ubuntu:15.10 --name=my_repo/my_test_image --tag=beta --apt=libxml2,libxslt1-dev my_test_env`

The python dependencies captured by a virtualenv don't always record all the dependencies
of a project. If you need to install packages with apt you can do so using the `--apt`
option as shown above. Packages are installed after `apt-get update -y` is run, and
before any additional pip dependencies are installed.


