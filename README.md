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

#### Verifying Services

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

#### The --debug and --json options

Passing the `--debug` option to any Anchore CLI can often help narrow down particular issues. 

```
# Example system status with --debug
root@4c0a95557659:/anchore-engine# anchore-cli --debug system status
INFO:anchorecli.clients.apiexternal:As Account = None
DEBUG:urllib3.connectionpool:Starting new HTTP connection (1): localhost:8228
DEBUG:urllib3.connectionpool:http://localhost:8228 "GET / HTTP/1.1" 200 5
INFO:anchorecli.clients.apiexternal:As Account = None
DEBUG:anchorecli.clients.apiexternal:GET url=http://localhost:8228/system
DEBUG:anchorecli.clients.apiexternal:GET insecure=True
DEBUG:urllib3.connectionpool:Starting new HTTP connection (1): localhost:8228
DEBUG:urllib3.connectionpool:http://localhost:8228 "GET /system HTTP/1.1" 200 2672
DEBUG:anchorecli.cli.utils:fetched httpcode from response: 200
Service analyzer (dockerhostid-anchore-engine, http://anchore-engine:8084): up
Service policy_engine (dockerhostid-anchore-engine, http://anchore-engine:8087): up
Service catalog (dockerhostid-anchore-engine, http://anchore-engine:8082): up
Service apiext (dockerhostid-anchore-engine, http://anchore-engine:8228): up
Service simplequeue (dockerhostid-anchore-engine, http://anchore-engine:8083): up
```

Passing the `--json` option to any Anchore CLI commands will output the data in JSON format.

```
# Example system status with --json
root@4c0a95557659:/anchore-engine# anchore-cli --json system status
{
    "service_states": [
        {
            "base_url": "http://anchore-engine:8087",
            "hostid": "dockerhostid-anchore-engine",
            "service_detail": {
                "available": true,
                "busy": false,
                "db_version": "0.0.9",
                "detail": {},
                "message": "all good",
                "up": true,
                "version": "0.3.4"
            },
            "servicename": "policy_engine",
            "status": true,
            "status_message": "available",
            "version": "v1"
        },
        {
            "base_url": "http://anchore-engine:8338",
            "hostid": "dockerhostid-anchore-engine",
            "service_detail": {
                "available": true,
                "busy": false,
                "db_version": "0.0.9",
                "detail": {},
                "message": "all good",
                "up": true,
                "version": "0.3.4"
            },
            "servicename": "kubernetes_webhook",
            "status": true,
            "status_message": "available",
            "version": "v1"
        },
        {
            "base_url": "http://anchore-engine:8084",
            "hostid": "dockerhostid-anchore-engine",
            "service_detail": {
                "available": true,
                "busy": false,
                "db_version": "0.0.9",
                "detail": {},
                "message": "all good",
                "up": true,
                "version": "0.3.4"
            },
            "servicename": "analyzer",
            "status": true,
            "status_message": "available",
            "version": "v1"
        },
        {
            "base_url": "http://anchore-engine:8082",
            "hostid": "dockerhostid-anchore-engine",
            "service_detail": {
                "available": true,
                "busy": false,
                "db_version": "0.0.9",
                "detail": {},
                "message": "all good",
                "up": true,
                "version": "0.3.4"
            },
            "servicename": "catalog",
            "status": true,
            "status_message": "available",
            "version": "v1"
        },
        {
            "base_url": "http://anchore-engine:8228",
            "hostid": "dockerhostid-anchore-engine",
            "service_detail": {
                "available": true,
                "busy": false,
                "db_version": "0.0.9",
                "detail": {},
                "message": "all good",
                "up": true,
                "version": "0.3.4"
            },
            "servicename": "apiext",
            "status": true,
            "status_message": "available",
            "version": "v1"
        },
        {
            "base_url": "http://anchore-engine:8083",
            "hostid": "dockerhostid-anchore-engine",
            "service_detail": {
                "available": true,
                "busy": false,
                "db_version": "0.0.9",
                "detail": {},
                "message": "all good",
                "up": true,
                "version": "0.3.4"
            },
            "servicename": "simplequeue",
            "status": true,
            "status_message": "available",
            "version": "v1"
        }
    ]
}

```

### Events

If you've successfully verified that all of the Anchore Engine sevices are up, but are still running into issues operating Anchore a good place check is the event log. 

The event log subsystem provides users with a mechanism to inspect asynchronous event occuring across various Anchore Engine services. Anchore events include periodically triggered activities such as vulnerability data feed sync in the policy_engine service, image analysis failures originating from the analyzer service, and other informational or system fault events. The catalog service may also generate events for any repositories or image tags that are being watched, when Anchore Engine encounters connectivity, authentication, authorization or other errors in the process of checking for updates. 

The event log is aimed at troubleshooting most common failure scenarios (especially those that happen during asynchronous engine operations) and to pinpoint the reasons for failures, that can be used subsequently to help with corrective actions. Events can be cleared from Anchore Engine in bulk or individually. 

#### Viewing Events

Running the following command will give a list of recent Anchore events: `anchore-cli event list`

```
# Viewing list of recent Anchore events

root@4c0a95557659:/anchore-engine# anchore-cli event list
Timestamp                          Level        Service              Host                                                           Event                                  ID                                      
2019-04-28T13:04:16.203425Z        INFO         policy_engine        dockerhostid-anchore-engine                                    feed_sync_complete                     16590324ffe443dfa8b8352a0e63bd14        
2019-04-28T13:04:04.202101Z        INFO         policy_engine        dockerhostid-anchore-engine                                    feed_sync_start                        7d0c3d0015e74302a5242f74b03092ec        
2019-04-28T07:04:03.946414Z        INFO         policy_engine        dockerhostid-anchore-engine                                    feed_sync_complete                     eb8a7f70c0eb4f92a8941ead61d3a5ef        
2019-04-28T07:03:53.112932Z        INFO         policy_engine        dockerhostid-anchore-engine                                    feed_sync_start                        1277941
```

#### Details about a specific event

If you would like more information about a specific event, you can run the following command: `anchore-cli event get <event-id>`

```
# Details about a specific Anchore event

root@4c0a95557659:/anchore-engine# anchore-cli event get 7f6dc74c8c8348ecad97f2f54ad488d6
details:
  sync_feed_types:
  - vulnerabilities
level: INFO
message: Feed sync started
resource:
  type: feeds
  user_id: admin
source:
  base_url: http://localhost:8087
  hostid: anchore-quickstart
  servicename: policy_engine
timestamp: '2019-02-06T04:15:11.372306Z'
type: feed_sync_start
```

**Note:** Depending on the output from the detailed events, looking into the logs for a particular servicename (example: policy_engine) is the next troubleshooting step.  