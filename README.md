# docker-gc

## Introduction
docker-gc is a microservice to cleanup docker images automatically based on recycling strategy. Dockerfile and helm charts are provided for easy installation.

## Why do I need docker-gc?
When you have a cluster running docker containers, whenever your docker container uses host docker by exposing docker socket, it leaves marks and you need to cleanup images produced by those containers eventually. Although Kubernetes comes with image and container GC, it only deals with the images/containers deployed by k8s but not the ones that get used/generated by containers that talks to host docker. Things will get worse especially when you have docker build agents running in the clusters that uses heavily on host docker. In this scenario, a GC service is needed, not only to cleanup unused images but also to keep as much images/layers as we can for caching purposes. If this makes sense to you, you will simply need it.

## How it works?
docker-gc runs every 60 minutes (configurable) to analysis host docker registry by generating full dependency tree and start deleting from leaf layers/images. If the image is old enough and never used by any containers, it will be removed. If the image is old enough but recently used by any container or used by active container, it will be preserved. There are two cleanup strategies available and everything is configurable. 

## Installation

Run docker-gc container standalone using default configs:
```
$ docker run -d -v /var/run/docker.sock:/var/run/docker.sock jackil/docker-gc
```

## Recycling Strategy
DockerGC will cleanup images older than pre-configured interval if no running containers were found or no containers were found in pre-configed blacklist states. Images used by containers in blacklist states will be removed, containers in blacklist states will also be stopped and removed as well.
    
#### 1. ByDate
When recycling strategy set to [ByDate], DockerGC will cleanup images older than pre-configured interval.
    
#### 2. ByDiskSpace
When recycling strategy set to [ByDiskSpace], DockerGC will cleanup images when disk usage of all docker images is over threshold.


### Example of Configurations [Environment variables]

    -DOCKERGC_DOCKER_ENDPOINT unix:///var/run/docker.sock
    -DOCKERGC_EXECUTION_INTERVAL_IN_MINUTES 60                   // docker-gc runs every 60 minutes

    -DOCKERGC_CONTAINER_STATE_BLACKLIST dead,exited              // docker-gc will not going to delete if
    -DOCKERGC_WAIT_FOR_CONTAINERS_IN_BLACKLIST_STATE_IN_DAYS 7   // container dead or exited within 7 days
                                                                 // you can also add more states based on your need
    -DOCKERGC_RECYCLING_STRATEGY ByDate
    -DOCKERGC_DAYS_BEFORE_DELETION 30
     OR
    -DOCKERGC_RECYCLING_STRATEGY ByDiskSpace
    -DOCKERGC_SIZE_LIMIT_IN_GIGABYTE 100

    -STATSD_LOGGER_ENABLED false                                 // statsd logger
    -STATSD_HOST Statsd
    -STATSD_PORT 8125

Note: When DOCKERGC_EXECUTION_INTERVAL_IN_MINUTES set to -1, docker-gc will perform one-time cleanup (Exited after finished)
