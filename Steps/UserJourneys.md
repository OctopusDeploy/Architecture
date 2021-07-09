There are a set of minimum deployment scenarios that each new platform supported by Octopus should support. This document lists those journeys and provides sample applications to validate they can be deployed in Octopus.

## Sample applications

### Docker

* [Backend Docker Image](https://github.com/OctopusSamples/RandonQuotes-Backend-Go)
* [Frontend Docker Image](https://github.com/OctopusSamples/RandomQuotes-Frontend-Go)

## Frontend/backend across multiple environments

It must be possible to deploy an application comprised of a frontend and backend across multiple environments.

### Container platforms

The [octopussamples/randomquotesbackendgo](https://hub.docker.com/r/octopussamples/randomquotesbackendgo) and [octopussamples/randomquotesfrontendgo](https://hub.docker.com/r/octopussamples/randomquotesfrontendgo) images must be able to be deployed to multiple environments.

Set the `APIENDPOINT` environment variable on the [octopussamples/randomquotesfrontendgo](https://hub.docker.com/r/octopussamples/randomquotesfrontendgo) image to point to the backend server. See the [docker-compose.yml](https://github.com/OctopusSamples/RandomQuotes-Frontend-Go/blob/main/docker-compose.yml) file for an example.

## Feature branches to development environment

Octopus should be able to deploy a feature branch side by side with the mainline deployment in a development environment.

### Container platforms

The frontend sample application has a number of feature branch images built. Two examples are:

* `octopussamples/randomquotesfrontendgo:0.1.9-blueheader`
* `octopussamples/randomquotesfrontendgo:0.1.11-greenheader`

## Blue/green deployment

Octopus should be able to deploy a new version of an application side by side with the old one, test it, and cut traffic over to it.

### Container platforms

The [octopussamples/randomquotesbackendgo](https://hub.docker.com/r/octopussamples/randomquotesbackendgo) image embeds a build version, which is displayed by the front end. To demonstrate a blue/green, deploy the following backend images:

1. `octopussamples/randomquotesbackendgo:0.1.26`
2. `octopussamples/randomquotesbackendgo:0.1.27`

## Canary deployment

Octopus should be able to deploy a new version of an application side by side with the old one, direct a subset of traffic to it, and progressively increase the traffic over time.

### Container platforms

The [octopussamples/randomquotesbackendgo](https://hub.docker.com/r/octopussamples/randomquotesbackendgo) image embeds a build version, which is displayed by the front end. To demonstrate a canary deployment, deploy the following backend images:

1. `octopussamples/randomquotesbackendgo:0.1.26`
2. `octopussamples/randomquotesbackendgo:0.1.27`