# Node-RED-Docker by Björn Ludwig
[![CircleCI](https://circleci.com/gh/PTB-PSt1/node-red-docker/tree/node-red.svg?style=shield)](https://circleci.com/gh/PTB-PSt1/node-red-docker/tree/node-red)
[![Get your own version badge on microbadger.com](https://images.microbadger.com/badges/version/bludoc/node-red:latest.svg)](https://microbadger.com/images/bludoc/node-red:latest)
[![Get your own image badge on microbadger.com](https://images.microbadger.com/badges/image/bludoc/node-red:latest.svg)](https://microbadger.com/images/bludoc/node-red:latest)

This project is a fork of [node-red/node-red-docker](https://github.com/node-red/node-red-docker) and contains simply some customization and is used to get full control of the actual contents
of the repository. In addition it enables authentication via hard coded users prepared in `settings.js` and serves the application at a different
URL. The default user only has read access to the flows. All users to gain write access need to be added to `settings.js` as stated [here](https://github.com/node-red/node-red-auth-github).

This project also provides the build for the `bludoc/node-red` container on [DockerHub](https://hub.docker.com/r/bludoc/node-red/).

To run this directly in docker at it's simplest just run

        docker run -it -p 1880:1880 --name mynodered bludoc/node-red

Let's dissect that command...

        docker run      - run this container... and build locally if necessary first.
        -it             - attach a terminal session so we can see what is going on
        -p 1880:1880    - connect local port 1880 to the exposed internal port 1880
        --name mynodered - give this machine a friendly local name
        bludoc/node-red - the image to base it on - currently Node-RED v0.20.7


Running that command should give a terminal window with a running instance of Node-RED

        Welcome to Node-RED
        ===================
        2 Aug 12:15:19 - [info] Node-RED version: v0.20.7
        2 Aug 12:15:19 - [info] Node.js  version: v8.16.0
        .... etc

You can then browse to `http://{host-ip}:1880/node-red` to get the familiar Node-RED desktop.

The advantage of doing this is that by giving it a name we can manipulate it more easily, and by fixing the host port we know we are on familiar ground.
(Of course this does mean we can only run one instance at a time... but one step at a time folks...)

If we are happy with what we see we can detach the terminal with `Ctrl-p``Ctrl-q` - the container will keep running in the background.

To reattach to the terminal (to see logging) run:

        $ docker attach mynodered

If you need to restart the container (e.g. after a reboot or restart of the Docker daemon)

        $ docker start mynodered

and stop it again when required

        $ docker stop mynodered

_**Note** : this Dockerfile is configured to store the flows.json file and any
extra nodes you install "outside" of the container. We do this so that you may rebuild the underlying
container without permanently losing all of your customisations._

## Images

The following images are built for each Node-RED release, using a Node.js v8 base image.

- **latest** - uses [official Node.JS v8 base image](https://hub.docker.com/_/node/).

Node-RED releases are also tagged with a version label, allowing you to fix on a specific version: `latest:X.Y.Z`.

Additional images using a newer Node.js v10 base image are available with the following tags.

- **v10**
Node-RED releases are also tagged with a version label, allowing you to fix on a specific version: `latest:X.Y.Z`,

You can see a full list of the tagged releases [here](https://hub.docker.com/r/nodered/node-red-docker/tags/).

## Project Layout

This repository contains a customized Dockerfile to build the Node-RED Docker image named above.

Build the image with the following command...

        $ docker build -f latest/Dockerfile -t mynodered:<version> .

### package.json

The package.json is a metafile that downloads and installs the required version
of Node-RED and any other npms you wish to install at build time. During the
Docker build process, the dependencies are installed under `/opt/node-red`.

The main sections to modify are

    "dependencies": {
        "node-red": "0.20.x",           <-- set the version of Node-RED here
        "node-red-node-rbe": "*"        <-- add any extra npm packages here
    },

This is where you can pre-define any extra nodes you want installed every time
by default, and then

    "scripts"      : {
        "start": "node-red -v $FLOWS"
    },

This is the command that starts Node-RED when the container is run.

### startup

Node-RED is started using NPM start from this `/opt/node-red`, with the `--userDir`
parameter pointing to the `/data` directory on the container. 

The flows configuration file is set using an environment parameter (**FLOWS**),
which defaults to *'flows.json'*. This can be changed at runtime using the
following command-line flag.

        $ docker run -it -p 1880:1880 -e FLOWS=my_flows.json bludoc/node-red

Node.js runtime arguments can be passed to the container using an environment
parameter (**NODE_OPTIONS**). For example, to fix the heap size used by
the Node.js garbage collector you would use the following command.

        $ docker run -it -p 1880:1880 -e NODE_OPTIONS="--max_old_space_size=128" nodered/node-red-docker

## Adding Nodes

Installing extra Node-RED nodes into an instance running with Docker can be
achieved by manually installing those nodes into the container, using the cli or
running npm commands within a container shell, or mounting a host directory with
those nodes as a data volume.

### Node-RED Admin Tool

Using the administration tool, with port forwarding on the container to the host
system, extra nodes can be installed without leaving the host system.

        $ npm install -g node-red-admin
        $ node-red-admin install node-red-node-openwhisk

This tool assumes Node-RED is available at the following address
`http://localhost:1880`.

Refreshing the browser page should now reveal the newly added node in the palette.

### Container Shell

        $ docker exec -it mynodered /bin/bash

Will give a command line inside the container - where you can then run the npm install
command you wish - e.g.

        $ cd /data
        $ npm install node-red-node-smooth
        node-red-node-smooth@0.0.3 node_modules/node-red-node-smooth
        $ exit
        $ docker stop mynodered
        $ docker start mynodered

Refreshing the browser page should now reveal the newly added node in the palette.

### Host Directory As Volume

Running a Node-RED container with a host directory mounted as the data volume,
you can manually run `npm install` within your host directory. Files created in
the host directory will automatically appear in the container's file system.

This command mounts the host's node-red directory, containing the user's
configuration and installed nodes, as the user configuration directory inside
the container.

        $ docker run -it -p 1880:1880 -v ~/.node-red:/data --name mynodered bludoc/node-red

_**Having file permissions issues after mounting a host directory?**_
_Docker maps the internal container uuid to 1000 on the host system. If you are running as a user with a different uuid, e.g. the second user added on a system, the container user account won't be able to read files on the host system. Fix this by using the `--run $USER` flag. See [here for more details](https://github.com/node-red/node-red-docker/issues/9)._

Adding extra nodes to the container can be accomplished by
running npm install locally.

        $ cd ~/.node-red
        $ npm install node-red-node-smooth
        node-red-node-smooth@0.0.3 node_modules/node-red-node-smooth
        $ docker stop mynodered
        $ docker start mynodered

_**Note** : Modules with a native dependencies will be compiled on the host
machine's architecture. These modules will not work inside the Node-RED
container unless the architecture matches the container's base image. For native
modules, it is recommended to install using a local shell or update the
project's package.json and re-build._

### Building Custom Image

Creating a new Docker image, using this public Node-RED image as the base image,
allows you to install extra nodes during the build process.

This Dockerfile builds a custom Node-RED image with the flightaware module
installed from NPM.

```
FROM bludoc/node-red
RUN npm install node-red-contrib-flightaware
```

Alternatively, you can modify the package.json in this repository and re-build
the images from scratch. This will also allow you to modify the version of
Node-RED that is installed. See below for more details...

## Managing User Data

Once you have customised the Node-RED instance running with Docker, we need to
ensure these modifications are not lost if the container is destroyed. Managing
this user data can be handed by persisting container state into a new image or
using named data volumes to handle move this data outside the container.

### Saving Changes As Custom Image

Modifications to files within the live container, e.g. manually adding nodes or
creating flows, do not exist outside the lifetime of the container. If that
container instance is destroyed, these changes will be lost.

Docker allows you to the current state of a container to a new image. This
means you can persist your changes as a new image that can be shared on other
systems.

        $ docker commit mynodered custom-node-red-docker

If we destroy the ```mynodered``` container, the instance can be recovered by
spawning a new container using the ```custom-node-red-docker``` image.

### Using Named Data Volumes

Docker supports using [data volumes](https://docs.docker.com/engine/tutorials/dockervolumes/) to store
persistent or shared data outside the container. Files and directories within data
volumes exist outside of the lifecycle of containers, i.e. the files still exist
after removing the container.

Node-RED uses the `/data` directory to store user configuration data.

Mounting a data volume inside the container at this directory path means user
configuration data can be saved outside of the container and even shared between
container instances.

Let's create a new named data volume to persist our user data and run a new
container using this volume.

        $ docker volume create --name node_red_user_data
        $ docker volume ls
        DRIVER              VOLUME NAME
        local               node_red_user_data
        $ docker run -it -p 1880:1880 -v node_red_user_data:/data --name mynodered bludoc/node-red

Using Node-RED to create and deploy some sample flows, we can now destroy the
container and start a new instance without losing our user data.

        $ docker rm mynodered
        $ docker run -it -p 1880:1880 -v node_red_user_data:/data --name mynodered bludoc/node-red

## Updating

Updating the base container image is as simple as

        $ docker pull bludoc/node-red
        $ docker stop mynodered
        $ docker start mynodered

## Running headless

The barest minimum we need to just run Node-RED is

    $ docker run -d -p 1880 bludoc/node-red

This will create a local running instance of a machine - that will have some
docker id number and be running on a random port... to find out run

    $ docker ps -a
    CONTAINER ID        IMAGE                       COMMAND             CREATED             STATUS                     PORTS                     NAMES
    4bbeb39dc8dc        bludoc/node-red:latest   "npm start"         4 seconds ago       Up 4 seconds               0.0.0.0:49154->1880/tcp   furious_yalow
    $

You can now point a browser to the host machine on the tcp port reported back, so in the example
above browse to  `http://{host ip}:49154`

## Linking Containers

You can link containers "internally" within the docker runtime by using the --link option.

For example I have a simple MQTT broker container available as

        docker run -it --name mybroker bludoc/node-red

(no need to expose the port 1883 globally unless you want to... as we do magic below)

Then run nodered docker - but this time with a link parameter (name:alias)

        docker run -it -p 1880:1880 --name mynodered --link mybroker:broker bludoc/node-red

the magic here being the `--link` that inserts a entry into the node-red instance
hosts file called *broker* that links to the mybroker instance....  but we do
expose the 1880 port so we can use an external browser to do the node-red editing.

Then a simple flow like below should work - using the alias *broker* we just set up a second ago.

        [{"id":"190c0df7.e6f3f2","type":"mqtt-broker","broker":"broker","port":"1883","clientid":""},{"id":"37963300.c869cc","type":"mqtt in","name":"","topic":"test","broker":"190c0df7.e6f3f2","x":226,"y":244,"z":"f34f9922.0cb068","wires":[["802d92f9.7fd27"]]},{"id":"edad4162.1252c","type":"mqtt out","name":"","topic":"test","qos":"","retain":"","broker":"190c0df7.e6f3f2","x":453,"y":135,"z":"f34f9922.0cb068","wires":[]},{"id":"13d1cf31.ec2e31","type":"inject","name":"","topic":"","payload":"","payloadType":"date","repeat":"","crontab":"","once":false,"x":226,"y":157,"z":"f34f9922.0cb068","wires":[["edad4162.1252c"]]},{"id":"802d92f9.7fd27","type":"debug","name":"","active":true,"console":"false","complete":"false","x":441,"y":261,"z":"f34f9922.0cb068","wires":[]}]

This way the internal broker is not exposed outside of the docker host - of course
you may add `-p 1883:1883`  etc to the broker run command if you want to see it...


## Issues

Here is a list of common issues users have reported with possible solutions.

### User Permission Errors

If you are seeing *permission denied* errors opening files or accessing host devices, try running the container as the root user.

```
docker run -it -p 1880:1880 --name mynodered --user=root bludoc/node-red
```

References:

https://github.com/node-red/node-red-docker/issues/15

https://github.com/node-red/node-red-docker/issues/8

### Accessing Host Devices

If you want to access a device from the host inside the container, e.g. serial port, use the following command-line flag to pass access through.

```
docker run -it -p 1880:1880 --name mynodered --device=/dev/ttyACM0 bludoc/node-red
```
References:
https://github.com/node-red/node-red-docker/issues/15

### Setting Timezone

If you want to modify the default timezone, use the TZ environment variable with the [relevant timezone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).

```
docker run -it -p 1880:1880 --name mynodered -e TZ="Europe/London" bludoc/node-red
```

References:
https://groups.google.com/forum/#!topic/node-red/ieo5IVFAo2o
