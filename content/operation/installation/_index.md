---
title: "Installation"
---

## Overview

This section describes how to install Humio on a single machine. If you want to run a distributed cluster, please refer to [cluster setup](/operation/installation/cluster_setup/).

## Running Humio as a Docker container

Humio is distributed as a Docker image. This means that you can start an instance without a complicated installation procedure.

### Steps

1. Create an empty file on the host machine to store the Humio configuration. For example, `humio.conf`.
<br />
You can use this file to pass on JVM arguments to the Humio Java process.

1. Enter the following settings into the configuration file:

```shell
HUMIO_JVM_ARGS=-Xss2M -XX:MaxDirectMemorySize=4G
```

<!--
{{% notice note %}}
These settings are for a machine with 8GB of RAM or more.
{{% /notice %}}
-->

1. Create an empty directory on the host machine to store data for Humio:

```shell
mkdir humio-data
```

1. Pull the latest Humio image:

```shell
docker pull humio/humio
```

1. Run the Humio Docker image as a container:

```shell
docker run -v $HOST_DATA_DIR:/data --net=host --name=humio --env-file=$PATH_TO_CONFIG_FILE humio/humio
```

Replace `$HOST_DATA_DIR` with the path to the humio-data directory you created on the host machine, and `$PATH_TO_CONFIG_FILE` with the path of the configuration file you created.

1. Humio is now running. Navigate to [http://localhost:8080](http://localhost:8080) to view the Humio web interface.

{{% notice info %}}
In the above example, we started the Humio container with full access to the network of the host machine (`--net=host`). In a production environment, you should restrict this access by using a firewall, or adjusting the Docker network configuration.  
Another possibility is to forward explicit ports. That is possible like this: `-p 8080:8080`. But then you need to forward all the ports you configure Humio to use. By default Humio is only using port 8080.
{{% /notice %}}


{{% notice note %}}
On a Mac there can be issues with using the host network (`--net=host`). In that case use `-p 8080:8080` to forward port 8080 on the host network to the Docker container.  
Another concern is to allow enough memory to the virtual machine running Docker on Mac. Open the Docker app and go to preferences and specify 4GB.
{{% /notice %}}

Updating Humio is described in the [upgrade section](/operation/installation/system_administration/#upgrading)

### Running Humio as a service

The Docker container can be started as a service using Docker's [restart policies](https://docs.docker.com/engine/reference/run/#restart-policies-restart).  
An example is adding `--detach --restart=always` to the above docker run:

```shell
docker run ... --detach --restart=always
```

## Configuring Humio
Please refer to the [configuration](/operation/installation/configuration_options/) page

## System Administration
Please refer to the [system administration](/operation/installation/system_administration/) page

## Instance Sizing
Please refer to the [hardware recommendations / sizing](/operation/installation/instance_sizing/) page
