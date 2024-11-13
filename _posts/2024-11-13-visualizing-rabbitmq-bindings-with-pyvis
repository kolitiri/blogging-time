---
title: Visualizing RabbitMQ bindings with PyVis
description: >-
  Using the PyVis library to generate a graph depicting
  bindings between RabbitMQ exchanges and queues
date: 2024-11-13 08:30:00 +0100
categories: [Python]
tags: [python,pyvis,rabbitmq]
tok: true
image:
  path: /assets/img/illustrations/rabbitmq-bindings-visualization.jpg
  alt: RabbitMQ-PyVis
---

---
For those who don't know, [RabbitMQ](https://www.rabbitmq.com/) is one of the most popular open source messaging brokers.

Taken from [wikipedia](https://en.wikipedia.org/wiki/Message_broker),

> _A message broker is an architectural pattern for message validation, transformation, and routing. It mediates communication among applications, minimizing the mutual awareness that applications should have of each other in order to be able to exchange messages, effectively implementing decoupling._

They way RabbitMQ achieves this is via entities such as exchanges and queues.

If you want to know more about RabbitMQ specifically, [cloudamqp](https://www.cloudamqp.com/blog/part1-rabbitmq-for-beginners-what-is-rabbitmq.html) is a great source of information to get you started.

## RabbitMQ Management Interface
RabbitMQ comes with a Management Interface out of the box.

The interface allows you to create exchanges and queues and bind them with each other, depending on the needs of your system.

However, unfortunately the interface doesn't offer any visualization tools, making it hard to visually check the relations between the various entities.

The easiest option is to simply start from an entry point (i.e an exchange) and traverse through all the connected paths.

This can be very daunting, especially if your system is complicated and comprises of hundredths of entities.

## RabbitMQ Definitions JSON
A great feature of the Management Interface is that it allows you to export the overall configuration (aka definitions) of your exchanges/queues in a JSON file.

The exported file contains all the exchanges and queues that have been created, but also the relationships (bindings) between them.

A part of the definitions file might look something like the one below:

```json
{
    "exchanges": [
        {
            "name": "worker-exchange.rpc",
            "vhost": "/backend",
            "type": "fanout",
            "durable": true,
            "auto_delete": false,
            "internal": false,
            "arguments": {}
        },
        {
            "name": "worker-exchange.rpc.hash",
            "vhost": "/backend",
            "type": "x-consistent-hash",
            "durable": true,
            "auto_delete": false,
            "internal": false,
            "arguments": {"hash-header": "sharding_key"}
        }
    ],
    "queues": [
        {
            "name": "worker-queue.rpc.0",
            "vhost": "/backend",
            "durable": true,
            "auto_delete": false,
            "arguments": {}
        },
        {
            "name": "worker-queue.rpc.1",
            "vhost": "/backend",
            "durable": true,
            "auto_delete": false,
            "arguments": {}
        },
        {
            "name": "worker-queue.rpc.2",
            "vhost": "/backend",
            "durable": true,
            "auto_delete": false,
            "arguments": {}
        }
    ],
    "bindings": [
        {
            "source": "worker-exchange.rpc",
            "vhost": "/backend",
            "destination": "worker-exchange.rpc.hash",
            "destination_type": "exchange",
            "routing_key": "",
            "arguments": {}
        },
        {
            "source": "worker-exchange.rpc.hash",
            "vhost": "/backend",
            "destination": "worker-queue.rpc.0",
            "destination_type": "queue",
            "routing_key": "200",
            "arguments": {}
        },
        {
            "source": "worker-exchange.rpc.hash",
            "vhost": "/backend",
            "destination": "worker-queue.rpc.1",
            "destination_type": "queue",
            "routing_key": "200",
            "arguments": {}
        },
        {
            "source": "worker-exchange.rpc.hash",
            "vhost": "/backend",
            "destination": "worker-queue.rpc.2",
            "destination_type": "queue",
            "routing_key": "200",
            "arguments": {}
        }
    ]
}
```
{: file='definitions.json'}

If you take a closer look, you will notice the following attributes:
- **Exchanges**: A list of all the exchanges in the system
- **Queues**: A list of all the queues in the system
- **Bindings**: A list of all the relationships between exchanges/queues

It becomes obvious very quickly, that the entities (exchanges and queues) could be represented in a graph as nodes connected via edges (bindings).

Given the detailed structure of the definitions file, this should be easy-peasy..

## PyVis
[PyVis](https://pyvis.readthedocs.io/en/latest/introduction.html) is a python library for constructing and visualizing network graphs.

There are other libraries such as [networkx](https://networkx.org/) that are even more powerful, but the nice thing with PyVis is that it is a wrapper around [visjs](https://visjs.github.io/vis-network/examples/) which allows us to create nice, interactive visualizations based on Javascript.

In our case, we won't be processing any extremely complex network relationships anyway, thus it should do the job just fine!

## Generating The Graph
Once you install PyVis, crafting a simple script is pretty straight forward, in less than 100 lines of code!

```python
import json
from typing import Any, Dict

from pyvis.network import Network


EXCHANGE_NODE_COLOR = '#dd4b39'
EXCHANGE_NODE_SIZE = 20
QUEUE_NODE_COLOR = '#00ff1e'
QUEUE_NODE_SIZE = 10
HOVER_WIDTH = 500
EDGE_LENGTH = 200
NETWORK_OPTIONS = """
    const options = {
        "physics": {
            "barnesHut": {
            "centralGravity": 0.05,
            "springLength": 200
            },
            "minVelocity": 0.75,
            "timestep": 0.7
        },
        "edges": {
            "smooth": {
            "type": "straightCross",
            "forceDirection": "none",
            "roundness": 1
            }
        }
    }"""


def build_graph(rabbit_defs: Dict[str, Any]) -> None:
    """Builds a network graph based on exchanges and queues"""

    # Instantiate a Network object
    net = Network(directed=True, select_menu=True, filter_menu=True)
    net.set_options(NETWORK_OPTIONS)

    # Generate the network nodes
    generate_nodes(net, rabbit_defs)

    # Export the graph into a HTML file
    net.show('nx.html', notebook=False)


def generate_nodes(net: Network, rabbit_defs: Dict[str, Any]) -> None:
    """Generates the nodes of the graph"""

    exchanges = [exchange['name'] for exchange in rabbit_defs['exchanges']]

    sources = set([])
    destinations = set([])
    for binding in rabbit_defs['bindings']:
        source = binding['source']
        source_type = 'exchange' if source in exchanges else 'queue'

        destination = binding['destination']
        destination_type = 'exchange' if destination in exchanges else 'queue'

        # Add nodes to the network
        if source not in sources:
            color = EXCHANGE_NODE_COLOR if source_type == 'exchange' else QUEUE_NODE_COLOR
            size = EXCHANGE_NODE_SIZE if source_type == 'exchange' else QUEUE_NODE_SIZE
            net.add_node(source, title=source_type, color=color, size=size)
            sources.add(source)

        if destination not in destinations:
            color = EXCHANGE_NODE_COLOR if destination_type == 'exchange' else QUEUE_NODE_COLOR
            size = EXCHANGE_NODE_SIZE if destination_type == 'exchange' else QUEUE_NODE_SIZE
            net.add_node(destination, title=destination_type, color=color, size=size)
            destinations.add(destination)

        # Add edges to the network
        routing_key = binding['routing_key']
        if routing_key:
            net.add_edge(source, destination, label=routing_key, length=EDGE_LENGTH, hover_width=HOVER_WIDTH)
        else:
            net.add_edge(source, destination, length=EDGE_LENGTH, hover_width=HOVER_WIDTH)


if __name__ == '__main__':
    with open('definitions.json', 'r') as file:
        rabbit_defs = json.load(file)
        build_graph(rabbit_defs)
```

The script is self explanatory, but essentially what we do is the following:
- We define a number of options for our graph (check the [PyVis tutorial](https://pyvis.readthedocs.io/en/latest/tutorial.html) for more details)
- We iterate over the bindings and:
    - Add the sources & destinations as **nodes**
    - Add the direction of the bindings (from source to destination) as **edges**
    - Add any routing keys (if exist) as **labels** on the edges

The end result for our simple example is the graph below:

![Rabbit Graph](/assets/img/illustrations/rabbit-graph.png){: width="972" height="589" }

And the best thing is that this is an HTML file shipped with all the javascript included to interact with the graph.

You can zoom, move your nodes around, highlight them, and even apply various filters directly on your browser.

You can also apply some `physics` rules that dictate how your nodes will behave in space, but I found that a bit useless in our case.

## Conclusion
Overall, I found this a pretty nice way to quickly draw a graph and check the current condition of my RabbitMQ relationships.

It can however become quite overcrowded when you are trying to graph a very complex system, but if you are lucky to be using good naming conventions among your exchanges/queues, you could work around this by creating `groups`.

Groups can be filtered directly in your browser reducing most of the noise.

Finally, [AliceMQ](https://github.com/alicelabs/alicemq) is a nice project that is packed with a lot more functionality, in case you'd like to experiment with, but it is probably an overkill if you simply want to view the relationships between your exchanges/queues.
