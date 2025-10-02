---
layout: post
title: "Dissecando Respostas Assíncronas em Aplicações Java Baseadas em Servlets"
date: 2025-10-01 11:41:00 +1100
ref: async_servlets
comments: true
lang: pt
categories: java async reactive
description: Nos bastidores de APIs assíncronas em Java
---

No dia a dia, se você trabalha com aplicações Java, as chances são grandes de que sua aplicação forneça algumas APIs REST construídas com algum framework que fornece alguma abstração em cima da API de Servlets. Embora as aplicações baseadas em Servlets estejam presentes em muitas aplicações ou serviços web baseados em Java, é fácil se perder na terminologia, especialmente quando se trata de processamento assíncrono e I/O não bloqueante.

Todo desenvolvedor Java backend sabe o que é um Servlet e espero que também saiba como ele funciona. É uma tecnologia fundamental no mundo Java e, como tudo, também é fácil de se dar por garantido. Assim como um [emulador de terminal](https://adolfoeloy.com/terminal/linux/xterm/2025/05/03/terminal-emulator.en.html) que investiguei em um post anterior, sempre há mais a ser descoberto por trás dos bastidores.

Movido pela curiosidade, tomei a decisão de mergulhar fundo no que está por trás do suporte assíncrono do Servlet até as chamadas de sistema do Linux.

## O que você encontrará neste post

Depois de minha pequena aventura sobre esse tópico, consegui entender melhor como o Tomcat lida com ~10.000 conexões simultâneas com um número menor de threads usando sockets com I/O não bloqueante e como o processamento assíncrono funciona nos bastidores, tanto internamente no Tomcat quanto em nível de chamadas de sistema.

Pessoalmente, eu curto muito investigar o que está por trás de coisas simples que usamos todos os dias. Ontem foi sobre emuladores de terminal, hoje é sobre Servlets, amanhã pode ser o que está por trás do event-loop do NodeJS (spoiler: eu já brinquei com [libuv](https://docs.libuv.org/en/v1.x/) - a biblioteca que fundamenta o loop de eventos do Node). Para o uso cotidiano de Servlets, deveria ser um tópico chato, mas encontrar o pequeno universo por trás disso é algo diferente e fascinante (pelo menos para mim).

![Async room]({{ "/assets/async/asyncnonblocking.png" | absolute_url }})

## Um pouco de história

Uma das coisas que gosto nesses deep-dives, é que através de códigos-fonte e APIs de baixo nível, as chances são grandes de que eu também possa encontrar alguma história interessante por trás da coisa sendo investigada. Para Servlets em particular, por que o suporte assíncrono foi adicionado por volta de 2009? Quais eram os problemas daquela época? O que estava acontecendo por volta de 2009 e antes? Quais eram as alternativas para a solução dada que podemos usar até hoje? Isso é pura arqueologia da software, e para mim isso é tão legal quanto ler e escrever código.

Para entender o contexto sobre como o processamento assíncrono surgiu em Servlets, viajei de volta no tempo lá atrás em 2006, quando a comunidade estava empolgada usando AJAX a todo vapor e todos estavam entusiasmados com a [Web 2.0](https://pt.wikipedia.org/wiki/Web_2.0). Antes da especificação final do Servlet 3.0 trazer o suporte assíncrono para o backend, muitas abordagens diferentes foram adotadas para resolver os mesmos ou problemas similares, que o Servlet 3.0 resolveu. Alguns exemplos são o [projeto Eclipse Jetty](https://jetty.org/index.html) e Comet (um estilo de streaming de dados orientado a eventos e push do servidor, cujo nome foi cunhado por Alex Russell em um post publicado em 2006 chamado [Commet: Low Latency Data for the Browser](https://infrequently.org/2006/03/comet-low-latency-data-for-the-browser/)).


### Os desafios da época

![Resource starvation]({{ "/assets/async/thread-starvation.png" | absolute_url }})

Todas aquelas tecnologias disponíveis na época para criar aplicações web mais interativas, como AJAX, também trouxe um novo conjunto de desafios para o backend. Um único cliente poderia abrir várias conexões com o servidor, e várias requisições poderiam acontecer simultaneamente, aumentando a carga no servidor. Imagine o quanto a carga aumentaria em uma página que suportava 1000 requisições simultâneas e, de repente, com mais interatividade, 10 requisições adicionais para cada cliente seriam feitas via AJAX para essa mesma página. Isso ultrapassaria o que o Tomcat poderia lidar na época, levando problemas como resource starvation e clientes insatisfeitos.

Uma maneira melhor e padronizada de lidar com processamento lento e conexões de longa duração era claramente necessária. Para começar a superar esses desafios, uma das mudanças propostas para o Servlet 3.0 foi sugerida como "Suporte Assíncrono e Comet" (veja [JSR-315](https://jcp.org/en/jsr/detail?id=315)). O foco principal dessa proposta era adicionar suporte ao processamento assíncrono de requisições, o que por si só deveria ajudar com o problema de resource starvation, bem como alavancar as aplicações no estilo Comet.

O suporte a I/O não bloqueante também foi considerado e aprimorado com o Servlet 3.1, permitindo leituras e gravações não bloqueantes de requisições e respostas, respectivamente. No entanto, isso ainda não seria a solução totalmente não bloqueante que conhecemos hoje, com por exemplo, Spring WebFlux com Netty, ou o que é fornecido por outras stacks, como NodeJs. Caso você queira saber mais sobre a história e ver alternativas propostas, como a do autor do Jetty, deixei algumas boas referências no final deste post.

### Abstrações

Quando aprendi sobre o suporte assíncrono do Servlet há muito tempo, não o usei imediatamente. Quando realmente precisei usar o suporte assíncrono do Servlet, estava usando uma abstração sobre os Servlets fornecida pelo Spring Web MVC (para encurtar, chamarei apenas de Spring Web).

Uma das perguntas que eu tinha na época era: se a resposta é deixada em aberto e é processada por outra thread que não a do worker, como o Tomcat sabe como enviar a requisição para a conexão correta (ou seja, para o cliente correto)? Eu sei que não é nada produtivo querer entender a fundo todas abstrações que uso no dia a dia, então, como a maioria das pessoas, aceitei que o Tomcat apenas funciona como esperado e segui em frente. No entanto, a curiosidade ainda estava lá.

Certo, agora que o Spring é um framework essencial para projetos Java, será que ainda vale a pena entender os detalhes minuciosos dos Servlets e do Tomcat? Eu acho que sim. Independentemente de usar o Spring Web ou diretamente os Servlets (embora eu espere que a maioria dos projetos esteja usando alguma abstração sobre os Servlets), os princípios permanecem os mesmos e uma compreensão sólida sobre isso ajuda a esclarecer mal-entendidos comuns sobre suporte assíncrono vs I/O não bloqueante. Além disso, esse modelo também é replicado em certa medida por outras stacks fora do mundo Java.

<div style="border: 1px solid #b3afd3ff; padding: 0.5em; background-color: #edebfaff; margin-bottom: 0.5em">
O Spring Web fornece <code>DeferredResult</code>, <code>Callable</code> e <code>WebAsyncTask</code> como tipos de retorno de controlador para suporte assíncrono. Ao retornar esses tipos, o Spring Web usa o suporte assíncrono do Servlet 3.0 para que a resposta seja retornada de uma thread separada (funciona um pouco diferente do exemplo nesta página, pois depende de mecanismos de despacho). Ele também permite o <a href="https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-async.html#mvc-ann-async-reactive-types">retorno de tipos reativos</a>, embora todos sejam adaptados para um tipo de resultado assíncrono equivalente (por exemplo, um <code>DeferredResult</code> para resultados de valor único).
</div>


## Chega de história, vamos ao que interessa

Para navegar por essa exploração, apresento tudo em um estilo _top-down_, seguindo o diagrama mostrado abaixo. Aqui começo explicando o que `HelloServlet` está fazendo, depois começo a mergulhar nos detalhes internos do Tomcat até chegar nas chamadas do sistema operacional via APIs e system calls. No meu caso, minha exploração começou com o Servlet enquanto olhava os logs do `strace` primeiro (mais sobre isso logo mais).

![Layers]({{ "/assets/async/layers-request.png" | absolute_url }})

## HelloServlet

O propósito do servlet aqui é muito simples:
1. Lidar com uma requisição para o endpoint `/hello`
2. Iniciar um contexto assíncrono
3. Transferir o processamento da requisição para uma nova thread chamada `custom-thread`. Isso é suficiente para liberar a thread do worker do Tomcat
4. Finalmente, enviar a resposta para o cliente a partir da `custom-thread` -- uma tarefa realizada por `BackgroundTask` -- responsável por completar o processamento da requisição.

O código-fonte completo para este simples projeto Servlet está disponível como outro repositório no meu GitHub: [asyncservlet](https://github.com/adolfoweloy/asyncservlet). Eu recomendo baixá-lo caso você queira executar alguns dos comandos que compartilho neste post.

```java
@WebServlet(value="/hello", name="helloServlet", asyncSupported = true)
public class HelloServlet extends HttpServlet {
    private final static String CUSTOM_THREAD_NAME = "custom-thread";

    @Override
    public void service(HttpServletRequest req, HttpServletResponse res) {
        System.out.println("Handling the request in thread: " 
            + Thread.currentThread().getName());

        AsyncContext asyncContext = req.startAsync();
        asyncContext.setTimeout(10_000);

        new Thread(
                new BackgroundTask(asyncContext),
                CUSTOM_THREAD_NAME
        ).start();
    }
}
```

A tarefa em background simplesmente dorme por quatro segundos e é responsável por escrever na resposta e completar o processamento assíncrono. O trecho a seguir mostra a tarefa em background.

```java
private record BackgroundTask(AsyncContext asyncContext) implements Runnable {

    @Override
        public void run() {
            try {
                System.out.println("Processing async in thread: "
                        + Thread.currentThread().getName());
                Thread.sleep(4_000);
                var res = (HttpServletResponse) asyncContext.getResponse();
                res.setContentType("application/json");
                writeResponse("Hello, World", 200);
            } catch (InterruptedException e) {
                writeResponse("db error", 500);
            } finally {
                asyncContext.complete();
            }
        }

        private void writeResponse(String message, int status) {
            var res = (HttpServletResponse) asyncContext.getResponse();
            try {
                res.setStatus(status);
                res.getWriter().println(message);
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
}
```

### Rodando o projeto de exemplo

Se você quiser executar o código, para ter uma experiência tranquila, recomendo que você instale o [SdkMan](https://sdkman.io/) caso ainda não o tenha. A maioria dos meus projetos Java vem com um `.sdkmanrc` com a versão do Java sendo usada para o projeto e, para usar a versão correta a partir do terminal, tudo o que é necessário é executar `sdk env` (o IntelliJ reconhece isso imediatamente, escolhendo a JVM correta para o projeto -- é claro que é esperado que você tenha a JVM instalada).

Uma vez que você tenha o Java 21 instalado, você pode simplesmente executar os seguintes comandos:

```bash
mvn clean package
java -jar target/asyncservlet-1.0-SNAPSHOT-shaded.jar
```

Eu também estou usando `wrk` como uma ferramenta de benchmark para comparar o processamento assíncrono vs síncrono. Aqui está a [página do projeto no GH](https://github.com/wg/wrk). Com `wrk` você pode enviar requisições concorrentes usando múltiplas conexões como no snippet abaixo. Neste exemplo estou criando 10 conexões e 20 threads com uma duração de 10 segundos. Isso deve permitir ver quantas requisições podem ser processadas dentro desse período de 10 segundos dadas as condições definidas nos parâmetros.

```bash
wrk --timeout 10s --latency -t20 -c10 -d10s http://localhost:8080/hello
```

## Tomcat internals

Nessa seção eu examino dois aspectos separados do Tomcat: os componentes de _rede de baixo nível_ e o __pipeline de requisição HTTP__. O primeiro gerencia a criação de sockets, polling e event dispatching, enquanto o segundo realmente processa a requisição, analisando-a, aplicando filtros, invocando o método `service` do Servlet e processando a resposta.

Antes de mergulhar no código ou nos logs, vamos examinar o nível de rede começando com os atores que colaboram para aceitar conexões e lidar com a requisição.

### Os atores por trás dos bastidores

Eu acho que a maneira mais fácil de realmente ver os atores em ação, é enviando requisições para o endpoint `/hello` e monitorando as threads do Tomcat usando uma ferramenta como [VisualVM](https://visualvm.github.io/) ou [jconsole](https://openjdk.org/tools/svc/jconsole/).

A imagem a seguir, capturada do VisualVM enquanto processava uma requisição para o endpoint `/hello`, mostra as threads que são relevantes para os propósitos deste post.

![Tomcat threads]({{ "/assets/async/selected-threads.png" | absolute_url }})

Diretamente ao ponto, os atores são o Acceptor, o Poller, alguns Workers e as Threads Customizadas. Aqui está a descrição de cada um deles:

- __Acceptor__: aceita novas conexões de clientes e entrega a requisição ao poller adicionando um `PollerEvent` à fila interna do `Poller`.
- __Poller__: o poller bloqueia por um segundo aguardando novos eventos na fila, lê um evento sem bloquear se houver um disponível ou é "acordado" pelo `Acceptor` quando adiciona um novo `PollerEvent` à fila do `Poller`. Sempre que há um evento a ser processado, o poller transfere o processamento para uma worker thread, também conhecida na documentação e no código-fonte do Tomcat como _container thread_. Eu continuarei chamando-a de worker thread.
- __http-nio-8080-exec-N__: esta é na verdade uma instância da __worker thread__ do Tomcat. O Tomcat começa com um pool de dez threads prontas para processar requisições recebidas. Por padrão, pode criar um máximo de 200 threads. Essas informações podem ser obtidas na documentação do Tomcat, mas também podem ser recuperadas dos atributos `ThreadPool` do JMX do Tomcat (o VisualVM não vem com a aba MBeans por padrão, então se você quiser ver isso por si mesmo, precisa instalar o plugin MBean para o VisualVM).
- __custom-thread__: esta é a thread que estou criando a partir do método `HelloServlet#service`. e eu me refiro a ela simplesmente como __custom thread__.


### Componentes de rede de baixo nível

Agora para colocar as coisas em perspectiva, os atores descritos anteriormente são todos instâncias de algumas das classes representadas no diagrama de classes __abaixo__. Eu considero `NioEndpoint` a classe principal neste diagrama porque `Poller` e `Acceptor` são todas suas classes internas. Além disso, todas elas colaboram para lidar com o trabalho de rede de baixo nível e coordenar a comunicação via threads Worker. Como uma tentativa de descrever brevemente o que está nesta imagem, essas classes colaboram para aceitar conexões, lidar com a requisição para os workers e iniciar o pipeline de requisição HTTP via `SocketProcessor`.

![Tomcat main networking classes]({{ "/assets/async/tomcat-main-classes.png" | absolute_url }})

## Linux System calls

Quando eu comecei a explorar como o Tomcat lida com requisições nos bastidores, eu queria ver como a thread customizada estava realmente enviando a resposta de volta para o cliente correto. E eu comecei examinando as chamadas de sistema do Linux usando o [strace](https://man7.org/linux/man-pages/man1/strace.1.html). Embora os logs possam ser às vezes enormes, conhecer as chamadas de sistema básicas e o que procurar nos logs pode ajudar muito a entender o que está acontecendo com um componente de software, mesmo se você não tiver o código-fonte. No meu caso, isso realmente me guiou sobre o que esperar do código-fonte do Tomcat. Caso você não saiba o que é `strace`, aqui está uma breve descrição da página de manual do `strace` em tradução livre:

> Ele intercepta e registra as chamadas de sistema feitas por um processo e os sinais que um processo recebe.

Ok, isso pode ser um pouco abstrato. O segundo parágrafo da seção de descrição é melhor e até inspirador (mais uma vez, tradução livre):

> strace é uma ferramenta útil de diagnóstico, instrução e depuração. Administradores de sistema, diagnosticians (isso eu não sei como traduzir) e solucionadores de problemas acharão isso inestimável para resolver problemas com programas para os quais o código-fonte não está prontamente disponível, já que a recompilação não é necessária para rastreamento. Estudantes, hackers e os excessivamente curiosos descobrirão que muito pode ser aprendido sobre um sistema e suas chamadas de sistema ao rastrear até mesmo programas comuns.

Eu me simpatizo com essa definição total! Especialmente quando diz "_muito pode ser aprendido sobre um sistema e suas chamadas de sistema ao rastrear até mesmo programas comuns_". É sobre isso que este post do blog se trata. Eu estava depurando um Servlet comum, algo simples aparentemente sem muita importância para muitos.

### Uma introdução às chamadas de sistema

As chamadas de sistema mais interessantes que eu mostro nos logs coletados após enviar uma requisição para `/hello` são:
- epoll_create1
- epoll_ctl
- epoll_wait
- fcntl

As outras chamadas de sistema são autoexplicativas, então não as abordarei aqui.

#### A API epoll

As chamadas `epoll_create1`, `epoll_ctl` e `epoll_wait` fazem parte da API epoll. De acordo com a [página do manual do epoll](https://linux.die.net/man/7/epoll), epoll é um mecanismo de registro de notificação de eventos de I/O. Em uma ideia talvez bem simplificada, o epoll pode ser usado para monitorar descritores de arquivos (_file descriptors_ geralmente referenciados por simplesmente `fd`). Exemplos de _file descriptors_, ou fds são pipes, sockets e dispositivos são representados como fds no Linux e você pode pensar neles como _handlers_).

Em outras palavras, você fornece ao epoll uma lista de _file descriptors_ para monitorar mudanças (por exemplo, o socket está pronto para leitura ou gravação) e espera que os eventos aconteçam. Mais uma coisa: `epoll_wait` __bloqueia__ enquanto espera por uma notificação, então aqui eu pude ver que o suporte assíncrono do Servlet é intrinsecamente bloqueante. No entanto, é claro que o Tomcat não fica sentado lá para sempre esperando por notificações. Na verdade, há um loop onde ele indiretamente (via NIO Selector) _epolla_ por um segundo. Na próxima iteração, pode ter acontecido que um evento se tornou pronto para ser processado e é quando a seguinte chamada de sistema acontece conforme o exemplo a seguir (note que o último argumento é zero -- com tempo de limite zero, o `epoll_wait` não espera para receber a notificação do evento):

```
epoll_wait(9, [{events=EPOLLIN, data={u32=11, u64=xx}}], 1024, 0) = 1 
```

O snippet a seguir é a lógica que é executada em um loop (um while true) dentro do método `Poller.run()` do Tomcat. Observe que a lógica no bloco `if` (quando a condição é verdadeira) é o que realmente mapeia para o `epoll_wait` com um tempo limite de zero segundos.

```java
if (wakeupCounter.getAndSet(-1) > 0) {
    // If we are here, means we have other stuff to do
    // Do a non-blocking select
    keyCount = selector.selectNow();
} else {
    keyCount = selector.select(selectorTimeout);
}
```

Não vou me extender sobre a lógica em torno do `wakeupCounter`, mas o resumo é que ele será usado como um mecanismo para acordar o poller para fazer uma seleção não bloqueante (`selectNow`) ou parar de esperar por um evento. Esse contador é incrementado toda vez que há um novo evento de poller iniciado pelo `Acceptor` quando uma nova requisição chega e isso pode levar a uma operação de wakeup no selector que, por sua vez, acordará o selector que está esperando com um tempo limite de um segundo (o bloco `else` do trecho anterior será acordado):

```java
// class Poller
private void addEvent(PollerEvent event) {
    events.offer(event);
    if (wakeupCounter.incrementAndGet() == 0) {
        selector.wakeup();
    }
}
```

Para saber mais sobre a chamada de sistema epoll, acho que o post do blog "[The method to epoll's madness](https://copyconstruct.medium.com/the-method-to-epolls-madness-d9d2d6378642)" é incrível. Também recomendo esta ótima visão geral sobre o assunto, escrita por Julia Evans, "[Async IO on Linux: select, poll, and epoll](https://jvns.ca/blog/2017/06/03/async-io-on-linux--select--poll--and-epoll/)".

<div style="border: 1px solid #b3afd3ff; padding: 0.5em; background-color: #edebfaff; margin-bottom: 0.5em">
O objeto selector aqui faz parte da API Java NIO, parte do que traz suporte a I/O não bloqueante no Java.
</div>

#### fcntl

A respeito disso serei breve, porque honestamente não explorei muito, mas o resumo é que ele executa operações em descritores de arquivos abertos. A razão pela qual estou mencionando isso aqui é porque há uma chamada interessante nos logs do `strace`:

```
fcntl(11, F_SETFL, O_RDWR|O_NONBLOCK) = 0
```

Esse comando está simplesmente definindo a propriedade `O_NONBLOCK` do _file descriptor_ do socket, o que significa que esse _file descriptor_ é de I/O não bloqueante. Certo, então temos suporte a I/O não bloqueante? Nem tão rápido! Essa é uma propriedade definida apenas para a conexão, o que significa que cada vez que alguém tenta ler do socket e não há nada para ler, em vez de bloquear, ele retorna uma flag `EAGAIN` e o programa continua até a próxima tentativa de leitura.

Essa chamada de sistema é feita exatamente quando uma nova conexão é aceita pelo `Acceptor` e algumas opções são definidas antes de adicioná-la ao mapa atual de `connections` e ser encapsulada pela classe `SocketWrapper` (mencionada no diagrama de classes). Essa operação é realizada pela seguinte linha do método `NioEndpoint.setSocketOptions`:

```java
socket.configureBlocking(false);
```

Quero enfatizar novamente que o mecanismo não bloqueante aqui é usado apenas na conexão. Se você tiver uma lógica que bloqueia a execução da thread customizada, essa thread ficará bloqueada, esperando pelo que quer que tenha que esperar, a menos que você use uma biblioteca não bloqueante para fazer o trabalho. Como exemplo, alguém poderia usar `WebClient` do Spring WebFlux para enviar uma requisição para outro serviço, o que seria realizado de maneira não bloqueante.

## Ligando os pontos

Agora que forneci contexto sobre as classes de rede de baixo nível do Tomcat e as chamadas de sistema relevantes, é hora de dissecar uma requisição. A primeira coisa a fazer é iniciar o serviço e enviar uma requisição para `/hello`. Ao depurar o Tomcat, iniciei o serviço com o comando `strace`, mas o rastreamento pode ser iniciado após o serviço estar rodando. Para fazer isso, basta fornecer o ID do processo (PID) do serviço com a opção `-p` do `strace`.

Iniciando o serviço com `strace`:

```bash
strace -ttt -f -o trace.log -e trace=network,desc -s 2000 \
java -jar target/asyncservlet-1.0-SNAPSHOT-shaded.jar
```

Então, tudo o que eu tive que fazer foi enviar uma requisição para `/hello`, esperar pela resposta e parar o processo para que o log não continue crescendo indefinidamente (ele cresce rápido). Aqui está um resumo do log de rastreamento com o que é relevante para essa depuração:

```
144471 5438.553 epoll_create1(EPOLL_CLOEXEC) = 9 
144523 5438.555 accept(8,  <unfinished ...>
...
144523 5440.167 <... accept resumed>{sa_family=AF_INET6, sin6_port=htons(59396), ...), sin6_scope_id=0}, [28]) = 11
144523 5440.175 fcntl(11, F_GETFL) = 0x2 (flags O_RDWR)
144523 5440.175 fcntl(11, F_SETFL, O_RDWR|O_NONBLOCK) = 0
144523 5440.177 accept(8, {sa_family=AF_INET6, sin6_port=htons(59400), ...), sin6_scope_id=0}, [28]) = 12
...
144522 5440.178 epoll_ctl(9, EPOLL_CTL_ADD, 11, {events=EPOLLIN, data={u32=11, u64=126985003073547}}) = 0
144522 5440.178 epoll_wait(9, [{events=EPOLLIN, data={u32=11, u64=126985003073547}}], 1024, 0) = 1 
144522 5440.178 epoll_ctl(9, EPOLL_CTL_DEL, 11, 0x737e05ffe514) = 0
...
144512 5440.201 read(11, "GET /hello HTTP/1.1\r\nHost: localhost:8080\r\nConnection: keep-alive...", 8192) = 695
144512 5440.218 write(1, "Handling the request in thread: http-nio-8080-exec-1\n", 43) = 43
...
144513 5440.221 write(1, "Processing async in thread: http-nio-8080-exec-2\n", 49) = 49
144513 5444.231 write(11, "HTTP/1.1 200 ...", 136) = 136
...
144522 5444.232 epoll_ctl(9, EPOLL_CTL_ADD, 11, {events=EPOLLIN, data={u32=11, u64=11}}) = 0
```

### Aceitando 

Tudo que acontece aqui são chamadas feitas na camada de rede de baixo nível do Tomcat, mas algumas delas também são o resultado do que é iniciado a partir do pipeline de requisição HTTP. A primeira linha é autoexplicativa, mostrando o (PID ou TID -- ID da Thread), um timestamp truncado e a chamada de sistema que, neste caso, está apenas criando um epoll onde o sistema pode adicionar _file descriptors_ para monitorar eventos.

A segunda linha é mais interessante e mostra quando a thread `144523` realmente bloqueia aceitando novas conexões, que é retomada quando uma requisição para `/hello` chega, levando ao `accept resumed`.

```
144523 5440.167 <... accept resumed>{<truncated>), sin6_scope_id=0}, [28]) = 11
```

Podemos assumir agora que `144523` é o `Acceptor`. Observe que, uma vez que a conexão é aceita, ela retorna um número e esse número é usado para referenciar o _file descriptor_ do socket (neste caso, `11`). Do ponto de vista do Tomcat, isso é feito pelo método `Acceptor.run()`. O trecho abaixo apresenta um trecho do método `run`:

```java
try {
    // Accept the next incoming connection from the server
    // socket
    socket = endpoint.serverSocketAccept();
} catch (Exception e) {
    // We didn't get a socket
    // commented for brevity
}
```

Lembra do `fcntl`? Aqui é onde o acceptor está pedindo ao `NioEndpoint` para definir algumas propriedades OR-ing com `O_NONBLOCK`, como explicado anteriormente. Como pode-se notar, há outra conexão aceita com o ID 12, sobre a qual não tenho certeza do que se trata (aparentemente o Chrome pode abrir conexões extras antes de enviar requisições, mas de novo, não tenho certeza sobre isso).

### Polling de eventos

Outra thread (TID `144522`) começa a registrar interesse em eventos de entrada (`EPOLLIN`) para o socket aceito número `11`.

```
144522 5440.178 epoll_ctl(9, EPOLL_CTL_ADD, 11, \
   {events=EPOLLIN, data={u32=11, u64=126985003073547}}) = 0
```

Essa chamada foi feita quando a conexão foi aceita e o `Acceptor` invocou `setSocketOptions` para o novo `SocketChannel`, que é uma chamada indireta para o método `Poller.register` mostrado abaixo.

```java
public void register(final NioSocketWrapper socketWrapper) {
    // this is what OP_REGISTER turns into.
    socketWrapper.interestOps(SelectionKey.OP_READ);
    PollerEvent pollerEvent = createPollerEvent(socketWrapper, OP_REGISTER);
    addEvent(pollerEvent);
}
```

Então, como próximo passo, ele começa a esperar por eventos (aqui estou registrando apenas a chamada `epoll_wait` que ocorreu com timeout zero, o que significa que o `selector` Java não teve a chance de ser acordado pelo `Acceptor` -- registrando novos eventos para o `Poller`).

```
144522 5440.178 epoll_wait(9, [{events=EPOLLIN, \
   data={u32=11, u64=126985003073547}}], 1024, 0) = 1 
```

Essa chamada acontece dentro do loop do `Poller` que verifica o sinal de wakeup (mesmo código mostrado anteriormente):

```java
if (wakeupCounter.getAndSet(-1) > 0) {
    // If we are here, means we have other stuff to do
    // Do a non-blocking select
    keyCount = selector.selectNow();
} else {
    keyCount = selector.select(selectorTimeout);
}
```

Mais tarde, quando o `Poller` está pronto para fazer o dispatching do processamento da requisição, ele remove o interesse por eventos de entrada no socket `11` da seguinte forma:

```
144522 5440.178 epoll_ctl(9, EPOLL_CTL_DEL, 11, 0x737e05ffe514) = 0
```

### Começa o pipeline de requisição HTTP

Então o pipeline de requisição HTTP começa e a thread de trabalho começa a ler a requisição, o que é visível na seguinte chamada de sistema (note que a primeira coluna mudou para TID `144512`):

```
144512 5440.201 read(11, "GET /hello HTTP/1.1\r\nHost: localhost:8080\r\nConnection: keep-alive...", 8192) = 695
144512 5440.218 write(1, "Handling the request in thread: http-nio-8080-exec-1\n", 43) = 43
```

A chamada de sistema write aqui está simplesmente escrevendo no console, o que eu fiz com um `System.out.println` de dentro do método `service` do Servlet.

### Finalmente, escrevendo a resposta

Sim, foi uma longa jornada. Agora, finalmente a custom thread (note o novo TID `144513`) escreve a resposta para a conexão inicialmente aceita com ID `11`. Aqui está a prova de que o mecanismo Async do Servlet realmente escreve na conexão certa.

E agora alguém pode me julgar: "Ei, vamos lá! Não é suficiente ver que o navegador recebeu a resposta para acreditar nisso?". Ok, eu entendo, mas pense nesse debug como uma prova matemática. Um famoso logicista passou anos para provar que um mais um é igual a dois, então por que eu não posso provar que o Servlet funciona olhando para o nível de chamadas de sistema do Linux?

```
144513 5440.221 write(1, "Processing async in thread: http-nio-8080-exec-2\n", 49) = 49
144513 5444.231 write(11, "HTTP/1.1 200 ...", 136) = 136
```

A propósito, se me lembro bem, o logicista de quem falei foi Bertrand Russel e a obra monumental foi a [Principia Mathematica](https://en.wikipedia.org/wiki/Principia_Mathematica), que foi na verdade o resultado do trabalho de Bertrand e Alfred North Whitehead.

## Suporte assíncrono em nível do request pipeline

Esta é uma seção curta apenas para descrever um pouco sobre por que temos que definir a requisição como assíncrona a partir do método de serviço. Novamente, isso pode parecer óbvio, mas é mesmo? Por que você não pode apenas criar a thread e iniciá-la sem realmente marcar a requisição como assíncrona? Se você tentar fazer isso, apenas passando o objeto `response` para a tarefa em segundo plano para que ela escreva na resposta quando for executada, então esse será o resultado (exceto para o `HelloServlet` em meu stack trace):

```
Exception in thread "custom-thread" java.lang.IllegalStateException: \
The response obj. has been recycled and is no longer associated w/ this facade
	at o.a.c.c.ResponseFacade.checkFacade(ResponseFacade.java:427)
	at o.a.c.c.ResponseFacade.isCommitted(ResponseFacade.java:190)
	at o.a.c.c.ResponseFacade.setContentType(ResponseFacade.java:150)
	at com.adolfoeloy.HelloServlet$BackgroundTask.run(HelloServlet.java:36)
```

Marcando a requisição como assíncrona com `request.startAsync()` vinculará o `AsyncContext` à requisição e isso será usado em todo o pipeline de requisição HTTP para garantir que a resposta seja mantida aberta para que a tarefa em segundo plano escreva a resposta quando estiver pronta para fazê-lo. Aqui está uma visão muito simplificada do que acontece quando o Servlet está sendo processado:

![Request pipeline]({{ "/assets/async/request-pipeline.png" | absolute_url }})

O ponto principal aqui que eu quero ilustrar é que quando o `HelloServlet` termina de processar, ele retorna potencialmente antes que o `BackgroundTask` seja concluído e a referência `AsyncContext` da requisição será usada pelo pipeline de requisição (ao retornar) para verificar se a requisição é assíncrona, a fim de manter a resposta aberta e controlar os status da máquina de estados de requisições assíncronas.

Quando o `BackgroundTask` conclui o processamento, o pipeline de requisição verificará e atualizará a máquina de estados de requisições assíncronas de forma apropriada e fechará os recursos após a conclusão.

## Considerações finais

Uma coisa que eu gostaria de reiterar aqui é que usar respostas assíncronas não é uma solução mágica que aumentará o throughput da sua aplicação. Se as threads customizadas também bloquearem porque estão esperando por uma longa chamada de banco de dados ser resolvida ou porque estão esperando por um serviço de terceiros lento, o pool de threads customizadas também pode se esgotar até que a fila do executor também atinja o limite e o serviço se torne simplesmente não responsivo da mesma forma que seria sem a abordagem assíncrona. Novamente, esta não é uma solução de I/O não bloqueante e não fornece backpressure.

Não menos importante, meu objetivo com este post começa com minha intenção de consolidar as coisas que aprendi enquanto fazia minha pesquisa sobre o funcionamento interno do Tomcat. Considero essa missão cumprida para os propósitos que tinha. No entanto, também acredito que esse conteúdo pode ajudar outras mentes curiosas a ter uma imagem clara sobre o que acontece quando uma requisição é processada pelo Tomcat e mais: que a confusão, infelizmente muito comum sobre async vs I/O não bloqueante seja esclarecida. Se você foi paciente para ler este post até aqui, obrigado por lê-lo e espero que tenha gostado tanto quanto eu gostei ao escrevê-lo.

# References
- [JSR 315: Java Servlet 3.0 Specification](https://jcp.org/en/jsr/detail?id=315)
- [JSR 315 Servlet 3.0 Specification Part I](https://webtide.com/jsr-315-servlet-3-0-specification-part-i/)
- [Comet low latency data for the browser](https://infrequently.org/2006/03/comet-low-latency-data-for-the-browser/)
- [Using Asynchonous Servlets for Web Push Notifications](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/async-servlet/async-servlets.html)
- [Discussion about the relationship between Non-blocking I/O support from Servlet 3.1 and the async processing of Servlet 3.0](https://download.oracle.com/javaee-archive/servlet-spec.java.net/users/2012/11/0294.html)
- [Non-blocking I/O using Servlet 3.1 example by Arun Gupta](https://web.archive.org/web/20131024234453/https://blogs.oracle.com/arungupta/entry/non_blocking_i_o_using)
- [Tenfold increase in server throughput with Servlet 3.0 asynchronous processing](https://web.archive.org/web/20120124181624/https://nurkiewicz.blogspot.com/2011/03/tenfold-increase-in-server-throughput.html)
- [Spring MVC 3.2 Preview: techniques for real-time updates](https://web.archive.org/web/20130621124909/https://blog.springsource.org/2012/05/08/spring-mvc-3-2-preview-techniques-for-real-time-updates/)
- [Spring MVC 3.2 Preview: Introducing Servlet 3, Async Support](https://spring.io/blog/2012/05/07/spring-mvc-3-2-preview-introducing-servlet-3-async-support)
- [Java NIO Selector](https://jenkov.com/tutorials/java-nio/selectors.html)
- [epoll Linux man page](https://linux.die.net/man/7/epoll)

# Footnotes
[^1]: https://akka.io/blog/reactive-programming-versus-reactive-systems
[^2]: https://projectreactor.io/docs/core/release/reference/gettingStarted.html
