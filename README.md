# Anchore Troubleshooting Guide

This guide will walkthrough some general troubleshooting tips with your Anchore Engine instance.

## Anchore CLI

The Anchore CLI provides a command line interface on top of the Anchore Engine installation. Anchore CLI is published as a Python package that can be installed from source from the Python PyPi package repository on any platform supporting PyPi. For more information on installing the CLI, check out the [GitHub Repository](https://github.com/anchore/anchore-cli). There is also a Anchore Engine CLI container image available on [Docker Hub](https://hub.docker.com/r/anchore/engine-cli/). Finally, the Anchore CLI comes packaged inside the Anchore Engine container, and can be accessed by executing into the running Anchore Engine container. 

### Troubleshooting the CLI

By default the Anchore CLI will try to connect to the Anchore Engine at `http://localhost:8228/v1` with no authentication. The username, password and URL for the server can be passed to the Anchore CLI as command line arguments. 

```
--u   TEXT   Username     eg. admin (default)
--p   TEXT   Password     eg. foobar (default)
--url TEXT   Service URL  eg. http://localhost:8228/v1
```

Rather than passing these parameters for every call to the cli they can be stores as environment variables.

```
ANCHORE_CLI_URL=http://myserver.example.com:8228/v1
ANCHORE_CLI_USER=admin
ANCHORE_CLI_PASS=foobar
```

If you run into an `"Unauthorized"` error, verify you have configured the Anchore CLI correctly, as this error is most commonly seen when the Username, Password, or Service URL are improperly set. 

**Note:** When passing the parameters through the command line, order matters. For example, `anchore-cli --url http://localhost:8228/v1 --u admin --p foobar system status`

## Anchore Engine

 Anchore Engine is built and delivered as a [Docker container](https://hub.docker.com/r/anchore/anchore-engine). The Anchore Engine is a collection of services that can be deployed co-located or fulkly distributed or anythin in-between, as such it can scale out to increase analysis throughput. The only external system required is PostgreSQL (9.6+) that all services connect to, but do not use for communication beyond some very simple service registration / lookup processes. 

Throughout this guide, I will be executing Anchore CLI commands to assist with troubleshooting. For more information on the Anchore CLI, please reference the CLI section above. 

### General Troubleshooting Approach

Verify that the following Anchore Engine services are up

- Service analyzer
- Service policy_engine
- Service catalog
- Service apiext
- Service simplequeue

You can do so by running: `anchore-cli system status`

```
anchore-cli system status
Service analyzer (dockerhostid-anchore-engine, http://anchore-engine:8084): up
Service policy_engine (dockerhostid-anchore-engine, http://anchore-engine:8087): up
Service catalog (dockerhostid-anchore-engine, http://anchore-engine:8082): up
Service apiext (dockerhostid-anchore-engine, http://anchore-engine:8228): up
Service simplequeue (dockerhostid-anchore-engine, http://anchore-engine:8083): up

Engine DB Version: 0.0.9
Engine Code Version: 0.3.4
```

**Note:** If specific services are down, you can investigate the logs for the services. The logs can be accessed by executing into the corresponding container and navigating to `/var/log/anchore`. From this location you can access logs for specific services (if co-located each of the service logs will be available for access under this directory).

```
# Example logs
# Co-located Anchore Engine installation 

root@4c0a95557659:/var/log/anchore# ls
anchore-api.log                 anchore-simplequeue.log     anchore-catalog.log             anchore-policy-engine.log  anchore-worker.log
```

### Events

If you've successfully verified that all of the Anchore Engine sevices are up, but are still running into issues operating Anchore a good place check is the event log. 

The event log subsystem provides users with a mechanism to inspect asynchronous event occuring across various Anchore Engine services. Anchore events include periodically triggered activities such as vulnerability data feed sync in the policy_engine service, image analysis failures originating from the analyzer service, and other informational or system fault events. The catalog service may also generate events for any repositories or image tags that are being watched, when Anchore Engine encounters connectivity, authentication, authorization or other errors in the process of checking for updates. 

The event log is aimed at troubleshooting most common failure scenarios (especially those that happen during asynchronous engine operations) and to pinpoint the reasons for failures, that can be used subsequently to help with corrective actions. Events can be cleared from Anchore Engine in bulk or individually. 

#### Viewing Events

