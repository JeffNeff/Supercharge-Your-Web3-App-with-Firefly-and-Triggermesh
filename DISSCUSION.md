# Supercharge Your Web3 App with Firefly and Triggermesh

## Introduction

Firefly exposes an awesome eventing system that enforces event driven application design. They expose a well refined schema registry, a well thought-out pub-sub scription system, REST api's and SDK's for managing your eventing infrastructure, and awesome user interfaces for managing everything.

With Firefly, our applications can create, subscribe, and react to on-chain events with realitve ease. However, what about off-chain events? What if we want to react to events from a cloud service, or third party service(s) with the same workflow as our on-chain events?

By integrateing Triggermesh with Firefly, we can now create a unified eventing system that allows us to react to events from any source, and any destination.

During this guide we will be looking at how to integrate [Triggermesh](https://www.triggermesh.com/) Event Sources into a [Firefly](https://hyperledger.github.io/firefly/overview/) application with the use [FireMesh](https://github.com/triggermesh/FireMesh). FireMesh is a beta Triggermesh product that allows us to easily integrate Triggermesh Event Sources, and soon Targets, into your Firefly eventing workflow.


## Objective

We will be creating a new Firefly development enviorment and then integrate both the [Triggermesh Kafka Source]() and the
[Triggermesh Webhook Source]() into this enviorment, effectively exposeing an HTTP endpoint that accepts arbitrary JSON data and reading from a particular Topic off a Kafa stream to create events that can be accesed via the standard Firefly API(s)/SDK(s)).


## Setup Local FF Environment
### Prerequisites

* [Docker](https://docs.docker.com/get-docker/)
* [Docker Compose](https://docs.docker.com/compose/install/)
* [Firefly CLI](https://hyperledger.github.io/firefly/gettingstarted/firefly_cli.html)

### Setup

After ensuring that that Docker is running. Create a new Firefly Project named `dev` with one member.

```
ff init dev 1
```

Then start the enviorment. This will return two URL's that we will use later.

```
ff start dev
this will take a few seconds longer since this is the first time you're running this stack...
done

Deployed TokenFactory contract to: 0x8389b6b85bbbaa7710bb76f1418227728a08bd7d
Source code for this contract can be found at /home/jeff/.firefly/stacks/dev/runtime/contracts/source

Web UI for member '0': http://127.0.0.1:5000/ui
Sandbox UI for member '0': http://127.0.0.1:5109

To see logs for your stack run:

ff logs dev

```

The sandbox and Web UI Firefly exposes is a great way to experiemnt and see what is going on within the Firefly stack. I encourage you to spend some time here playing.

### Add Triggermesh Event Sources

Before adding any of the Triggermesh Event Sources, the `FireMesh` service needs to be avalible in the stack.

`FireMesh` is a service that allows Triggermesh components, or anything that uses the Cloudevent standard, to communicate effectively with the Firefly eventing system.

To install `FireMesh`, into your Firefly environment, find the docker-compose file found at `~/.firefly/stacks/dev/dockerc-compose.yaml` and add the following to the `services` section:

```yaml
    firemesh:
        image: gcr.io/triggermesh/firemesh
        ports:
            - 8080
        environment:
            # If no topic is provided here, firemesh will use the incoming event type to dynamically set the topic name.
            TOPIC: firemesh
            FF: http://firefly_core_0:5000
```

Note that the `TOPIC` enviorment variable is optional and the only configurable here. If no topic is provided, FireMesh will use the incoming event type to dynamically set the topic name.


Once you have added the FireMesh service to your docker-compose file, we can now also add a Triggermesh event source.

We are now going to add the following trigermesh componenets:

- [Kafka Source]
- [Webhook Source]

After creating the required Kafka resources, we can add the following to the `services` section of the docker-compose file, right under our `firemesh` resource:

```yaml
    webhook:
        platform: linux/amd64
        image: gcr.io/triggermesh/webhooksource-adapter
        environment:
          WEBHOOK_EVENT_TYPE: webhook.event
          WEBHOOK_EVENT_SOURCE: webhook
          K_SINK: "http://firemesh:8080"
        ports:
          - 8000:8080

    kafka-source:
      # platform: linux/amd64
      image: gcr.io/triggermesh/kafkasource-adapter
      environment:
        # Stream Pool Name
        TOPICS:
        # <Tennancy>/<email>/<OCID>
        USERNAME:
        # Auth Token
        PASSWORD:
        # Kafka Connection Settings
        BOOTSTRAP_SERVERS:
        # Dont Touch me
        SECURITY_MECHANISMS: PLAIN
        GROUP_ID: ocikafka-group
        SKIP_VERIFY: "true"
        SASL_ENABLE: "true"
        TLS_ENABLE: "true"
        K_SINK: "http://firemesh:8080"
      ports:
        - 8080


```

Restart your dev enviorment:
```
ff stop dev
ff start dev
```


And thats it! We can now expect to see any events that appear in our Kafka stream or that we send to our Webhook source should appear in our Firefly event stream!

### Webhook Source Example

To utilize the Webhook source, we can send events with a simple POST request containing a JSON payload on port `8000` of the local host.

```
curl -X POST -H "Content-Type: application/json" -d '{"message":"Hello World"}' http://localhost:8000
```


### Subscribe to Firefly Events

**Note** If you just want a quick way to view all of your events you can use the FireFly Web UI. The Web UI is a great way to see what is going on in your FireFly stack. You can find the Web UI in the logs of the terminal that you started the FireFly stack in.

To begin consuming the events from the Firefly event stream, first create a new subscription. This can be accomplished with the following command:

```
curl --location --request POST 'http://127.0.0.1:5000/api/v1/namespaces/default/subscriptions' \
--header 'Content-Type: application/json' \
--data-raw ' {
	"transport": "websockets",
	"name": "app1",
	"filter": {
	  "blockchainevent": {
		"listener": ".*",
		"name": ".*"
	  },
	  "events": ".*",
	  "message": {
		"author": ".*",
		"group": ".*",
		"tag": ".*",
		"topics": ".*"
	  },
	  "transaction": {
		"type": ".*"
	  }
	},
	"options": {
	  "firstEvent": "newest",
	  "readAhead": 50
	}
  }'
```
Above we just create a new subscription, `app1`, with a "catch-all" filter.

Now that the subscription is in place.. we can begin consuming messages from it.


One way we can do this is with the `websocat` cli (but any ws client will work).

```bash
websocat "ws://127.0.0.1:5000/ws?namespace=default&name=app1&autoack=true"
```

Notice this will not show us the data, only a refrence to this data. we will have to retrieve the data
via another API call.

### Deploying the Sources in production

Its actually much easier deploying the Triggermesh event sources in production than it is in development. All of the Sources come with a corresponding CRD that makes it easy to deploy them into a Kubernetes cluster.

For example, to deploy the Kafka Source, after installing Triggermesh , we can use a manifest like the following:

```yaml
apiVersion: sources.triggermesh.io/v1alpha1
kind: KafkaSource
metadata:
  name: oci-streaming-source
spec:
  groupID: <group-id>
  bootstrapServers:
    - <bootstrap-server>
  topics:
    - <topic>
  auth:
    saslEnable: true
    tlsEnable: true
    securityMechanism: PLAIN
    username:
    password:
      valueFromSecret:
        name: oci-kafka
        key: pass
  sink:
	# Where to send the events goes here
```



### Leveraging Other Triggermesh Sources

The Triggermesh project has a number of other event sources that can be used to integrate with Firefly. The process for deploying the sources in production is well documented in the Triggermesh [documentation](docs.triggermesh.io). However, they are not documented to be deployed in this mannor. So in order to deploy them via docker-compose, you need to know what the expected enviorment variables are for each of the Source componenets you require.

Instead of listing all of the enviorment variables here, I will instead show you how to retrieve them for yourself. (teach a man to fish, right?)

#### Find the Required Environment Variables

For this example we will be going after the Oracle Cloud Metrics Source (or OCIMetricsSource).

* Navigate to the folder containing the reconciler for all of the Triggermesh Source components:
```
https://github.com/triggermesh/triggermesh/tree/main/pkg/sources/reconciler
```

In this folder you will find a folder for every Triggermesh Source. For example, the OCIMetricsSource is located here. In these folders contains the `Reconciler`, or the code that is responsible for building and deploying a Source adapter (including adding the enviorment variables). So here we can find the enviorment variables that are required for every Source, including the OCIMetricsSource in question.


* Navigate to the OCIMetricsSource reconciler:
```
https://github.com/triggermesh/triggermesh/tree/main/pkg/sources/reconciler/ocimetricssource
```

In this folder, the only file that we care about is `adapter.go`. Here will will be able to find the enviorment variables that are required for the OCIMetricsSource.

In this file we can find all of the enviorment variables are exported at the top as constants [here.](https://github.com/triggermesh/triggermesh/blob/bfacde991c3b5400afd1cea94fd736a5b5f39e0b/pkg/sources/reconciler/ocimetricssource/adapter.go#L35)

```go
const (
	oracleAPIKey            = "ORACLE_API_PRIVATE_KEY"
	oracleAPIKeyPassphrase  = "ORACLE_API_PRIVATE_KEY_PASSPHRASE"
	oracleAPIKeyFingerprint = "ORACLE_API_PRIVATE_KEY_FINGERPRINT"
	userOCID                = "ORACLE_USER_OCID"
	tenantOCID              = "ORACLE_TENANT_OCID"
	oracleRegion            = "ORACLE_REGION"
	pollingFrequency        = "ORACLE_METRICS_POLLING_FREQUENCY"
	metrics                 = "ORACLE_METRICS"
)
```

If we scroll down to the `MakeAppEnv` function, we can see how each of these variables are being set [here](https://github.com/triggermesh/triggermesh/blob/bfacde991c3b5400afd1cea94fd736a5b5f39e0b/pkg/sources/reconciler/ocimetricssource/adapter.go#L79).

***IMPORTANT*** NOT EVERY TRIGGERMESH SOURCE COMPONENT CURRENTLY WORKS WITH DOCKER-COMPOSE. If there is any logic in the `Reconciler` other than setting the enviorment variables, then the Source will not work in a development enviorment like this, and you will have to deploy it in a Kubernetes cluster.

**Note:** Just in case you are curious, and want to know how these enviorment variables are then used by the Source adapter, you can find the code for the Source adapters [here](https://github.com/triggermesh/triggermesh/tree/main/pkg/sources/adapter)
