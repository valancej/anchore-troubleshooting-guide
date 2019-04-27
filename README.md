# Anchore Troubleshooting Guide

This guide will walkthrough some general troubleshooting tips with your Anchore Engine instance. Anchore Engine is built and delivered as a [Docker container](https://hub.docker.com/r/anchore/anchore-engine). The Anchore Engine is a collection of services that can be deployed co-located or fulkly distributed or anythin in-between, as such it can scale out to increase analysis throughput. The only external system required is PostgreSQL (9.6+) that all services connect to, but do not use for communication beyond some very simple service registration / lookup processes. 

## Anchore CLI

The Anchore CLI provides a command line interface on top of the Anchore Engine installation. Anchore CLI is published as a Python package that can be installed from source from the Python PyPi package repository on any platform supporting PyPi. For more information on installing the CLI, check out the [GitHub Repository](https://github.com/anchore/anchore-cli). There is also a Anchore Engine CLI container image available on [Docker Hub](https://hub.docker.com/r/anchore/engine-cli/). Finally, the Anchore CLI comes packaged inside the Anchore Engine container, and can be accessed by executing into the running Anchore Engine container. 
