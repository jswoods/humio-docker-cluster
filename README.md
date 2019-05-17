# Humio Docker Cluster

This project contains all you need to get a local cluster of Humio running
locally.

You can find more information on installing Humio yourself outside of this
project in the Humio documentation: https://docs.humio.com/installation/

## Prerequisites

To proceed, you need to make sure you have Docker installed. Notably, you
need to make sure the `docker` and `docker-compose` commands are available.
If they aren't, then you should follow the below steps to install Docker.
This project assumes you're running on a system with a bash-compatible shell.

### Installing Docker

You can install Docker on a Mac either via [Homebrew](https://brew.sh/) or
via Docker's installer. If you're using Homebrew run:

```
brew cask install docker
```

Otherwise, download Docker's installer ([Mac](https://docs.docker.com/docker-for-mac/install/)).

If you're on an Ubuntu box, then you can follow [these instructions](https://www.ostechnix.com/install-docker-ubuntu/).

### Determining Network Settings

Inside the `docker-compose.yml` file, you'll find a `networks` section at the end.
This defines the subnet being used by the docker cluster. By default, we're using
`172.20.0.0/16`, but you can change this to whatever you like. If you do change it,
you'll need to update the `ipv4_address` field for each container and the
`extra_hosts` field that populates `/etc/hosts` on the containers to facilitate
communication between them.

### Volumes

Each container mounts a variety of volumes (typically a data volume, a logs volume,
and a single config file for zookeeper and kafka). All these mounts points are stored
in the `mounts` directory under the container directory of this project. You can
peruse logs or other files there if you like. In production environments, it's
important to have the kafka volume seperate from the other services though because
under some conditions it can fill volumes relatively fast. By putting it on a separate
partition or volume, you can prevent the other services from crashing if Kafka eats
up all of its disk space.

### Humio Config Settings

Each Humio container directory has a `humio.env` file in it that contains Humio's
config settings. You can get more information about potential configuration options
[here](https://docs.humio.com/configuration/). One important thing to note is that
these settings are only read at **container creation**. If you change them, you need
to run the following commands for those to be picked up:

```
$ docker-compose stop humio{1..3}
$ docker-compose up --no-start humio{1..3}
```

followed by starting them one by one again (as described in the following sections):

```
$ docker-compose up -d humio1
```

## Getting Started

### Create the Containers

The first step is to create all the required docker containers. To do this, run:

```
docker-compose up --no-start
```

Note that the `--no-start` is important here because we need to boot the containers
in a specific order (this could be controlled via elaborate scripts, but for the
purposes of this project we rely on doing it by hand).

### Start Zookeeper Containers

The first set of containers we want to start are the zookeepers. To do this, run:

```
docker-compose up -d zookeeper{1..3}
```

The `-d` will bring up the containers in detached mode. The `{1..3}` is bash
expansion notation and will convert to `zookeeper1 zookeeper2 zookeeper3`
behind the scenes. If you're not using bash, you can just manually type those
out.

To check on how those containers are doing, you can check their logs by
running:

```
docker-compose logs -f zookeeper{1..3}
```

If all went well, then you should see something like this:

```shell
$ docker-compose logs zookeeper{1..3} | grep ELECTION
humio-zookeeper1 | [2019-05-17 21:27:09,163] INFO FOLLOWING - LEADER ELECTION TOOK - 309 (org.apache.zookeeper.server.quorum.Learner)
humio-zookeeper3 | [2019-05-17 21:27:09,197] INFO LEADING - LEADER ELECTION TOOK - 338 (org.apache.zookeeper.server.quorum.Leader)
humio-zookeeper2 | [2019-05-17 21:27:09,185] INFO FOLLOWING - LEADER ELECTION TOOK - 273 (org.apache.zookeeper.server.quorum.Learner)
```

### Start Kafka Containers

Now that the zookeeper containers are running, we can start the Kafka containers:

```
docker-compose up -d kafka{1..3}
```

If all went well, then you should see something like this:

```
$ docker-compose logs kafka{1..3} | grep started
humio-kafka3  | [2019-05-17 21:29:31,488] INFO [KafkaServer id=3] started (kafka.server.KafkaServer)
humio-kafka1  | [2019-05-17 21:29:31,435] INFO [KafkaServer id=1] started (kafka.server.KafkaServer)
humio-kafka2  | [2019-05-17 21:29:31,786] INFO [KafkaServer id=2] started (kafka.server.KafkaServer)
```

### Start Humio Containers

Due to how Humio works, these containers need to be started one at a time:

```
docker-compose up -d humio1
```

You can watch its log output until you see something like this:

```shell
$ docker-compose logs -f humio1
Attaching to humio-core1
humio-core1   | Initializing Log4j2
humio-core1   | Initializing Log4j2 completed
humio-core1   | WARNING: sun.reflect.Reflection.getCallerClass is not supported. This will impact performance.
humio-core1   | Humio API bound to host=0.0.0.0 port=8080
humio-core1   | Humio server is now running!
```

The "Humio server is now running!" line is the indication that it's done booting.
You might see some warning lines referencing "reflective access" – it's safe to
ignore those. At this point, you should be able to browse to http://0.0.0.0:8080
and access the Humio install. Once there, click on your profile picture in the top
right of the page and choose "Administration". This will take you to the view
that shows information on the Humio cluster. You should see the node you just started
here.

Now it's time to start up the second node:

```
docker-compose up -d humio2
```

Follow the same steps above. Once the server is done booting, you should see it show
up in the Cluster Nodes interface mentioned above. It will initially show up with
a red "down" status dot, but once it queries the initial node for cluster state
information and copies the data over it needs, it should turn blue. Once it's blue, you
can start the third node:

```
docker-compose up -d humio3
```

### Rebalance Storage & Digest

Once all three are up, you might notice that only the first node is being used for
storage and digest. Ideally, this will be balanced across all three nodes. You can
do this by hand in the Cluster Nodes area by clicking the "Edit" button in the Rules
side panel, but currently it's easier to do via the API. To balance them, run these
commands:

```shell
# get the local admin token from the first humio container and save it
TOKEN=$(docker exec -it humio-core1 cat /data/humio-data/local-admin-token.txt)

# balance the storage across all three nodes so you have a replication factor of 3
# with 24 partitions
curl -v -X POST http://0.0.0.0:8080/api/v1/clusterconfig/segments/partitions/set-replication-defaults \
  -d '{ "partitionCount": 24, "replicas": 3 }' \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN"; echo

# balance the digest across all three nodes evenly
curl -v -X POST http://0.0.0.0:8080/api/v1/clusterconfig/ingestpartitions/setdefaults \
  -H "Authorization: Bearer $TOKEN"; echo
```

You can get more information about Storage Rules [here](https://docs.humio.com/administration/storage-rules/)
and more information about the Cluster Management API [here](https://docs.humio.com/api/cluster-management-api/).

## Cleaning up

Once you're done testing, you can stop the whole cluster with this command:

```
docker-compose down
```

All data will still be on-disk, but the containers will be destroyed. At this
point you could follow the process above to start over with new containers and
all your previous data would still be there.

However, if you want to start from scratch completely, you can run the `cleanup`
script in the root of this project. It will run `docker-compose down` and then
delete all files that were generated by Humio, Kafka, and Zookeeper.