---
layout: post
title:  "Estudando o protocolo AMQP"
date:   2016-09-07 18:08:00 -0300
lang: pt
ref: rabbitmq
categories: rabbitmq amqp
description: Básico sobre o protocolo AMQP e RabbitMQ com Java
---

O protocolo AMQP (Advanced Message Queue Protocol), é uma especificação para troca de mensagens assíncronas que tem como principal característica o fato de favorecer a interoperabilidade entre diferentes plataformas. Apesar de essa característica ser possível com JMS (outra especificação de troca de mensagens assíncronas) o mesmo conta com soluções proprietárias que podem trazer sérias desvantagens dependendo do contexto em que uma solução estiver sendo empregada. O objetivo desse post é publicar de forma resumida um pouco do que estudei sobre o protocolo AMQP, portanto, caso queira ver uma comparação bem detalhada entre AMQP e JMS sugiro dar uma lida no documento [Understanding the Diferences between AMQP and JMS](http://www.wmrichards.com/amqp.pdf) de Mark Richards.

## Arquitetura geral do AMQP
Antes de utilizar qualquer broker que implemente a especificação AMQP, procurei conhecer os principais componentes do modelo AMQP e como eles interagem. Os três principais componentes são:
  
  - **Exchange**: componente que recebe as mensagens e faz o roteamento para as filas ou "message queues".
  
  - **Message queue**: responsável por manter as mensagens e encaminhar para os consumidores.
  
  - **Binding**: relaciona exchanges e message queues definindo regras de roteamento.

Além dos componentes principais do AMQP é importante conhecer as entidades envolvidas na troca de mensagens com um servidor AMQP.
  - **Publisher**: é a entidade que envia mensagens para o servidor de filas (ou broker). Um publisher geralmente publica mensagens informando o Exchange que por sua vez decide para qual fila encaminhar a mensagem.
  
  - **Consumer**: responsável por consumir as mensagens das filas (provavelmente para realizar algum processamento baseado no que foi consumido).
  
  - **Broker**: é o servidor de filas propriamente dito. Nesse caso um servidor que implemente a especificação AMQP.

A figura abaixo mostra os componentes básicos discutidos até agora e como os mesmos se relacionam (de forma bem simples):

![Arquitetura do AMQP]({{ "/assets/amqp.png" | absolute_url }})

Agora vamos detalhar um pouco mais cada um dos componentes citados:

### Message Queues
As filas possuem algumas propriedades que quando combinadas podem ser usadas para construir entidades convencionais de middlewares. Algumas propriedades que valem ser citadas são: 
  - **private or shared**: indica se são filas privadas onde apenas um consumidor pode receber as mensagens ou se são filas compartilhadas onde mais de um consumidor pode receber mensagens. Filas declaradas como **exclusive** podem ser consideradas private.
  
  - **durable or temporary**: indica se a fila será destruída ou não quando o servidor reiniciar.
  
  - **client-named or servernamed**: indica se é o client quem define o nome da fila durante a declaração da mesma ou se o próprio servidor irá se encarregar de criar o nome em tempo de execução.

Ao combinar as propriedades acima podemos criar algumas entidades já conhecidas e utilizadas na maioria dos middlewares conforme mostrado abaixo:
  - **store and forward queue**: tipo de fila normal que armazena mensagens e distribui entre os consumers utilizando o algoritmo round-robin. São geralmente do tipo durable, e compartilhadas entre consumers (shared).
  
  - **private reply queue**: recebe as mensagens e encaminha para um único consumer e geralmente são filas temporárias nomeadas pelo servidor.
  
  - **private subscription queue**: coleta mensagens de várias fontes e encaminha para um único consumer. Geralmente são temporárias, privadas e nomeadas pelo servidor.

### Exchanges
  - Recebe as mensagens criadas por publishers e envia para as *Queues* de acordo com critérios pré definidos (Tais critérios são chamados *Bindings*). É possível utilizar Exchanges *default* sem nomes e também exchanges com nomes para que seja possível realizar o binding com Queues.
  
  - Exchanges são modelos abstratos que permitem adicionar extensibilidade a servidores AMQP (mesmo que impactando em interoperabilidade).
  
  - Existem tipos distintos como **direct** e **topic**.

### Routing Key
  - É um endereço virtual que permite à Exchange decidir como rotear uma mensagem. Trata-se de um campo disponível no cabeçalho de uma mensagem AMQP.
  
  - É possível fazer uma analogia com um servidor de email onde o routing key é o email, a fila é o inbox e o exchange é um MTA(mail transfer agent) que baseado no endereço de email decide como rotear a mensagem.

### Binding
  - Relacionamento entre exchange e filas que pode ser feito de formas distintas. É declarado através de um nome ou padrão que é usado durante o roteamento realizado por exchanges. Antes de realizar o roteamento para uma fila específica, campo routing-key do cabeçalho da mensagem é comparado com o nome que foi definido no binding. Podem ser realizadas comparações diretas e comparações baseadas em padrões como por exemplo `rabbit.*`. Dessa forma, mensagens que cheguem com o routing key `rabbit.fast` ou `rabbit.slow` ou `rabbit.qualquercoisa`, serão redirecionadas para a fila definida no binding. Um `*` casa com uma palavra enquanto que `#` casa com zero ou mais palavras.

#### Exemplos de declaração de filas
##### Criando uma shared queue
Uma shared queue pode ser criada utilizando exchange default (bastando para isso não declarar o nome do exchange). Abaixo seguem pseudo-códigos para isso:

Declaração de uma fila com o nome `app.atticus`:

```
Queue.declare
    queue=app.atticus
```

Consumidor:

```
Basic.Consume
    queue=app.atticus
```

Publisher:

```
Basic.Publish
    routing-key=app.atticus
```

##### Criando uma Reply Queue
Uma fila de reply geralmente é declarada como temporária e privada de modo que seu nome é definido pelo próprio broker e devolvido na resposta de Ok para o client.

Exemplo de declaração de queue de reply:

```
Queue.declare
    queue=<empty>
    exclusive=TRUE

// resposta do servidor
S:Queue.Declare-Ok
    queue=tmp.star
```

Enviando mensagem para a fila:

```
Basic.Publish
    exchange=<empty>
    routing-key=tmp.star
```

##### Criando uma Pub-Sub Queue
Permite que mensagens com routing-keys que casem com regras definidas através de binding sejam entregues para uma determinada fila. O protocolo AMQP não define uma entidade de *subscription*. Para declarar uma fila com essa característica é necessário executar passos distintos que são: binding e armazenamento da mensagem.

Os pseudo códigos abaixo mostram como fazer isso:

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

Consumidor:

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

Repare que nos pseudo códigos apresentados, foram utilizados comandos que se assemelham a métodos de classes (como os utilizados em programação orientada a objetos). E a estrutura utilizada nos exemplos não segue esse modelo por acaso.
A especificação AMQP define uma arquitetura de comandos que de fato se baseia na estrutura mostrada nos exemplos. A idéia foi facilitar o design do protocolo uma vez que Middlewares por si só já complicados o suficiente.

O mais interessante é que se estudarmos a API do Rabbitmq (que é uma implementação da especificação AMQP), poderemos notar que a sintaxe dos métodos são bem parecidos com o que é utilizado em nível mais baixo pelo protocolo.

Abaixo um exemplo de código escrito em java que envia mensagens para uma fila:

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
        // until the current received message won't processed
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

E agora um exemplo de código que consome as mensagens enviadas para a fila "hallo":

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
                System.out.printf(" \t[X] starting processing task: {{ "%s" }}\n", task);
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

Essa foi mais uma edição do diário de bordo do devadventures.

Nesse post a idéia não é entrar nos detalhes da implementação de um Publisher e um Consumer. O exemplo de código acima foi adicionado nesse post apenas para exemplificar a comunicação com um broker AMQP.

No próximo post irei colocar um pouco do que estudei do RabbitMQ utilizando o client java fornecido no próprio site do RabbitMQ.
