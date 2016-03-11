# venv2docker

venv2docker is a bash shell script which packages a python virtualenv into
a docker container. The script supports numerous options for controlling
the content of the image at build time, as well as the behavior of the
container at runtime.

## Releases

##### [0.1.1-beta](https://github.com/Markbnj/venv2docker/releases/tag/0.1.1-beta)
  * 3/11/2015, added code to install python into debian:jessie by default.

##### [0.1.0-beta](https://github.com/Markbnj/venv2docker/releases/tag/v0.1.0-beta)
  * 3/11/2015, first primarily feature-complete release.

## Documentation

### Table of Contents
* [Why](#why)
* [Pre-release](#pre-release)
* [Prerequisites](#prerequisites)
* [Installation](#installation)
* [Use](#use)
* [Relocation and --relocatable](#relocation-and---relocatable)
* [Architecture and base image compatibility](#architecture-and-base-image-compatibility)
* [Entrypoints and passing arguments](#entrypoints-and-passing-arguments)
* [Ports](#ports)
* [Running the image](#running-the-image)
* [Debugging techniques](#debugging-techniques)
* [Examples](#examples)
* [Options](#options)
* [Tutorial: a quickie django image](#tutorial-a-quickie-django-image)

### Why?

It's a good question! Physically migrating your virtualenv into a docker
container is probably not a good way to proceed for production deployments,
where most engineers will want to freeze pip requirements, recreate the
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

### Pre-release

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

### Prerequisites

venv2docker requires docker to be installed, of course. If you don't have it
you can [get it here](https://docs.docker.com/linux/step_one/).

A working python and [virtualenv](https://virtualenv.readthedocs.org/en/latest/)
installation is also required.

### Installation

Either grab the latest release archive from the links at the top, or clone the repository
to get latest. You can either run the script right in the bin directory or put it on the path.
The script is standalone and you can copy it anywhere you like.

### Use

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
    -b, --base=IMAGE        Use BASE as the base image.
    --bin-path=PATH         Install the project folder to PATH in the image.
    -c, --clean             Perform build clean step and exit
.   -d, --debug             Print diagnostic information after build.
    --dockerdir=PATH        Use DIR for build files (defaults to .venv2docker).
    --entrypoint=COMMAND    Execute COMMAND as entrypoint at startup.
    -e, --env=ENV           Set variables in ENV inside image.
    --lib-path=PATH         Install the venv directories to PATH in the image.
    --maintainer=NAME       Use NAME as the maintainer string in the image.
    -n, --name=NAME         Use NAME as the committed image name.
    --no-dangling           Remove an existing image before building.
    --no-log                Don't log docker build output.
    --no-project-path       Don't add the project root to the system path.
    --no-project-pypath     Don't add the project root to the python path.
    --paths=PATHS           Add PATHS to system path.
    --pip=FILE              Install each package in FILE at build.
    --ports=PORTS           Expose PORTS in image.
    -p, --push              Push the image to the Docker hub after building.
    --pypaths=PATHS         Add PATHS to python path.
    -r, --run               Launch the image after building.
    -s, --skip-image        Generate the dockerfile but skip building.
    -t, --tag=TAG           Use TAG as the tag for the built image.
    -u, --user=USER         Change to USER before running entrypoint.
    -w, --workdir=PATH      Change to DIR before running entrypoint.
    -y, --no-prompt         Do not prompt for confirmation before building.
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

### Relocation and --relocatable

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

### Architecture and base image compatibility

Often the only thing you care about when deploying docker containers is that
the host machine is running a compatible version of the linux kernel. Not so with
virtualenvs. The constraints are a little tighter. There are at least two reasons
for this.

The first is that virtualenv symlinks to a lot of files and directories in the system
python installation. What this means is that if you try to move a virtualenv to a
different platform it may fail to run if python is in a different place or missing
altogether. It's for this reason that the script automatically installs python into
the default `debian:jessie` base image if you don't. If you start the interpreter in
the image and see messages about missing platform-dependent or -independent libraries,
or missing `site.py` then you have a python installation incompatibility.

The second is simply that virtualenv builds libraries for the environment on which
they are being installed, in some cases even compiling C code in the process. If
you try to move a venv created on a 32-bit platform to one running a 64-bit kernel
it may not work.

The bottomn line is that you can use these tools to move virtualenvs between
compatible platforms. If you violate that constraint YMMV.

### Entrypoints and passing arguments

While you may just want an image you can shell into and play around or test with,
more often your virtualenv will contain some program that you want to run, and
given the nature of containers that program will likely be some sort of server
process or job that you need to start, and to which you need to pass some
arguments for configuration.

This readme doesn't aim to be a docker tutorial. The docker docs [contain a decent
one](https://docs.docker.com/linux/). But there are a couple of things that are
important to know. First, every docker container expects to run a single process
at startup. This process may be a shell, or any other program. It is assigned
PID 1 and when it dies the container transitions from running to stopped.

In a dockerfile (the build template for a docker image) there are two ways to
specify what command you want executed at startup, the ENTRYPOINT and CMD
directives. I'm not going into the details of the differences between them,
but suffice it to say that ENTRYPOINT is what works best for the scenario
that venv2docker was written to enable, and setting the entrypoint in your
image is done through the `--entrypoint` option.

`venv2docker --entrypoint=/usr/local/bin/myprogram`

When you use this option to the script the following is generated into the
dockerfile at build time:

`ENTRYPOINT ["/usr/local/bin/myprogram"]`

The path in this example is absolute, but it may be relative to the current
working directory. You can set the working directory for the image using the `-w|--workdir`
option. It can also be a file on the PATH that you have set inside
the image using the `--paths` argument (Note that venv2docker places your project
root folder on the system path by default).

If you need to pass arguments to the entrypoint command you use the `--args` option
followed by a comma-separated list of strings:

`venv2docker --entrypoint=python --args=manage.py,runserver,0.0.0.0:8000`

When you use these options to the script the following is generated into the
dockerfile at build time:

`ENTRYPOINT ["python", "manage.py", "runserver", "0.0.0.0:8000"]`

Because these arguments are strings and are simply inserted into the right spot
in the dockerfile, you can mostly do anything with them that you might do in
a script, such as refer to an environment variable:

`venv2docker --entrypoint=python --args=manage.py,runserver,${IP}:${PORT}`

If you want to change the behavior of the container after the image is built
you can do it by setting environment variables at run time as shown below. You can
also append new arguments to the list of those to be passed to the entrypoint
by passing them to the `docker run` command:

`docker run -d image_name arg1=1 arg2=2`

If you run the container this way then the behavior is the same as if the following
had been present in the dockerfile at build:

`ENTRYPOINT ["python", "manage.py", "runserver", "0.0.0.0:8000", "arg1=1", "arg2=2"]`

Lastly every docker container needs some command to run at start, so if you have
no entrypoint specified for your image using the `--entrypoint` option then you
must pass a command to the `docker run` command or the container won't run:

`docker run -d image_name /bin/bash`

If you _do_ have an entrypoint specified in your image, and you pass a command
as shown above, the text of the command will get appended to the list of args
to the entrypoint as described above, so it is important to understand the
differences when you do or don't have an entrypoint defined.

### Ports

A docker container is connected internally to a virtual ethernet interface. It
has its own IP and port ranges separate from the host system. This means that
if you start a process inside a container and it binds to 0.0.0.0:8000 then
any other process running in that same container can connect to it there, but
processes (such as a web client) that are running on the host cannot.

If your image implements a process that you need to connect to from outside
you can use the `--ports` option to venv2docker to specify which ports you
want to make available to the outside world:

`venv2docker --ports=80,443 image_name`

When this option is used the following is generated into the dockerfile at build
time:

`EXPOSE 80 443`

However this is not enough to get you connected. While this does document which
port the container expects to receive traffic on, it does not tell the system
which port on the host to connect it to. That is done in the `docker run` command:

`docker run -d -p 80:80 -p 443:443 image_name`

The ports on the host and container don't need to match, giving you a lot of flexibility
in how you connect to your service from outside.

### Running the image

Once your image is successfully built you can use the `docker run` command to start
and test it. Exactly how you use this command depends on the options you used when
building the image, and what your application does but here are a few general
examples:

1. Run an image in the foreground and get a shell inside it:

    `docker run -it image_name /bin/bash`

2. Run an image in the background and connect a host port to a port in the container:

    `docker run -d -p 8000:8000 image_name`

3. Run an image, setting an environment variable inside it:

    `docker run -d -e "MYVAR=MYVALUE" image_name`

If you run your image in the foreground with a shell then killing it is as easy as
exiting the shell by typing `exit`. For images running detached in the background
you can either do `docker rm <container_id>` or `docker rm <container_name>` if you named
the container using the `--name` flag to the `docker run` command. If you don't know
the container id you can find it with `docker ps.` Lastly, containers that have been
stopped or in which PID 1 has exited remain in their created state, and an attempt to
remove and rebuild the image will fail since the container depends on it, as will
attempts to create the container again with the same name. To find stopped/exited
containers use `docker ps -a`.

More information can be found in [the docker run reference](https://docs.docker.com/engine/reference/run/).

### Debugging techniques

A docker container, especially one running in detached mode in the background, can
seem like an inscrutable black box when something goes wrong. Why didn't it start?
Why can't I connect to my service? Fortunately there are a couple of tools and
techniques you can use to see what is going on.

If your process writes diagnostic information to stdout or stderr you can retrieve
it, as long as the container exists, using the [docker logs](https://docs.docker.com/engine/reference/commandline/logs/)
command. You can use the `--follow` option to this command to get streaming output from
both pipes.

You can get information about the current status of the container using the [docker inspect]
(https://docs.docker.com/engine/reference/commandline/inspect/) command.

You can execute a command inside a container that is in the running state using the
[docker exec](https://docs.docker.com/engine/reference/commandline/exec/) command,
which is a very powerful tool.

For example, to get a shell inside a running container:

`docker exec -it <container_name_or_id> /bin/bash`

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

`venv2docker --base=ubuntu:15.10 --name=my_repo/my_test_image --tag=beta my_test_env`

The default base image for builds is debian jessie, however you can use the
`-b/--base` option to set any base image you want to use by name.

`venv2docker --base=ubuntu:15.10 --name=my_repo/my_test_image --tag=beta --apt=libxml2,libxslt1-dev my_test_env`

The python dependencies captured by a virtualenv don't always record all the dependencies
of a project. If you need to install packages with apt you can do so using the `--apt`
option as shown above. Packages are installed after `apt-get update -y` is run, and
before any additional pip dependencies are installed.

### Options

The following command line parameters control various aspects of the final image
produced by venv2docker.

----

###### --apt=PACKAGE[,PACKAGE][...]
###### --apt=FILE

Example:

`venv2docker --apt=libxml2,libxslt1-dev my_test_env`
`venv2docker --apt=path/to/file.txt my_test_env`

Use to install extra dependencies at build time. Takes either a comma-delimited
list of package names, or the path to a text file with one package per line.

The script installs apt packages by running `apt-get update` and then
`apt-get install`. Packages are installed as the first step in the image build
process.

----

###### --args=ARG[,ARG][...]

Example:

`venv2docker --entrypoint=python --args=manage.py,runserver my_test_env`

Use to pass arguments to an entrypoint command. Takes a comma-delimited list
of argument strings, each of which is appended as-is to the arguments list
of the ENTRYPOINT directive in the generated dockerfile.

----

###### -b|--base=IMAGE

Example:

`venv2docker --base=ubuntu:15.10 my_test_env`

Docker images that you create are layered on top of an existing, or "base"
image that contains the operating system dependencies. By default venv2docker
will use the debian:jessie base image from the Docker hub, and will automatically
install python into this image at build time if the user hasn't specified it on
the command line using `--apt`. Use this argument to override the default and use
a different base image.

----

###### --bin-path=PATH

Example:

`venv2docker --base=ubuntu:15.10 --bin-path=/etc/stuff my_test_env`

The venv2docker script installs your python code into /usr/local/bin by default.
Use this argument to override the default behavior and install the project to
a different path in the image.

----

###### -c|--clean

Example:

`venv2docker --clean`

Overrides all other options and causes the venv2docker script to remove the content
of the build folder, and the folder itself if possible, and then exit. If any content
has been copied into that folder it will not be removed, and the folder will be left
on disk.

----

###### -d|--debug

Example:

`venv2docker --debug`

Prints the values of some internal variables after the image build completes.

----

###### --dockerdir=PATH

Example:

`venv2docker --dockerdir=.docker my_test_env`

By default venv2docker places the generated dockerfile, the archived project and
system dependencies, and the build log into a folder it creates under your project
root named `.venv2docker`. Use this option to override the default and specify a
different folder name.

----

###### --entrypoint=COMMAND

Example:

`venv2docker --entrypoint=python my_test_env`

Use this argument to specify the command to serve as the entrypoint for the image
at startup. For more information see [Entrypoints and passing arguments](#entrypoints-and-passing-arguments).

----

###### -e|--env='VAR=VALUE[,VAR=VALUE][...]'

Example:

`venv2docker --env='PORT=80,IF=0.0.0.0' my_test_env`

Use this argument to add environment variables to the image at build time (as opposed to setting
them at runtime with the `docker run` command.)

----

###### --lib-path=PATH'

Example:

`venv2docker --lib-path=/etc/stuff my_test_env`

By default venv2docker installs the virtualenv binaries and library dependencies into
/usr/local/lib. Use this argument to override the default and specify a different install path.

----

###### --maintainer=NAME'

Example:

`venv2docker --maintainer='Mark Betz <betz.mark@no_way.com' my_test_env`

Use this argument to set the maintainer string inside the docker image.

----

###### -n/--name=NAME'

Example:

`venv2docker --name=myrepo/myimage my_test_env`

Use this argument to set the name you want to assign to the built docker image.
The name must include a repository if you want to push it to the docker hub using
either the `-p/--push` option or by manual command. If a name is not supplied
then venv2docker will use the virtualenv name.

----

###### --no-dangling

Example:

`venv2docker --no-dangling my_test_env`

Use this argument to cause venv2docker to remove the existing docker image before
building the new one. The default behavior is to leave the previous image in place,
which will result in it being untagged and accessible only by ID. This behavior
can be valuable to "roll back" to a previous version, but if you don't want to
have to occasionally clean out untagged images use this option. To be removed the
image repository, name, and tag must match those of the new image being built.

----

###### --no-log

Example:

`venv2docker --no-log my_test_env`

Ordinarily venv2docker logs the output of the `docker build` command to a file
named `build.log` in the build directory (.venv2docker by default). Use this
option to skip generating the log file.

----

###### --no-project-path

Example:

`venv2docker --no-project-path my_test_env`

By default venv2docker will add the root folder of your project to the system
PATH variable inside the image. Using the default settings this folder will be
/usr/local/bin/my_project. Use this option to prevent adding the folder to the
system path.

----

###### --no-project-pypath

Example:

`venv2docker --no-project-pypath my_test_env`

By default venv2docker will add the root folder of your project to the PYTHONPATH
variable inside the image. Using the default settings this folder will be
/usr/local/bin/my_project. Use this option to prevent adding the folder to the
python path.

----

###### --paths=PATH[,PATH][...]

Example:

`venv2docker --paths=/etc/stuff,/etc/mystuff my_test_env`

Use this option to specify additional paths to be added to the system path when the
image is built. Paths must be absolute and separated by commas.

----

###### --pip=FILE

Example:

`venv2docker --pip=/etc/stuff/requirements.txt my_test_env`

Use this option to specify a path to a file containing additional pip requirements to
be installed in the image. The file will be copied into the venv2docker build
directory, renamed to requirements.txt, added into the image and installed during
the next build step.

----

###### --ports=PORT[,PORT][...]

Example:

`venv2docker --ports=80,443 my_test_env`

Use this option to specify ports to be exposed on the container at runtime. For
more information see [Ports](#ports).

----

###### -p|--push

Example:

`venv2docker --push my_test_env`

Use this argument to instruct venv2docker to push your image to a registry after a
successful build. For this option to succeed the image name must include a repository
name/url.

----

###### --pypaths=PATH[,PATH][...]

Example:

`venv2docker --pypaths=/etc/stuff,/etc/mystuff my_test_env`

Use this argument to specify additional paths to be added to the python path inside
the image. Paths must be absolute and separated by commas.

----

###### -r|--run

Example:

`venv2docker --run my_test_env`

NOT IMPLEMENTED AT THIS TIME.

----

###### -s|--skip-image

Example:

`venv2docker --skip-image my_test_env`

Use this option to cause venv2docker to assemble the necessary files and create
the dockerfile, but skip building the actual image.

----

###### -t|--tag=TAG

Example:

`venv2docker --name=myrepo/myimage --tag=dev my_test_env`

Docker uses a tag system for versioning images of the same name within a single
repository. By default the tag "latest" is assigned to all images that do not
specify a specific tag value. Use this argument to override the default tag
and set your own. Note that only the "latest" tag is implicit in docker commands.
All other tags must be explicitly specified when identifying images.

----

###### -u|--user=USER

Example:

`venv2docker --user=my_user my_test_env`

By default all docker commands run inside an image run as the root user, and
all processes started in the container at runtime execute as the root user.
The docker build command offers the USER directive which causes the specified
user to become the active user for all commands subsequent to the directive
in the dockerfile. Aso the last USER specified will be the user for processes
launched then when the container is started. This option offers limited
support for switching the user. It is limited in that it happens once, right
before the ENTRYPOINT directive. So it is not possible to switch users, run
some command at build time, and then switch back.

----

###### -w|--workdir=PATH

Example:

`venv2docker --workdir=/etc/mystuff/bin my_test_env`

The default working directory for processes created inside a docker container
at runtime is the root folder '/'. The default behavior of venv2docker is to
set the working directory to the root folder of your project using a WORKDIR
directive in the dockerfile. Use this argument to specify a different working
directory.

----

###### -y|--no-prompt

Example:

`venv2docker --no-prompt my_test_env`

Suppresses the confirmation prompt normally displayed before the image builds.

### Tutorial: a quickie django image

A quick walk-through of the whole process for a quick-start django image. Hardly
deserves to be called a tutorial, but really there isn't all that much to it.

##### Step 1: create a virtualenv for the project

```bash
mark@viking:~/workspace$ mkproject django_v2d
New python executable in django-v2d/bin/python
Installing setuptools, pip, wheel...done.
Creating /home/mark/workspace/django_v2d
Setting project for django_v2d to /home/mark/workspace/django_v2d
(django_v2d)mark@viking:~/workspace/django_v2d$
```

##### Step 2: install django

```bash
(django_v2d)mark@viking:~/workspace/django_v2d$ pip install Django
Collecting Django
  Using cached Django-1.9.4-py2.py3-none-any.whl
Installing collected packages: Django
Successfully installed Django-1.9.4
(django_v2d)mark@viking:~/workspace/django_v2d$ python -c "import django; print(django.get_version())"
1.9.4
(django_v2d)mark@viking:~/workspace/django_v2d$
```

##### Step 3: start the django project

Django wants to create the project folder and will complain if it exists, so at
this point back up a folder and remove the one that virtualenvwrapper created.
Then run the `startproject` command.

```bash
(django_v2d)mark@viking:~/workspace/django_v2d$ cd ..
(django_v2d)mark@viking:~/workspace$ rm -rf django_v2d
(django_v2d)mark@viking:~/workspace$ django-admin startproject django_v2d
(django_v2d)mark@viking:~/workspace$ cd django_v2d
(django_v2d)mark@viking:~/workspace/django_v2d$ ll
total 24
drwxrwxr-x 3 mark mark 4096 Mar  9 23:54 ./
drwxrwxr-x 9 mark mark 4096 Mar  9 23:54 ../
drwxrwxr-x 2 mark mark 4096 Mar  9 23:54 django_v2d/
-rwxrwxr-x 1 mark mark  253 Mar  9 23:54 manage.py*
(django_v2d)mark@viking:~/workspace/django_v2d$
```

##### Step 5: run migrations

You might not need to do this, but it removes a warning on the server startup.

```bash
(django_v2d)mark@viking:~/workspace/django_v2d$ python manage.py migrate
Operations to perform:
  Apply all migrations: admin, contenttypes, auth, sessions
Running migrations:
  Rendering model states... DONE
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying sessions.0001_initial... OK
(django_v2d)mark@viking:~/workspace/django_v2d$
```

##### Step 6: test the server

```bash
(django_v2d)mark@viking:~/workspace/django_v2d$ python manage.py runserver
Performing system checks...

System check identified no issues (0 silenced).
March 10, 2016 - 05:00:21
Django version 1.9.4, using settings 'django_v2d.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

```bash
mark@viking:~/workspace/django_v2d$ curl localhost:8000
```
```html
<!DOCTYPE html>
<!-- ... -->
<body>
<div id="summary">
  <h1>It worked!</h1>
  <h2>Congratulations on your first Django-powered page.</h2>
</div>

<div id="instructions">
  <p>
    Of course, you haven't actually done any work yet. Next, start your first app by running <code>python manage.py startapp [app_label]</code>.
  </p>
</div>

<div id="explanation">
  <p>
    You're seeing this message because you have <code>DEBUG = True</code> in your Django settings file and you haven't configured any URLs. Get to work!
  </p>
</div>
</body></html>
```
```bash
mark@viking:~/workspace/django_v2d$
```

##### Step 7: build the docker image

```bash
mark@viking:~/venv2docker/bin$ ./venv2docker --entrypoint=python --args=manage.py,runserver,0.0.0.0:8000 --name=django_v2d --ports=8000 --no-dangling django_v2d
```

Note that an argument is added to tell the django development server to bind to 0.0.0.0 so
that we can connect from the host system. Without this it will bind to 127.0.0.1 and won't
be able to accept traffic across the bridge from the host.

After the build completes the image has been added to the local docker repository, and the
project directory now contains the build directory as shown:

```bash
mark@viking:~/workspace/django_v2d/.venv2docker$ ll
total 7140
drwxrwxr-x 2 mark mark    4096 Mar 10 00:07 ./
drwxrwxr-x 4 mark mark    4096 Mar 10 00:07 ../
-rw-rw-r-- 1 mark mark   10976 Mar 10 00:07 build.log
-rw-rw-r-- 1 mark mark    5109 Mar 10 00:07 django_v2d.tar.gz
-rw-rw-r-- 1 mark mark    1355 Mar 10 00:07 Dockerfile
-rw-rw-r-- 1 mark mark 1588037 Mar 10 00:07 venv_bin.tar.gz
-rw-rw-r-- 1 mark mark     165 Mar 10 00:07 venv_include.tar.gz
-rw-rw-r-- 1 mark mark 5620772 Mar 10 00:07 venv_lib.tar.gz
-rw-rw-r-- 1 mark mark     148 Mar 10 00:07 venv_local_lib.tar.gz
```

It's worth looking at the generated dockerfile to get a sense of what it is
doing and in what order:

```
# venv2docker generated dockerfile
#
# generated on: Thu Mar 10 00:07:08 EST 2016
#
# project path: /home/mark/workspace/django_v2d
# venv path: django_v2d
# image name: django_v2d
# script version: 0.0.1
#
# This file will be recreated if venv2docker is run again, so be
# sure to preserve any edits separately.

FROM debian:jessie

# apt dependencies
RUN apt-get update -y
RUN apt-get install -y python

# python project and dependencies
COPY django_v2d.tar.gz /usr/local/bin/
RUN cd /usr/local/bin/ && tar xfz django_v2d.tar.gz && rm django_v2d.tar.gz
COPY venv_bin.tar.gz /usr/local/lib/
RUN cd /usr/local/lib/ && tar xfz venv_bin.tar.gz && rm venv_bin.tar.gz
COPY venv_include.tar.gz /usr/local/lib/
RUN cd /usr/local/lib/ && tar xfz venv_include.tar.gz && rm venv_include.tar.gz
COPY venv_lib.tar.gz /usr/local/lib/
RUN cd /usr/local/lib/ && tar xfz venv_lib.tar.gz && rm venv_lib.tar.gz
COPY venv_local_lib.tar.gz /usr/local/lib/
RUN cd /usr/local/lib/ && tar xfz venv_local_lib.tar.gz && rm venv_local_lib.tar.gz

# exposed ports
EXPOSE 8000

# PATH and PYTHONPATH
ENV PATH /usr/local/bin/django_v2d:/usr/local/lib/django_v2d/bin:$PATH
ENV PYTHONPATH /usr/local/bin/django_v2d:$PYTHONPATH

# set working directory
WORKDIR /usr/local/bin/django_v2d

# entrypoint and arguments
ENTRYPOINT ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

##### Step 8: run and test the image

```bash
mark@viking:~/workspace/venv2docker/bin$ docker run -d -p 8000:8000 django_v2d
473f81f4fbe8887ad56a4f61ed293807aa92fa9a3e7376793e87b1a681af3b68
mark@viking:~/workspace/venv2docker/bin$
```

```bash
mark@viking:~/workspace/venv2docker/bin$ curl localhost:8000
```
```html
<!DOCTYPE html>
<!-- ... -->
<body>
<div id="summary">
  <h1>It worked!</h1>
  <h2>Congratulations on your first Django-powered page.</h2>
</div>

<div id="instructions">
  <p>
    Of course, you haven't actually done any work yet. Next, start your first app by running <code>python manage.py startapp [app_label]</code>.
  </p>
</div>

<div id="explanation">
  <p>
    You're seeing this message because you have <code>DEBUG = True</code> in your Django settings file and you haven't configured any URLs. Get to work!
  </p>
</div>
</body></html>
```
```bash
mark@viking:~/workspace/venv2docker/bin$
```

##### Step 8: remove the container and image

```bash
mark@viking:~/workspace/venv2docker/bin$ docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
473f81f4fbe8        django_v2d          "python manage.py run"   2 minutes ago       Up 2 minutes        0.0.0.0:8000->8000/tcp   gloomy_mestorf
mark@viking:~/workspace/venv2docker/bin$ docker stop 473f81f4fbe8
473f81f4fbe8
mark@viking:~/workspace/venv2docker/bin$ docker rm 473f81f4fbe8
473f81f4fbe8
mark@viking:~/workspace/venv2docker/bin$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
django_v2d          latest              1ad9d3bd337a        10 minutes ago      200.1 MB
debian              jessie              f50f9524513f        8 days ago          125.1 MB
mark@viking:~/workspace/venv2docker/bin$ docker rmi django_v2d
Untagged: django_v2d:latest
Deleted: sha256:1ad9d3bd337a115bf86427b00dfa7153a7856d5c535a189009c8d3c2af0bfcf4
...
mark@viking:~/workspace/venv2docker/bin$
```
