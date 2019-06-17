---
layout: post
title: Containerizing cranc
comments: true
---


How to run a docker container with the CLI app.
First clone the repo:

`git clone https://pagure.io/cranc.git`

`cd cranc`

Create a new dockerfile:

`vim Dockerfile`

Use for example the python:3-onbuild, as suggested in this
[guide](https://docker-curriculum.com/). It is supposed to include some
onbuild triggers, which for example COPY the `requirements.txt` file, RUN
`pip install` on it and copy the current directory into `/usr/src/app`.

`FROM python:3-onbuild`

The next thing necessary is to install the app:

`RUN python setup.py install`

And finally to run the command (as entrypoint):

`ENTRYPOINT cranc`

Time to make a build:

`docker build -t cranc .`

And run:

`docker run cranc`

Now it's possible to run cranc in a container for example like this:

`docker run cranc get issues`

Docker starts the container, runs the command and exits the container. To avoid having to
type `docker run` every time I want to run the cranc from a container, it is possible to
create a bash script for example in user bin: `/usr/bin/cranc` with:

`#!/bin/bash`

`exec docker run -rm cranc "$@"`

which means that every cranc command will add `docker run` before and append whatever comes
after ("$@").

Change it to executable:

`chmod -x ~/bin/cranc`

Don't forget to source it:

`source ~/bin/cranc`

Like this, every time I write a cranc command, docker container will be started, command will
run and container will exit. There is a way to avoid starting and exiting new container
every time.

It is also possible to write a bash script which runs a docker container infinitely and the
cranc commands can be faster, since there is no need to start a container with every command:

```bash
#!/bin/bash

docker exec cranc cranc "$@"
rc="$?"
if [ $rc -eq 1 ]; then
    docker run --entrypoint "sleep" --name cranc --detach cranc infinity
    docker exec cranc cranc "$@"
    rc="$?"
fi

exit "$rc"
```

The script executes a docker container, that will run the commands - the bash and whatever follows.
If it fails - the exit status equals to 1, that means that the container is not running ATM,
then docker container will be ran from the
sleep entrypoint (infinite sleep) and it will be detached (ran in the background).
Then the docker exec cranc command will be ran again and the exit status will be returned.
