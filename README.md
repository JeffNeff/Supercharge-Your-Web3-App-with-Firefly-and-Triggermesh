# Integrating Triggermesh Event Sources into Firefly


## Introduction
Firefly exposes an awesome eventing system that makes it easy to start designing your applications in an event driven mannor. They expose a well refined schema registry, a well thought-out pub-sub scription system, REST api's and SDK's for managing your eventing infrastructure, and awesome user interfaces.

With Firefly, our applications can create, subscribe, and react to on-chain events with realitve ease. However, what about off-chain events? What if we want to react to events from a cloud service, or a third party service with the same workflow as our on-chain events?

Immagine, for example, that the web3 application we are developing requires data from a Kafka topic or possibly we just want to expose endpoint(s) that we can point/subscribe webhook events to?


## Purpose Statement

The question we are looking to solve with this blog is:

```
How does one easily integrate these various data sources into our Firefly eventing workflow?
```

It turns out, with realtive ease Triggermesh event sources can be "plugged" into the Firefly stack to easily bring in events from a variety of off-chain sources.


## Setup Local FF Environment
### Prerequisites

* [Docker](https://docs.docker.com/get-docker/)
* [Docker Compose](https://docs.docker.com/compose/install/)
* [golang](https://golang.org/doc/install)
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
        image: firemesh
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
        TOPICS: id
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


And thats it! We can now expect to see any events that appear in our Kafka stream or that we send to our Webhook source should appear in our Firefly event stream.

### Webhook Source Example

To utilize the Webhook source, we can send events with a simple POST request containing a JSON payload on port `8000` of the local host.

```
curl -X POST -H "Content-Type: application/json" -d '{"message":"Hello World"}' http://localhost:8000
```


### Subscribe to Firefly Events

To view the events in the Firefly event stream, First create a new subscription.

This will create a new subscription, `app1`, with a "catch-all" filter.

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

Now that the subscription is in place.. we can begin consuming messages at any point.



```bash
websocat "ws://127.0.0.1:5000/ws?namespace=default&name=app1&autoack=true"
```

Notice this will not show us the data, only a refrence to this data. we will have to retrieve the data
via another API call.
