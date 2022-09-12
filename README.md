# Supercharge Your Web3 App with Firefly and Triggermesh

## Introduction

### What does FireFly have to do with event-driven architecture?

[Hyperledger FireFly](https://github.com/hyperledger/firefly) is the first open source Supernode: a complete stack for enterprises to build and scale secure Web3 applications. The FireFly API for digital assets, data flows, and blockchain transactions makes it radically faster to build production-ready apps on popular chains and protocols.

Firefly exposes an awesome eventing system that enables event driven application design. They expose a schema registry, a pub/sub subscription mechanism, REST APIs and SDKs for managing your eventing infrastructure, and awesome user interfaces for managing everything.

With Firefly, our applications can create, subscribe, and react to on-chain events with realitve ease. However, what about off-chain events? What if we want to react to events from a cloud service, or third party service with the same workflow as our on-chain events?

By integrating TriggerMesh with Firefly, we can now create a unified eventing system that allows us to react to events from any source, and any destination.

During this guide we will be looking at how to integrate [TriggerMesh](https://www.triggermesh.com/) Event Sources into a [Firefly](https://hyperledger.github.io/firefly/overview/) application with the use of [FireMesh](https://github.com/triggermesh/FireMesh). FireMesh is an experimental TriggerMesh component that lets you easily integrate TriggerMesh Event Sources, and soon Targets, into your Firefly eventing workflow.


### Goal for this post

We will be creating a new Firefly development environment and then integrate both the [TriggerMesh Kafka Source]() and the
[Triggermesh Webhook Source]() into this enviorment, effectively exposing an HTTP endpoint that accepts arbitrary JSON data and reading from a particular Kafka topic to create events that can then be accessed via the standard Firefly API and SDK.


## Hands on with FireFly and TriggerMesh
### Prerequisites

* [Docker](https://docs.docker.com/get-docker/)
* [Docker Compose](https://docs.docker.com/compose/install/)
* [Firefly CLI](https://hyperledger.github.io/firefly/gettingstarted/firefly_cli.html)

### Setup

You should now have access to the FireFly CLI called "ff".

After ensuring that Docker is running. Create a new Firefly Project named `dev` with one member.

```
ff init dev 1
```

Then start the environment. This will return two URLs that we will use later.

```
ff start dev
```
This will take a few seconds if this is the first time you're running this stack. It should result in output similar to below:

```
Deployed TokenFactory contract to: 0x8389b6b85bbbaa7710bb76f1418227728a08bd7d
Source code for this contract can be found at /home/jeff/.firefly/stacks/dev/runtime/contracts/source

Web UI for member '0': http://127.0.0.1:5000/ui
Sandbox UI for member '0': http://127.0.0.1:5109
```

To see logs for your stack run:
```
ff logs dev

```

The sandbox and Web UI Firefly exposes is a great way to experiment and see what is going on within the Firefly stack. I encourage you to spend some time here playing.

### Add TriggerMesh Event Sources

Before adding any of the TriggerMesh Event Sources, the `FireMesh` component needs to be available in the stack.

`FireMesh` is a component that allows TriggerMesh, or anything that uses the [CloudEvents](https://cloudevents.io/) standard, to communicate effectively with the Firefly eventing system.

To install `FireMesh`, into your Firefly environment, find the docker-compose file found at `~/.firefly/stacks/dev/docker-compose.yaml` and add the following to the `services` section:

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

Note that the `TOPIC` environment variable is optional and the only configurable here. If no topic is provided, FireMesh will use the incoming event type to dynamically set the topic name.


Once you have added the FireMesh service to your docker-compose file, we can now also add a TriggerMesh event source.

We are now going to add the following TriggerMesh components:

- [KafkaSource](https://docs.triggermesh.io/apis/sources/#sources.triggermesh.io/v1alpha1.KafkaSource)
- [WebhookSource](https://docs.triggermesh.io/apis/sources/#sources.triggermesh.io/v1alpha1.WebhookSource)

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

Restart your dev environment:
```
ff stop dev
ff start dev
```


And that's it! We can now expect that any events that appear in our Kafka topic or that we send to our Webhook Source will appear in our Firefly event stream!

### Webhook Source Example

To use the Webhook source, we can send events with a simple POST request containing a JSON payload on port `8000` of the local host.

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

Now that the subscription is in place we can begin consuming messages from it.

One way we can do this is with the `websocat` cli (but any ws client will work).

```bash
websocat "ws://127.0.0.1:5000/ws?namespace=default&name=app1&autoack=true"
```

Notice this will not show us the data, only a reference to this data. We will have to retrieve the data
via another API call.

### Deploying the Sources in production

It is actually much easier to deploy the TriggerMesh event sources in production than it is in development. All of the Sources come with a corresponding CRD that makes it easy to deploy them into a Kubernetes cluster.

For example, to deploy the Kafka Source, after installing TriggerMesh , we can use a manifest like the following:

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

## Leveraging Other TriggerMesh Sources

The TriggerMesh project has a number of other event sources that can be used to integrate with Firefly. The process for deploying the sources in production is well documented in the TriggerMesh [documentation](docs.triggermesh.io). However, they are not documented to be deployed in this manner. So in order to deploy them via docker-compose, you need to know what the expected environment variables are for each of the Source components that you require.

Instead of listing all of the environment variables here, I will instead show you how to retrieve them for yourself. (teach a man to fish, right?)

#### Find the Required Environment Variables

For this example we will be going after the Oracle Cloud Metrics Source (or OCIMetricsSource).

* Navigate to the folder containing the reconciler for all of the TriggerMesh Source components:
```
https://github.com/triggermesh/triggermesh/tree/main/pkg/sources/reconciler
```

In this folder you will find a folder for every TriggerMesh Source. For example, the OCIMetricsSource is located here. In these folders contains the `Reconciler`, or the code that is responsible for building and deploying a Source adapter (including adding the environment variables). So here we can find the environment variables that are required for every Source, including the OCIMetricsSource in question.


* Navigate to the OCIMetricsSource reconciler:
```
https://github.com/triggermesh/triggermesh/tree/main/pkg/sources/reconciler/ocimetricssource
```

In this folder, the only file that we care about is `adapter.go`. Here will will be able to find the environment variables that are required for the OCIMetricsSource.

In this file we can find all of the environment variables are exported at the top as constants [here.](https://github.com/triggermesh/triggermesh/blob/bfacde991c3b5400afd1cea94fd736a5b5f39e0b/pkg/sources/reconciler/ocimetricssource/adapter.go#L35)

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

***IMPORTANT*** NOT EVERY TRIGGERMESH SOURCE COMPONENT CURRENTLY WORKS WITH DOCKER-COMPOSE. If there is any logic in the `Reconciler` other than setting the environment variables, then the Source will not work in a development environment like this, and you will have to deploy it in a Kubernetes cluster.

**Note:** Just in case you are curious, and want to know how these environment variables are then used by the Source adapter, you can find the code for the Source adapters [here](https://github.com/triggermesh/triggermesh/tree/main/pkg/sources/adapter).
