# Overview

This image contains [IBM MQ Advanced for Developers](http://www-03.ibm.com/software/products/en/ibm-mq-advanced-for-developers).  The Dockerfile for this image can be found on the [ibm-messaging GitHub](http://github.com/ibm-messaging/mq-docker).  This is currently a **technical preview** and IBM would [welcome feedback](#issues-and-contributions).

# Preparing your Docker host
It is necessary to [configure operating settings on your Docker host](http://www-01.ibm.com/support/knowledgecenter/SSFKSJ_8.0.0/com.ibm.mq.ins.doc/q008550_.htm?lang=en), to allow WebSphere MQ access to the resources it needs.  This covers things like maximum file handles, which are governed by the Docker host, and not containers.  If the host is configured incorrectly, the container will terminate with a message about what has failed.

You need to make sure that you either have a Linux kernel version of V3.16, or else you need to add the [`--ipc host`](http://docs.docker.com/reference/run/#ipc-settings) option when you run an MQ container.  The reason for this is that IBM MQ uses shared memory, and on Linux kernels prior to V3.16, containers are usually limited to 32 MB of shared memory.  In a [change](https://git.kernel.org/cgit/linux/kernel/git/mhocko/mm.git/commit/include/uapi/linux/shm.h?id=060028bac94bf60a65415d1d55a359c3a17d5c31
) to Linux kernel V3.16, the hard-coded limit is greatly increased.  This kernel version is available in Ubuntu 14.04.2 onwards, Fedora V20 onwards, and boot2docker V1.2 onwards.  If you are using a host with an older kernel version, but Docker version 1.4 or newer, then you can still run MQ, but you have to give it access to the host's IPC namespace using the [`--ipc host`](http://docs.docker.com/reference/run/#ipc-settings) option on `docker run`.  Note that this reduces the security isolation of your container.  Using the host's IPC namespace is a temporary workaround, and you should not attempt shared-memory connections to queue managers from outside the container.

# Usage

In order to use the image, it is necessary to accept the terms of the IBM MQ for Developers license.  This is achieved by specifying the environment variable `LICENSE` equal to `accept` when running the image.  You can also view the license terms by setting this variable to `view`. Failure to set the variable will result in the termination of the container with a usage statement.  You can view the license in a different language by also setting the `LANG` environment variable.

This image is primarily intended to be used as an example base image for your own MQ images.

The docker image exposes:

1. Port 1414 - MQ Listener
2. Port 1883 - MQTT Listener
3. Volume '/var/mqm' to persist MQ data and configuration
4. Volume '/etc/mqm' to provide custom MQSC scripts 


## Running with the default configuration

You can run a queue manager with the default configuration and a listener on port 1414 using the following command.  Note that the default configuration is locked-down from a security perspective, so you will need to customize the configuration in order to effectively use the queue manager.  For example, the following command creates and starts a queue manager called `QM1`, and maps port 1414 on the host to the MQ listener on port 1414 inside the container:

~~~
sudo docker run \
  --env LICENSE=accept \
  --env MQ_QMGR_NAME=QM1 \
  --volume /var/example:/var/mqm \
  --publish 1414:1414 \
  --detach \
  ibmcom/mqadvanced:mqv8
~~~

Note that the filesystem for the mounted volume directory (`/var/example` in the above example) must be [supported](http://www-01.ibm.com/support/knowledgecenter/SSFKSJ_8.0.0/com.ibm.mq.pla.doc/q005820_.htm?lang=en).

## Customizing the queue manager configuration

You can customize the configuration in several ways:

1. By creating your own image and adding your an MQSC file called `/etc/mqm/config.mqsc`.  This file will be run when your queue manager is created.
2. By using [remote MQ administration](http://www-01.ibm.com/support/knowledgecenter/SSFKSJ_8.0.0/com.ibm.mq.adm.doc/q021090_.htm).  Note that this will require additional configuration as remote administration is not enabled by default.
3. By linking an additional volume (e.g. `/var/mqm/scripts`) that contain a custom `config.mqsc` file to exposed script path (`/etc/mqm/`) of the docker image.

Note that a listener is always created on port 1414 inside the container.  This port can be mapped to any port on the Docker host.

The following is an *example* `Dockerfile` for creating your own pre-configured image, which adds a custom `config.mqsc` and an administrative user `alice`.  Note that it is not normally recommended to include passwords in this way:

~~~
FROM ibmcom/mqadvanced
RUN useradd alice -G mqm && \
    echo alice:passw0rd | chpasswd
COPY config.mqsc /etc/mqm/
~~~

Here is an example corresponding `config.mqsc` script from the [mqdev blog](https://www.ibm.com/developerworks/community/blogs/messaging/entry/getting_going_without_turning_off_mq_security?lang=en), which allows users with passwords to connect on the `PASSWORD.SVRCONN` channel:

~~~
DEFINE CHANNEL(PASSWORD.SVRCONN) CHLTYPE(SVRCONN)
SET CHLAUTH(PASSWORD.SVRCONN) TYPE(BLOCKUSER) USERLIST('nobody') DESCR('Allow privileged users on this channel')
SET CHLAUTH('*') TYPE(ADDRESSMAP) ADDRESS('*') USERSRC(NOACCESS) DESCR('BackStop rule')
SET CHLAUTH(PASSWORD.SVRCONN) TYPE(ADDRESSMAP) ADDRESS('*') USERSRC(CHANNEL) CHCKCLNT(REQUIRED)
ALTER AUTHINFO(SYSTEM.DEFAULT.AUTHINFO.IDPWOS) AUTHTYPE(IDPWOS) ADOPTCTX(YES)
REFRESH SECURITY TYPE(CONNAUTH)
~~~

## Running MQ commands

It is recommended that you configure MQ in your own custom image.  However, you may need to run MQ commands directly inside the process space of the container.  To run a command against a running queue manager, you can use `docker exec`.  If you run commands non-interactively under Bash, then the MQ environment will be configured correctly for you.  For example:

~~~
sudo docker exec \
  --tty \
  --interactive \
  ${CONTAINER_ID} \
  bash -c dspmq
~~~

Using this technique, you can have full control over all aspects of the MQ installation.  Note that if you use this technique to make changes to the filesystem, then those changes would be lost if you re-created your container unless you make those changes in volumes.


## Installed components

This image includes the core MQ server, language packs, and GSKit.  If you want to install other features, such as Managed File Transfer, the Telemetry Service, or Advanced Message Security, then you will need to re-build your own Docker image, using the example provided on the [ibm-messaging GitHub](http://github.com/ibm-messaging/mq-docker).  See the [MQ documentation](http://www-01.ibm.com/support/knowledgecenter/SSFKSJ_8.0.0/com.ibm.mq.ins.doc/q008350_.htm?lang=en) for details of which RPMs to choose.

# Issues and contributions

For issues relating specifically to this Docker image, please use the [GitHub issue tracker](https://github.com/ibm-messaging/mq-docker/issues). For more general issues relating to IBM MQ or to discuss the Docker technical preview, please use the [messaging community](https://developer.ibm.com/answers/?community=messaging). If you do submit a Pull Request related to this Docker image, please indicate in the Pull Request that you accept and agree to be bound by the terms of the [IBM Contributor License Agreement](CLA.md).

# License

The Dockerfile and associated scripts are licensed under the [Eclipse Public License 1.0](./LICENSE). IBM MQ Advanced for Developers is licensed under the IBM International License Agreement for Non-Warranted Programs. This license may be viewed from the image using the `LICENSE=view` environment variable as described above or may be found [online](http://www14.software.ibm.com/cgi-bin/weblap/lap.pl?li_formnum=L-APIG-9BUHAE). Note that this license does not permit further distribution.
