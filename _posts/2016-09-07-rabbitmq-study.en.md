---
layout: post
title:  "Studying AMQP protocol"
date:   2016-09-07 18:08:00 -0300
lang: en
ref: rabbitmq
categories: rabbitmq amqp
description: Basics about AMQP protocol and RabbitMQ with Java
---

AMQP protocol (Advanced Message Queue Protocol) is an asynchronous message exchange specification which allows for interoperability between different platforms. That is one of the main features provided by AMQP. Besides the possibilities to achieve the same with JMS (another specification that provides asynchronous message exchange), JMS rely with proprietary solutions with some drawbacks depending on regarded context. This post's goal is to provide summary content around what I had studied about AMQP. If you want to see more detailed comparision between AMQP and JMS it worths reading [Understanding the Diferences between AMQP and JMS](http://www.wmrichards.com/amqp.pdf) by Mark Richards.

## General AMQP's architecture
Before start using any AMQP broker, I figured out to understand the main components from AMQP model and how they interact with each other. The three main components are:
  - **Exchange**: this component receives messages and routes them to proper message queues.

  - **Message queue**: responsible for keeping messages and to forward them to binded consumers.

  - **Binding**: binds exchanges with message queues defining routing rules.

Besides to know AMQP's main components it's important to know the entities involved in exchanging messages with an AMQP server.

  - **Publisher**: is the entity responsible for sending messages to queue's server (broker). A publisher usually present messages to an Exchange which in turn decides the message queue to forward.

  - **Consumer**: consumes from message queues (likely to perform some processing based on message's content).

  - **Broker**: is the queues server which implements the AMQP specification.

The figure that follows shows tha basic components addressed so far and how they relate to each other:

![AMQP's architecture]({{ "/assets/amqp.png" | absolute_url }})

And now lets take a look for more detail on each component:

### Message Queues

Message queues have some properties that when combined together can be used to build conventional entities from middlewares. Some properties that worth studying are:

  - **private or shared**: this property defines if it's a private message queue with just one consumer allowed to listen for messages or if it's a shared queue where more than one consumer are allowed to listen for messages. Queues declared as **exclusive** are considered private queues.
  
  - **durable or temporary**: defines if a queue will be destroyed or not when rebooting the server.
  
  - **client-named or servernamed**: if the client defines the name of the queue during it's declaration it will be **client-named**. Otherwise, if the server by itself take the responsibility for naming the queue, it will be considered **servernamed**.

When combining such properties, one can create some entities already known and used by the most middlewares as follows:
  
  - **store and forward queue**: an ordinary kind of queue which just keeps the messages and deliver them among consumers relying with round-robin algorithm. They are usually *durable* and *shared*.
  
  - **private reply queue**: kind of queue that receives the messages and forwards them to just one bound consumer. They are usually *temporary* and *server named*.
  
  - **private subscription queue**: collects messages from different sources and forwards them for just one bound consumer. They are usually *temporary*, *private* and *server named*.

### Exchanges
  - Receives all messages created by publishers and send messages to *Queues* following pre-defined rules which are called *Bindings*. It is possible to declare nameless *default* Exchanges and also Exchanges with names so they can be bound with Queues.

  - Exchanges are abstract models which allows to add extensibility to AMQP servers (even though at the cost of interoperability).

  - There are **direct** and **topic** kinds of exchanges.

### Routing Key
  - Is virtual address which allows for the Exchange to decides how to route any message. É um endereço virtual que permite à Exchange decidir como rotear uma mensagem. It's an available field in the header of an AMQP message.
  
  - It is easy to make a resemblance with a mail server where the routing key is the email address, the queue is the inbox and the exchange is such as an MTA (mail transfer agent) which knows how to route the message based on mail address.

### Binding
  - Defines the relation between exchangeas and queues which can be achieved in different ways. It must be declared with a name or pattern which can be used during routings performed by exchanges. Before routing to an specific queue, the routing-key header's field is compared with the name defined in binding rule. Such comparisions can be direct or can also be through pattern matching (i.e. `rabbit.*`). Using such pattern, messages which come with routing keys `rabbit.fast` or `rabbit.slow` or `rabbit.anything`, will be redirected to the queue defined on binding rule `rabbit.*`. An `*` matches with a word and `#` matches with zero on more words.  

#### Queues declarations examples
##### Creating a shared queue
A shared queue can be created using the default exchange just by not declaring it's exchange name. Some pseudo-code follows to clarify the example:

Declaring a queue with `app.atticus` name:

```
Queue.declare
    queue=app.atticus
```

Consumer:

```
Basic.Consume
    queue=app.atticus
```

Publisher:

```
Basic.Publish
    routing-key=app.atticus
```

##### Creating a reply queue
A reply queue usually is declared as temporary and and private so its name is declared by the broker. Its name is returned in Ok responses to the client.

Example of reply queue declaration:

```
Queue.declare
    queue=<empty>
    exclusive=TRUE

// server response
S:Queue.Declare-Ok
    queue=tmp.star
```

Sending a message to the queue:

```
Basic.Publish
    exchange=<empty>
    routing-key=tmp.star
```

##### Declaring an pub-sub queue
Allows that messages with routing-keys that matches with binding defined rules can be delivered to a specified queue. The AMQP protocol does not define a *subscription* entity.

The following pseudocodes presents how to do it: 

```
Queue.Declare
    queue=<empty>
    exclusive=TRUE
S:Queue.Declare-Ok
    queue=tmp.vogel

Queue.Bind
    queue=tmp.vogel
    TO exchange=amq.topic
    WHERE routing-key=DAS.IST.EIN.*
```

Consumer:

```
Basic.Consume
    queue=tmp.vogel
```

Publisher:

```
Basic.Publish
    exchange=amqp.topic
    routing-key=DAS.IST.EIN.VOGEL
```

Looking to the presented pseudocodes, one can see that the examples are using statements which looks like Class methods (such as used by object oriented programming languages). Such structure does not follow this model by chance.
The AMQP spec defines such a statements architecture which is based in structure presented in examples before. The idea was to facilitate the protocol design once the Middlewares by itself are too much complicated.

The most interesting point here is that if you study the Rabbitmq's API (which is an AMQP implementation broker), you should note that the sintax defined by the API's methods are similar with low level statements defined by the protocol.

As follows is an example of a java code which sends messages to a queue:

```java
public class Send {
    private final static String QUEUE_NAME = "hallo";

    public static void main(String[] args) throws IOException, TimeoutException {
        // creates RabbitMQ connection factory
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        // effectively creates a connection with declared factory
        Connection connection = factory.newConnection();

        // create a channel to start switching messages with broker
        Channel channel = connection.createChannel();

        // declares a queue as durable and shared
        boolean durable = true;
        channel.queueDeclare(QUEUE_NAME, durable, false, false, null);

        // defines the prefetch count as 1 so the queue does not accepts more messages
        // until the current received message won't be processed
        int prefetchCount = 1;
        channel.basicQos(prefetchCount);

        // concats all strings sent as arguments for this simple app
        String message = getMessage(args);

        // publish a message to declared queue.
        // here we are using a default exchange because the name of an existent exchange won't set
        channel.basicPublish("", QUEUE_NAME,
                MessageProperties.PERSISTENT_TEXT_PLAIN,
                message.getBytes());

        System.out.println(" [x] Sent '" + message + "'");

        channel.close();
        connection.close();
    }

    private static String getMessage(String[] args) {
        if (args.length < 1) {
            return "Hallo Welt!";
        }
        return joinStrings(args, " ");
    }

    private static String joinStrings(String[] strings, String delimiter) {
        int length = strings.length;
        if (length == 0)
            return "";

        StringBuilder words = new StringBuilder(strings[0]);
        for (int i = 1; i < length; i++) {
            words.append(delimiter).append(strings[i]);
        }
        return words.toString();
    }
}
```

And now an example of a message consumer which listens for messages sent to the "hallo" queue:

```java
public class Recv {

    private final static String QUEUE_NAME = "hallo";

    public void receive() throws IOException, TimeoutException {
        // creates RabbitMQ connection factory
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        // effectively creates a connection with declared factory
        Connection connection = factory.newConnection();

        // create a channel to start switching messages with broker
        Channel channel = connection.createChannel();

        // declares a queue as durable and shared
        boolean durable = true;
        channel.queueDeclare(QUEUE_NAME, durable, false, false, null);
        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        // defines the prefetch count as 1 so the queue does not accepts more messages
        // until the current received message won't processed
        int prefetchCount = 1;
        channel.basicQos(prefetchCount);

        // Creates the callback defined for message consuming
        Consumer consumerCallback = new DefaultConsumer(channel) {

            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                    BasicProperties properties, byte[] body)
                    throws IOException {
                String message = new String(body, "UTF-8");
                System.out.println(" [x] Received '" + message + "'");

                try {
                    doWork(message);

                    // acknowledge the message so Rabbitmq can delete the message from queue
                    long deliveryTag = envelope.getDeliveryTag();
                    channel.basicAck(deliveryTag, false);

                } catch (InterruptedException e) {
                    System.out.println(" [X] Message weren't processed");
                } finally {
                    System.out.println(" [X] Done");
                }
            }

            private void doWork(String task) throws InterruptedException {
                System.out.printf(" \t[X] starting processing task: {{ "%s"}} \n", task);
                for (char ch : task.toCharArray()) {
                    if (ch == '.') Thread.sleep(1000);
                }
            }

        };

        boolean autoAck = false;
        channel.basicConsume(QUEUE_NAME, autoAck, consumerCallback);

    }

    public static void main(String[] args) throws IOException, TimeoutException {
        Recv recv = new Recv();
        recv.receive();
    }
}
```

This was another edition of the logbook of devadventures.

The main idea here wasn't to get into details of Publisher or Consumer implementation. The code presented before was just added as an example of AMQP broker communication.

On next post I will put a bit about what I have studied of RabbitMQ.
