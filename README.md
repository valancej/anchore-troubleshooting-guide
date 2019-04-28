# Anchore Troubleshooting Guide

This guide will walkthrough some general troubleshooting tips with your Anchore Engine instance.

Table of contents
=================

<!--ts-->
   * [Anchore CLI](#anchore-cli)
      * [Troubleshooting the CLI](#troubleshooting-the-cli)
   * [Anchore Engine](#anchore-engine)
      * [General Troubleshooting Approach](#general-troubleshooting-approach)
      * [Events](#events)
      * [Logs](#logs)
      * [Feeds](#feeds)
<!--te-->

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

When troubleshooting Anchore Engine, the recommend approach is to first verify all Anchore services are up, use the event subsystem to narrow down particular issues, and then navigate to the logs for specific services to find out more information.

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

**Note:** If specific services are down, you can investigate the logs for the services. See [Logs Section](#logs)

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
        ...
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

### Logs

Anchore logs can be accessed by executing into the Anchore container and navigating to `/var/log/anchore`. From this location you can access logs for specific services (if co-located each of the service logs will be available for access under this directory).

```
# Example logs
# Co-located Anchore Engine installation 

root@4c0a95557659:/var/log/anchore# ls
anchore-api.log                 anchore-simplequeue.log     anchore-catalog.log             anchore-policy-engine.log  anchore-worker.log
```

### Feeds

When the Anchore Engine runs it will begin to synchronize security feed data from the Anchore feed service.

**Note:** Upon a fresh installation of Anchore Engine, the system will take some time to bootstrap. CVE data for Linux distributions such as Alpine, CentOS, Debian, Oracle, Red Hat and Ubuntu will be downloaded. The initial sync may take anywhere from  10 to 60 minutes depending on the speed of your network connection. 

#### Viewing feeds

The following command will report a list of feeds synced by Anchore: `anchore-cli system feeds list`

```
anchore-cli system feeds list
Feed                   Group                  LastSync                          RecordCount        
nvd                    nvddb:2002             2019-02-25T21:35:12.802608        6745               
nvd                    nvddb:2003             2019-02-25T21:35:13.188204        1547               
nvd                    nvddb:2004             2019-02-25T21:35:13.774093        2702               
nvd                    nvddb:2005             2019-02-25T21:35:14.281344        4749               
nvd                    nvddb:2006             2019-02-25T21:39:01.936476        7127               
nvd                    nvddb:2007             2019-02-25T21:39:02.432799        6556               
nvd                    nvddb:2008             2019-02-25T22:29:19.704624        7147               
nvd                    nvddb:2009             2019-02-25T22:29:20.292788        4964               
nvd                    nvddb:2010             2019-02-25T22:29:20.720235        5073               
nvd                    nvddb:2011             2019-02-25T21:30:43.003078        4621               
nvd                    nvddb:2012             2019-02-25T21:35:11.663650        5549               
nvd                    nvddb:2013             2019-02-25T21:39:01.289722        6160               
nvd                    nvddb:2014             2019-02-25T21:42:11.148478        8493               
nvd                    nvddb:2015             2019-02-25T21:44:55.773423        8023               
nvd                    nvddb:2016             2019-02-25T21:48:13.150698        9872               
nvd                    nvddb:2017             2019-02-25T22:03:35.550272        15162              
nvd                    nvddb:2018             2019-02-25T22:26:12.131914        13541              
nvd                    nvddb:2019             2019-02-25T22:29:19.116614        963                
vulnerabilities        alpine:3.3             2019-04-28T13:04:11.054665        457                
vulnerabilities        alpine:3.4             2019-04-28T13:04:11.283342        681                
vulnerabilities        alpine:3.5             2019-04-28T13:04:10.741848        875                
vulnerabilities        alpine:3.6             2019-04-28T13:04:13.506188        1051               
vulnerabilities        alpine:3.7             2019-04-28T13:04:10.510544        1125               
vulnerabilities        alpine:3.8             2019-04-28T13:04:08.909376        1220               
vulnerabilities        alpine:3.9             2019-04-28T13:04:08.308430        1218               
vulnerabilities        amzn:2                 2019-04-28T13:04:14.120807        163                
vulnerabilities        centos:5               2019-04-28T13:04:10.278929        1323               
vulnerabilities        centos:6               2019-04-28T13:04:12.089106        1328               
vulnerabilities        centos:7               2019-04-28T13:04:13.261358        778                
vulnerabilities        debian:10              2019-04-28T13:04:12.408950        20095              
vulnerabilities        debian:7               2019-04-28T13:04:12.643238        20455              
vulnerabilities        debian:8               2019-04-28T13:04:15.673385        21557              
vulnerabilities        debian:9               2019-04-28T13:04:07.625729        20319              
vulnerabilities        debian:unstable        2019-04-28T13:04:13.900741        20952              
vulnerabilities        ol:5                   2019-04-28T13:04:14.578852        1230               
vulnerabilities        ol:6                   2019-04-28T13:04:11.595896        1401               
vulnerabilities        ol:7                   2019-04-28T13:04:11.857659        889                
vulnerabilities        ubuntu:12.04           2019-04-28T13:04:09.166082        14948              
vulnerabilities        ubuntu:12.10           2019-04-28T13:04:07.207705        5652               
vulnerabilities        ubuntu:13.04           2019-04-28T13:04:09.730803        4127               
vulnerabilities        ubuntu:14.04           2019-04-28T13:04:10.044314        18504              
vulnerabilities        ubuntu:14.10           2019-04-28T13:04:14.350013        4456               
vulnerabilities        ubuntu:15.04           2019-04-28T13:04:07.980058        5789               
vulnerabilities        ubuntu:15.10           2019-04-28T13:04:16.144666        6513               
vulnerabilities        ubuntu:16.04           2019-04-28T13:04:12.989542        15484              
vulnerabilities        ubuntu:16.10           2019-04-28T13:04:14.885677        8647               
vulnerabilities        ubuntu:17.04           2019-04-28T13:04:15.133018        9157               
vulnerabilities        ubuntu:17.10           2019-04-28T13:04:15.914109        7935               
vulnerabilities        ubuntu:18.04           2019-04-28T13:04:09.501847        9736               
vulnerabilities        ubuntu:18.10           2019-04-28T13:04:15.418772        7823               
vulnerabilities        ubuntu:19.04           2019-04-28T13:04:08.653333        6274 
```

**Note:** In order to return vulnerability results on analyzed images feed data must be synced.

#### System wait

You can run the following command to wait until Anchore Engine is available and ready. This can be useful when waiting for vulnerability data to sync on intial installation. `anchore-cli system wait`

```
# Blocking operation that will return when anchore-engine is available and ready

root@4c0a95557659:/anchore-engine# anchore-cli system wait
Starting checks to wait for anchore-engine to be available timeout=-1.0 interval=5.0
API availability: Checking anchore-engine URL (http://localhost:8228)...
API availability: Success.
Service availability: Checking for service set (catalog,apiext,policy_engine,simplequeue,analyzer)...
Service availability: Success.
Feed sync: Checking sync completion for feed set (vulnerabilities)...
Feed sync: Success.
```

#### Feed sync failures

If you are running into feed sync failures a good place to begin investigation is the the policy engine service logs (`/var/log/anchore/anchore-policy-engine.log`)

### Image analysis

Image analysis is performed as a distinct, asynchronous, and scheduled task driven by queues that analyzer workers periodically poll. Image records have a small state-machine as follows:

#### Image analysis failures

If you run into issues with images failing analysis a good place to start inspecting is the analyzer logs (`/var/log/anchore/anchore-worker.log`)

The analyzer is the only component that can set an image state to 'analysis_failed', so you should be able to see a record of what happened. 

### Registry 