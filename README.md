# Leveraging Modern Blockchain Technology at your Event Broker

If you are designing an Event Driven Architecture (EDA) for your application with the following requirements, then you may want to consider replacing your existing broker with a similar architecture covered in this article.

* One time guarenteed event delivery. (That means we know for sure that the event was delivered to, and properly consumed by, the consumer.)

* Ablility to store events and deliver them to consumers at a later time. ( possibly even consumers that do not exist yet. )

* Maintaining a history of events in the order that they occured is important.

*



## Architecture Overview

Obviously we do not want to build everything from scratch, so to bootstrap our solution we will leverage the Hyperledger project [Firefly](https://hyperledger.github.io/firefly/) as a starting point. Firefly does all of the hard work for us by exposing a rest api with a set of endpoints that we can use to manage our events.

### What is Firefly

// Breif overview of Firefly

## Setup and Configuration

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

I would recommend spending some time familiarizing yourself with the Firefly docs. They are very well structured and easy to follow. [https://hyperledger.github.io/firefly/](https://hyperledger.github.io/firefly/)

### Project Configuration

Now we need two projects to hold an example event consumer and producer.

Create two new folders named `event-producer` and `event-consumer`. (These will be go projects so put them in your GOPATH if they need to be there.

```
mkdir event-producer
mkdir event-consumer
```


After verifying everything is operationl, we can begin designing our architecture.

Its good practice to implement/maintain a well currated collection of data schemas that are relevant to your architecture.
This is a great place to start off development (IMO).

1. Define a schema for the data you want to store.

This is acomplished by sending OpenAPI spec of the object to the `/api/v1/namespaces/default/datatypes?confirm=true` endpoint
of one of the nodes deployed previously.

Here is an example golang function defining a new Datatype, `Widget`. (Remember there is also swagger endpoint avalibe on the nodes :)
 http://localhost:5000/api#/ )
```
func broadcastNewDataType() {
	url := "http://127.0.0.1:5000/api/v1/namespaces/default/datatypes?confirm=true"
	method := "POST"

	payload := strings.NewReader(`{
			  "name": "widget",
			  "version": "0.0.3",
			  "value": {
				"$id": "https://example.com/widget.schema.json",
				"$schema": "https://json-schema.org/draft/2020-12/schema",
				"title": "Widget",
				"type": "object",
				"properties": {
				  "id": {
					"type": "string",
					"description": "The unique identifier for the widget."
				  },
				  "name": {
					"type": "string",
					"description": "The person's last name."
				  }
				}
			  }
			}`)

	client := &http.Client{}
	req, err := http.NewRequest(method, url, payload)

	if err != nil {
		fmt.Println(err)
		return
	}
	req.Header.Add("accept", "application/json")
	req.Header.Add("Request-Timeout", "120s")
	req.Header.Add("Content-Type", "application/json")

	res, err := client.Do(req)
	if err != nil {
		fmt.Println(err)
		return
	}
	defer res.Body.Close()

	body, err := ioutil.ReadAll(res.Body)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(string(body))
}
```

If you are not a fan of golang. here is a curl request that does the same thing:

```
curl --location --request POST 'http://127.0.0.1:5000/api/v1/namespaces/default/broadcast/datatype' \
--header 'accept: application/json' \
--header 'Content-Type: application/json' \
--data-raw '{
  "name": "widget",
  "version": "0.0.2",
  "value": {
    "$id": "https://example.com/widget.schema.json",
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "title": "Widget",
    "type": "object",
    "properties": {
      "id": {
        "type": "string",
        "description": "The unique identifier for the widget."
      },
      "name": {
        "type": "string",
        "description": "The person'\''s last name."
      }
    }
  }
}'
```


Now lets create some functionality to post our Widget data to the Blockchain.


One way we could accomplish this in golang:
```
// https://hyperledger.github.io/firefly/tutorials/broadcast_data.html
func broadcastInterface(ev interface{}) {
	url := "http://127.0.0.1:5000/api/v1/namespaces/default/messages/broadcast"
	method := "POST"
	// ev := struct {
	// 	Name       string `json:"name"`
	// 	Age        int    `json:"age"`
	// 	Occupation string `json:"occupation"`
	// }{
	// 	Name:       "Jeff",
	// 	Age:        31,
	// 	Occupation: "God",
	// }
	bmd := BroadcastMessageData{Value: ev}
	bmh := BroadcastMessageHeader{Tag: "broadcast", Topics: []string{"broadcast"}}
	bm := BroadcastMessage{Data: []BroadcastMessageData{bmd}, Header: bmh}

	b, err := json.Marshal(bm)
	if err != nil {
		log.Println("marshal:", err)
		return
	}
	ior := bytes.NewBuffer(b)

	client := &http.Client{}
	req, err := http.NewRequest(method, url, ior)
	if err != nil {
		log.Println("newrequest:", err)
		return
	}
	req.Header.Add("accept", "application/json")
	req.Header.Add("content-type", "application/json")
	res, err := client.Do(req)
	if err != nil {
		log.Println("do:", err)
		return
	}
	defer res.Body.Close()
	body, err := ioutil.ReadAll(res.Body)
	if err != nil {
		log.Println("readall:", err)
		return
	}
	log.Printf("%s %s %s", method, url, body)
}
```

In curl this can be accomplished via:
```
curl --location --request POST 'http://127.0.0.1:5000/api/v1/namespaces/default/messages/broadcast' \
--header 'accept: application/json' \
--header 'Content-Type: application/json' \
--data-raw '{"name":"jeff","id":"12"}'
```

We now need to create an interface to consume this data.

This can be acomplished a number of ways, but ff provides a simple way to do this with lots of included functionality via (`Subscriptions`)[https://hyperledger.github.io/firefly/reference/types/subscription.html].


`Subscriptions` provide an interface for us to not only consume the events, but also to define which events are and are not consumed by the use of filtering and regex expressions against the event data.


Define and Create a new Subscription for our Event Consumer.

This can be acomplished by sending OpenAPI spec of the object to the `/api/v1/namespaces/default/subscriptions` endpoint.


```

func setUpWSSubscription() {
	wss := `{
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
  }`

	url := "http://localhost:5000/api/v1/namespaces/default/subscriptions"
	method := "POST"

	ior := bytes.NewBuffer([]byte(wss))

	client := &http.Client{}
	req, err := http.NewRequest(method, url, ior)
	if err != nil {
		log.Println("newrequest:", err)
		return
	}
	req.Header.Add("accept", "application/json")
	req.Header.Add("content-type", "application/json")
	res, err := client.Do(req)
	if err != nil {
		log.Println("do:", err)
		return
	}
	defer res.Body.Close()
	body, err := ioutil.ReadAll(res.Body)
	if err != nil {
		log.Println("readall:", err)
		return
	}
	log.Printf("%s %s %s", method, url, body)
}
```

After we have a Subscription, we can use it to consume the events by creating a WS connection and specifying the Subscription name we want to use.

```

type Event struct {
	ID           string    `json:"id"`
	Sequence     int       `json:"sequence"`
	Type         string    `json:"type"`
	Namespace    string    `json:"namespace"`
	Reference    string    `json:"reference"`
	Created      time.Time `json:"created"`
	Subscription struct {
		ID        string `json:"id"`
		Namespace string `json:"namespace"`
		Name      string `json:"name"`
	} `json:"subscription"`
	Message struct {
		Header struct {
			ID        string    `json:"id"`
			Type      string    `json:"type"`
			Txtype    string    `json:"txtype"`
			Author    string    `json:"author"`
			Created   time.Time `json:"created"`
			Namespace string    `json:"namespace"`
			Topic     []string  `json:"topic"`
			Datahash  string    `json:"datahash"`
		} `json:"header"`
		Hash      string    `json:"hash"`
		BatchID   string    `json:"batchID"`
		State     string    `json:"state"`
		Confirmed time.Time `json:"confirmed"`
		Data      []struct {
			ID   string `json:"id"`
			Hash string `json:"hash"`
		} `json:"data"`
	} `json:"message"`
}

type WSSubscriptionACK struct {
	ID   string `json:"id"`
	Type string `json:"type"`
}

func consomeMessagesFromWSSubscription(name string) {
	interrupt := make(chan os.Signal, 1)
	signal.Notify(interrupt, os.Interrupt)

	x := "ws://localhost:5000/ws?namespace=default&name=" + name
	log.Printf("connecting to %s", x)

	c, _, err := websocket.DefaultDialer.Dial(x, nil)
	if err != nil {
		log.Fatal("dial:", err)
	}
	defer c.Close()

	done := make(chan struct{})

	defer close(done)
	for {
		_, message, err := c.ReadMessage()
		if err != nil {
			log.Panic("read:", err)
			return
		}
		// log.Printf("recv: %s", message)
		event := &Event{}
		err = json.Unmarshal(message, event)
		if err != nil {
			log.Panic("unmarshal:", err)
			return
		}
		e := &Event{}
		err = json.Unmarshal(message, e)
		if err != nil {
			log.Panic("unmarshal:", err)
			return
		}

		if e.Type == "message_confirmed" {
			downloadMessage(e.Message.Header.ID)
		}
		ack := &WSSubscriptionACK{
			ID:   e.ID,
			Type: "ack",
		}
		ackJSON, err := json.Marshal(ack)
		if err != nil {
			log.Panic("marshal:", err)
			return
		}

		c.WriteMessage(websocket.TextMessage, ackJSON)

	}

}


func downloadMessage(id string) {
	url := "http://192.168.1.18:5000/api/v1/namespaces/default/messages/" + id + "/data"
	// url := "http://192.168.1.18:5000/api/v1/namespaces/default/messages/244cd247-38aa-4ae2-a4e2-03f6fe63ec66/data"
	method := "GET"

	client := &http.Client{}
	req, err := http.NewRequest(method, url, nil)

	if err != nil {
		fmt.Println(err)
		return
	}
	req.Header.Add("accept", "application/json")
	req.Header.Add("Request-Timeout", "120s")

	res, err := client.Do(req)
	if err != nil {
		fmt.Println(err)
		return
	}
	defer res.Body.Close()

	body, err := ioutil.ReadAll(res.Body)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println("")
	fmt.Println("")
	fmt.Println("")
	fmt.Println("Recieved Event:")
	fmt.Println(string(body))
	fmt.Println("")
	fmt.Println("")
	fmt.Println("")
}
```
Now that everything is wired up, give it a try!

Here is the complete code for the example.
```


After successfully consuming a few messages, try shutting down the consumer service, emit a message to the chain, and then start the consumer up.. you might find a, plesent, un-expected result.


Now lets say I want to extend someone elses work that has been done with this methodology or (possibly I forgot the schemas I created xd) Developers can query the Datatype registry for all of the avalble datatypes.

Lets query for the datatype we created above.

```
curl -X 'GET' \
  'http://127.0.0.1:5000/api/v1/namespaces/default/datatypes?limit=25' \
  -H 'accept: application/json' \
  -H 'Request-Timeout: 120s'

[
  {
    "id": "e1c7c309-17cf-4989-a91c-3218a30bee5f",
    "message": "51425bf3-2419-4b41-b33d-6297dddb5b29",
    "validator": "json",
    "namespace": "default",
    "name": "widget",
    "version": "0.0.3",
    "hash": "a4dceb79a21937ca5ea9fa22419011ca937b4b8bc563d690cea3114af9abce2c",
    "created": "2022-08-03T22:41:31.350949913Z",
    "value": {
      "$id": "https://example.com/widget.schema.json",
      "$schema": "https://json-schema.org/draft/2020-12/schema",
      "title": "Widget",
      "type": "object",
      "properties": {
        "id": {
          "type": "string",
          "description": "The unique identifier for the widget."
        },
        "name": {
          "type": "string",
          "description": "The person's last name."
        }
      }
    }
  },
  {
    "id": "f8549eea-a83f-491c-b1aa-599d21618f14",
    "message": "391e007a-d0ae-46ba-af0f-410e30077c23",
    "validator": "json",
    "namespace": "default",
    "name": "widget",
    "version": "0.0.2",
    "hash": "a4dceb79a21937ca5ea9fa22419011ca937b4b8bc563d690cea3114af9abce2c",
    "created": "2022-08-03T22:39:52.348866021Z",
    "value": {
      "$id": "https://example.com/widget.schema.json",
      "$schema": "https://json-schema.org/draft/2020-12/schema",
      "title": "Widget",
      "type": "object",
      "properties": {
        "id": {
          "type": "string",
          "description": "The unique identifier for the widget."
        },
        "name": {
          "type": "string",
          "description": "The person's last name."
        }
      }
    }
  }
]

```

Notice there is two, I included another version of `Widget` because I wanted to demonstrate the versioning system.



Another fun thing we can easily do now is reply all, or a subsection, of our events back into the chain to be re-consumed for debugging and analysis purposes.



This can be acomplished by calling the `/api/v1/namespaces/default/messages?` endpoint and providing a combination of filter parameters to match your needs.

For example
```
/api/v1/namespaces/default/messages?timestamp=%3C2022-08-03T19%3A57%3A39.37863438Z&limit=25
```

```

func replayMessages() {
	url := "http://127.0.0.1:5000/api/v1/namespaces/default/messages?timestamp=%3C2022-08-03T19%3A57%3A39.37863438Z&limit=25"
	// url := "http://192.168.1.18:5000/api/v1/namespaces/default/messages/244cd247-38aa-4ae2-a4e2-03f6fe63ec66/data"
	method := "GET"

	client := &http.Client{}
	req, err := http.NewRequest(method, url, nil)

	if err != nil {
		fmt.Println(err)
		return
	}
	req.Header.Add("accept", "application/json")
	req.Header.Add("Request-Timeout", "120s")

	res, err := client.Do(req)
	if err != nil {
		fmt.Println(err)
		return
	}
	defer res.Body.Close()

	body, err := ioutil.ReadAll(res.Body)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println("")
	fmt.Println("")
	fmt.Println("")
	fmt.Println("Recieved Event:")
	fmt.Println(string(body))
	fmt.Println("")
	fmt.Println("")
	fmt.Println("")
}
```


After sending some events have a browse around the firefly node UI exporer and check out your event history/timeline, your node stats, the subscriptions/ws you created, the registerd Datatype we created.