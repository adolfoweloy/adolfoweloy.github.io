---
layout: post
title:  "Pensando em programação funcional"
date:   2017-01-15 12:44:00 -0300
lang: pt
ref: functional_java
comments: true
categories: functional java
description: algumas idéias sobre programação funcional com Java8
---

Hoje em dia programação funcional já não é mais novidade pra ninguém. Muita gente vem falando sobre o assunto seja por moda ou por real necessidade. A questão é que depois que tive que trabalhar com máquina de estados que lançam eventos que podem ser capturados por *listeners* distribuídos em partes distintas do código, percebi que o alto nível de desacoplamento as vezes pode trazer alguns problemas. Na minha opinião, quando muita coisa começa a ser escrita utilizando-se basicamente uma implementação do padrão **Observer**, fica difícil encontrar as coisas no sistema (por mais que se utilize os melhores nomes possíveis e os pacotes estejam bem organizados). Sabendo que alguns frameworks de programação reativa ajudam a melhorar a legibilidade de um sistema baseado em eventos, acabei me interessando por programação reativa funcional (ou FRP - functional reactive programming).

Mas o título do post não diz respeito a programação funcional? Por que estou está falando de programação reativa funcional? Bom, antes de mais nada vamos dar uma olhada no que diz a wikipedia a respeito de [Functional reactive programming](https://en.wikipedia.org/wiki/Functional_reactive_programming):

> FRP is a programming paradigm for reactive programming (asynchronous dataflow programming) using the building blocks of functional programming.

Traduzindo o trecho acima, podemos dizer que programação reativa é um paradigma que faz uso dos blocos de construção da programação funcional. Não gostei muito dessa tradução mas a questão é que o que é chamado de *building blocks* não são nada mais que as funções de combinação `filter`, `map` e `fold` (fold é a base do reduce conforme veremos adiante). Agora já dá pra ter uma idéia da importância de se aprender as bases da programação funcional antes de começar a usar um framework reativo. Essa é então uma das razões para se aprender programação funcional.

Outras razões para se aprender Programação funcional, incluem resolver problemas de concorrência e melhorar o design de estruturas de dados e expressividade de um código escrito utilizando linguagem orientada a objetos.

Mas programação funcional não exclui a orientação a objetos? Esse é o ponto! É possível utilizar muitos dos conceitos da programação funcional quando estiver desenvolvendo algo com uma linguagem orientada a objetos. Além disso, algumas pessoas já tem se questionado se realmente precisamos modelar tudo como objetos em nossos sistemas. Ao se trabalhar com grandes volumes de dados (Big Data), será que ter um grafo complexo de objetos vai ajudar? Existem muitos cenários onde os dados recuperados de um banco de dados podem simplesmente ser representados como uma coleção de dados que podem ser manipulados utilizando programação funcional. Portanto, temos mais um fator motivacional em direção à programação funcional que é ter a consciência de que OOP não é a solução para todos os problemas.

# O que é programação funcional?

Apesar de essa pergunta também já ter sido respondida por muita gente, vou colocar aqui o que penso sobre o que é programação funcional para contextualizar o assunto aqui no post! Além do mais, acho difícil encontrar uma explicação que seja simples e que vá direto ao ponto.

Programação funcional tenta trazer conceitos básicos de funções matemáticas para uma linguagem de programação sendo que a função é o elemento básico da programação funcional. Assim como na matemática, uma função deve sempre devolver o mesmo resultado quando for passado o mesmo argumento. Através delas podemos combinar funções básicas que nos levam a soluções para problemas complexos.

Linguagens como Java, representam as funções através dos métodos, porém a diferença é que os métodos executam no contexto do objeto ao qual eles pertencem. Uma função é uma unidade independente que não precisa executar no contexto de um objeto. Teoricamente todo método de uma linguagem orientada a objetos recebe como parâmetro, uma referencia para o próprio objeto. É assim que podemos alterar valores do objeto acessando os atributos através da referência `this` no Java. Em python por exemplo, isso é mais explícito, pois um método precisa declarar que recebe a referência para o objeto através do argumento `self` conforme mostrado abaixo:

```python
class User:
    def __init__(self, firstName, lastName):
        self.firstName = firstName
        self.lastName = lastName

    def name(self):
        return self.firstName + ' ' + self.lastName
```

# Principais características
Devido a natureza das funções matemáticas, algumas características são incorporadas na forma de se programar usando o paradigma funcional conforme explicado a seguir.

## Imutabilidade
Assim como as funções matemáticas não alteram os valores de seus argumentos, as funções que escrevemos em nossos programas também não devem modificar os valores dos parâmetros. A imutabilidade é a chave para evitar problemas de concorrência em ambientes multi-thread.

## Sem efeitos colaterais
As funções não devem gerar efeitos colaterais pelo sistema evitando com isso bugs difíceis de se encontrar. Para evitar a geração de efeitos a idéia é não alterar o estado de outros objetos. Em uma linguagem orientada a objeto, isso também pode ser atingido quando um método não altera o estado do objeto a qual pertence.

## Declarativo ao invés de imperativo
Ao invés de expressar a intenção do código como uma sequência de passos seguidos um após o outro (modo imperativo), quando usamos programação funcional escrevemos nosso código de maneira declarativa. Uma função que calcula o fatorial de um número é um bom exemplo de comparação. Vejamos abaixo um exemplo de código interativo para calcular o valor do fatorial de um número inteiro. O exemplo a seguir mostra como uma sequência de instruções são ordenadas para que sejam executadas uma após a outra:

```java
public static long fact(long n) {
    long result = 1;
    if (n == 0) return 1;

    for (int i = 1; i <= n; i++) {
        result *= i;
    }

    return result;
}
```

Agora utilizando recursividade, podemos expressar o código de maneira declarativa:

```java
public static long fact(long n) {
    if (n <= 1) return 1;
    return fact(n - 1) * n;
}
```

Tentar entender o código acima como uma sequência de passos é totalmente inviável. Um código deve ser interpretado como uma regra expressa de forma declarativa. Fica claro entender que o fatorial de um número é o produto de todos os elementos menores ou iguais a `n`.

## Lazy evaluation
Em linguagens que dão suporte a lazy evaluation, um código escrito de forma declarativa pode ser interpretado sem a necessidade de executar partes do código em um momento inicial. Para exemplificar, vejamos o código abaixo que retorna uma sequência lazy que representa os números naturais:

```java
private LazySeq<Integer> naturals(int from) {
    return LazySeq.cons(from, () -> naturals(from + 1));
}
```

A expressão `() -> naturals(from + 1)` não vai ser executada quando chamarmos o método `naturals`. Tal **função** será executada apenas quando for necessário. Segue abaixo um exemplo de utilização da sequência lazy de modo que os métodos `map` e `take` não iteram sobre todos os números naturais (se fizesse isso também, logo estourariamos a pilha de execução do método):

```java
LazySeq<Integer> first10Doubles = naturals(2).map(n -> n * 2).take(10);
```

Um artigo muito legal sobre esse assunto e que utiliza Java pode ser acessado em [Lazy sequences implementation for Java 8](https://dzone.com/articles/lazy-sequences-implementation). 


# Problemas que a programação funcional ajuda a resolver
- Race conditions: problema de concorrência onde threads diferentes acessam dados compartilhados. 

- Manutenção: se imaginarmos que uma determinada função hora devolve um valor para uma dada entrada e hora devolve um valor diferente para uma mesma entrada, como vamos ter certeza sobre o funcionamento das partes do sistema que usam tal função? 
 
# Estruturas de dados utilizando programação funcional

O interessante de tudo é que programação funcional não se trata apenas de criar funções que não alteram o estado de objetos e variáveis. A questão é que para que tudo isso funcione é preciso pensar em estruturas de dados que são imutáveis e que deem suporte para a programação funcional. Antes de criar nossa primeira estrutura de dados vamos antes buscar a definição dos seguintes tipos de dados: tipos de dados algébricos, tipos de dados abstratos e concretos. É importante entendermos tais definições pois ganhar vocabulário sobre o assunto que estamos estudando nos ajuda a comunicar melhor e consequentemente representar melhor um conceito.

## Tipos de dados concretos
São os tipos mais básicos usados para compor tipos mais complexos como os tipos de dados algébricos e abstratos.

## Tipos de dados algébricos
Os tipos de dados algébricos (pensando de maneira bem abstrata) são tipos que podem ser criados a partir de operações de soma e multiplicação. Porém, não vamos pensar aqui nas operações utilizadas na aritmética. Nesse caso, produto e soma estão mais relacionados com o formalismo utilizado para se definir conjuntos. Podemos criar um tipo a partir da composição de uma classe com alguns atributos. Em python, podemos criar uma tupla que é composta pelos elementos que são definidos por tipos mais básicos. Esses tipos são conhecidos como *product types*. Já os tipos definidos através da operação de soma, também chamados de *sum types* podem ser emulados em Java através do mecanismo de herança onde agregamos atributos e comportamentos de outras classes em um subtipo. *Sum Types* podem ser representados através de estruturas Lazy sendo definidas por funções que não são executadas imediatamente.

Um exemplo de utilização de tipo algébrico em Java, pode ser dado através da seguinte declaração:

```java
Optional<String> name;
```

Através do `Optional` podemos trabalhar com um tipo de dado bem definido que pode conter um valor ou ser vazio. Ao trabalhar diretamente com uma `String` em Java, o programador acaba tendo que estar ciente que uma variável realmente conter uma cadeia de caracteres ou um valor nulo. O problema do valor nulo aqui é que o mesmo não se trata de uma `String`, pois é como se fosse um outro tipo definido pela linguagem. **Observação importante:** não estou dizendo aqui para sair usando `Optional` em todo lugar, o que particularmente acho que pode ser ruim em alguns cenários. Acredito que um bom exemplo de utilização do `Optional` é para métodos que podem devolver um valor ou não como um método `Optional<User> findByName(String name);`.

Listas também são exemplos de tipos de dados algébricos que podem ser compostas por outros tipos de dados bem definidos.

Para mais detalhes sobre tipos de dados algébricos, sugiro dar uma olhada no post do blog [Probably Done Before](http://merrigrove.blogspot.com.br/2011/12/another-introduction-to-algebraic-data.html) pois foi o lugar onde encontrei uma explicação mais fácil de ser entendida por programadores que não possuem uma forte inclinação para a matemática.

## Tipos de dados abstratos

Os tipos de dados abstratos podem ser entendidos como tipos mais genéricos que deixam algumas partes de dados indefinidos conforme pode ser visto no link sobre Abstract Data Type da [wiki.haskell](https://wiki.haskell.org/Abstract_data_type). Pensando em orientação a objetos, os tipos abstratos não impõem limite nos possíveis subtipos.

O uso de generics do Java por exemplo permite definir tipos abstratos de dados pois permite deixar indefinidos alguns tipos utilizados por uma classe.

# Chega de teoria

Agora vamos para um pouco de prática através de exemplos que vão mostrar a beleza da programação funcional.

## Lista encadeada recursiva

Irei utilizar aqui, os exemplos que estudei no livro [Functional Programming For Java Developers](https://www.amazon.com/Functional-Programming-Java-Developers-Concurrency/dp/1449311032). O exemplo mostra a declaração de uma lista imutável que utiliza o conceito de listas encadeadas para compor uma coleção de elementos armazenados em sequência. Para isso vamos montar uma estrutura recursiva onde cada elemento da lista possui um valor (`head`) e um atributo que aponta para uma outra lista. Perceba que estamos falando aqui de uma estrutura recursiva com elementos finitos. Como os elementos são finitos não iremos trabalhar de maneira Lazy e portanto precisamos definir um tipo básico que determinará o final da lista. Mas como definir esse ponto de parada para a lista? Para isso, vamos usar uma lista vazia. Usando uma lista vazia podemos navegar nos elementos da lista (lembrando que cada elemento aponta para uma outra lista) até que seja atingida uma lista vazia. Com uma lista vazia não há mais por onde navegar. A figura a seguir dá uma idéia de como vamos representar essa lista:

![Lista recursiva]({{ "/assets/recursive-list.png" | absolute_url }})

Precisamos representar a lista vazia e a lista que contém outra lista, por isso foram criadas duas classes `EmptyRecursiveList` e `NonEmptyRecursiveList` ambas implemtentando uma interface comum chamada `RecursiveList` conforme mostrado nas listagens a seguir:

### Interface __RecursiveList__

```java
public interface RecursiveList<T> {

    T head();

    RecursiveList<T> tail();

    boolean isEmpty();

}
```

### Classe __EmptyRecursiveList__
```java
public class EmptyRecursiveList<T> implements RecursiveList<T> {

    public EmptyRecursiveList() {
    }

    public T head() {
        throw new RuntimeException("the list is empty");
    }

    public RecursiveList<T> tail() {
        throw new RuntimeException("the list is empty");
    }

    public boolean isEmpty() {
        return true;
    }

}
```

### e a classe __NonEmptyRecursiveList__
```java
 public class NonEmptyRecursiveList<T> implements RecursiveList<T> {

    private final T head;

    private final RecursiveList<T> tail;

    public NonEmptyRecursiveList(T head, RecursiveList<T> tail) {
        super();
        this.head = head;
        this.tail = tail;
    }

    @Override
    public T head() {
        return head;
    }

    @Override
    public RecursiveList<T> tail() {
        return tail;
    }

    @Override
    public boolean isEmpty() {
        return false;
    }

}
```

Ok, agora se precisarmos criar uma lista usando essa estrutura precisaremos fazer algo assim:

```java
RecursiveList<Integer> list =
    new NonEmptyRecursiveList<Integer>(1,
        new NonEmptyRecursiveList<>(2,
            new EmptyRecursiveList<>()));

```

Parece um pouco confuso não é mesmo? Vamos então criar uma classe responsável por manipular nossa lista recursiva. Assim ela também pode ter um `factory method` para ajudar a deixar esse código um pouco mais limpo:

```java
public class ListModule {

    private static final RecursiveList<? extends Object> EMPTY 
        = new EmptyRecursiveList<Object>();

    public static <T> RecursiveList<T> list(T head, RecursiveList<T> tail) {
        return new NonEmptyRecursiveList<T>(head, tail);
    }

    @SuppressWarnings("unchecked")
    public static <T> RecursiveList<T> empty() {
        return (RecursiveList<T>) ListModule.EMPTY;
    }

}
```
E agora para criar a lista fica um pouco mais interessante:

```java
ListModule.list(1, ListModule.list(2, ListModule.empty()));
```

Para melhorar um pouco mais, vamos utilizar `import static` para a classe `ListModule` e agora o código para criação da lista fica assim:

```java
list(1, list(2, empty()));
```

Essa lista recursiva ainda não permite que você recupere um elemento ou que faça uma iteração. Dada essas necessidades, vamos começar a escrever código funcional.

Vamos começar criando um método `forEach` para que possamos iterar sobre os elementos da lista. Esse método será adicionado na interface `RecursiveList`.

```java
public interface RecursiveList<T> {

    T head();

    RecursiveList<T> tail();

    boolean isEmpty();

    void forEach(Consumer<T> consumer);
}
```

Se fossemos criar um forEach de forma interativa (e imperativa) poderiamos usar um `for` e para cada iteração executar o método da interface funcional `Consumer`. Com isso a lógica imperativa estaria apenas sendo encapsulada. Ao invés disso, veja como o código fica simples utilizando apenas recursão:

```java
@Override
public void forEach(Consumer<T> consumer) {
    consumer.accept(head());
    tail().forEach(consumer);
}
```

Essa é a implementação do método `forEach` da classe `NonEmptyRecursiveList`. Para a implementação da lista vazia não é preciso iterar sobre valor algum conforme mostrado abaixo:

```java
@Override
public void forEach(Consumer<T> consumer) {
}
```

E agora para utilizar o `forEach`, passamos um **lambda** disponível no Java 8. Em versões anteriores do Java, seria preciso passar uma classe anônima que implementaria alguma interface parecida com a interface **Consumer**:

```java
RecursiveList<Integer> nums = list(1, list(2, empty()));
nums.forEach(item -> System.out.println(item));
```

E como vamos obter um valor específico da lista? Podemos criar um método `filter` que busca os elementos de acordo com um predicado que informamos como parâmetro.

```java
@Override
public RecursiveList<T> filter(Function<T, Boolean> f) {
    if (f.apply(head())) {
        return list(head(), tail().filter(f));
    }
    return tail().filter(f);
}
```

Repare que não existe um `for` para criarmos o filtro. A busca é realizada de maneira recursiva criando uma nova lista com os elementos filtrados pelo predicado passado como parâmetro. A implementação de filtro mostrada acima deve ser adicionada na classe `NonEmptyRecursiveList` e o método abaixo na classe que representa a lista vazia:

```java
@Override
public RecursiveList<T> filter(Function<T, Boolean> f) {
    return empty();
}
```

E agora vamos direto a um exemplo de utilização do filtro que recupera apenas números pares:

```java
RecursiveList<Integer> even = nums.filter((item) -> item % 2 == 0);
```

Apesar de termos uma lista filtrada ainda queremos recuperar um elemento específico. Uma vez que realizamos o filtro podemos por exemplo solicitar o primeiro elemento encontrado na lista filtrada utilizando algo parecido com:

```java
Integer num = even.findFirst().orElse(0);
```

Uma vez que temos uma lista filtrada podemos invocar o método `findFirst` em uma lista vazia. Portanto, o valor que vamos obter da lista é opcional, ou seja, ele pode não existir na lista. No caso acima, caso não for encontrado um elemento par, vamos retornar 0.

Para esse método funcionar vamos adicionar a seguinte declaração na interface `RecursiveList`:

```java
Optional<T> findFirst();
```

E a seguir a implementação para a lista não vazia e a lista vazia respectivamente:

```java
@Override
public Optional<T> findFirst() {
    return Optional.of(head());
}
```

```java
@Override
public Optional<T> findFirst() {
    return Optional.ofNullable(null);
}
```

Outro método interessante que podemos adicionar em nossa lista é o `map` que permite derivar uma nova lista de outro tipo de dados. Vejamos a implementação para a lista não vazia:

```java
@Override
public <U> RecursiveList<U> map(Function<T, U> f) {
    return list(f.apply(head()), tail().map(f));
}
```

Para a lista vazia basta devolvermos outra lista vazia conforme mostrado abaixo:

```java
@Override
public <U> RecursiveList<U> map(Function<T, U> f) {
    return empty();
}
```

Se estiver escrevendo código de acordo com os exemplos apresentados aqui, não esqueça de declarar o método na interface. Caso tenha algum problema ao executar qualquer código aqui não se preocupe, pois todos os exemplos estão disponíveis no GitHub através do link [funcjava](https://github.com/adolfoweloy/funcjava)

Como exemplo de uso do map, vamos imprimir no console o dobro dos valores de uma lista:

```java
RecursiveList<Integer> numbers = list(1, list(2, list(3, empty())));

numbers
    .map((item) -> item * 2)
    .forEach(item -> System.out.println(item));
```

Temos mais um método importante na programação funcional que é o `fold`. O método fold é a base para um outro método disponível na maioria das linguagens que é o `reduce`. Mas por que `fold`? Fica muito fácil entender o propósito do `fold` quando pensamos na palavra ao pé da letra. Quando executamos o método fold é como se estivessemos dobrando uma lista conforme mostrado na imagem abaixo, até chegar em um único valor:

![Folding]({{ "/assets/fold.png" | absolute_url }})

É interessante perceber a relação do fold com o reduce. Fomos dobrando a lista até reduzir a mesma a um único valor. Abaixo segue o código completo da lista não vazia e logo em seguida o código completo da lista vazia contendo os métodos `foldLeft` e `foldRight`.

```java
public class NonEmptyRecursiveList<T> implements RecursiveList<T> {

    private final T head;

    private final RecursiveList<T> tail;

    public NonEmptyRecursiveList(T head, RecursiveList<T> tail) {
        super();
        this.head = head;
        this.tail = tail;
    }

    @Override
    public T head() {
        return head;
    }

    @Override
    public RecursiveList<T> tail() {
        return tail;
    }

    @Override
    public boolean isEmpty() {
        return false;
    }

    @Override
    public void forEach(Consumer<T> consumer) {
        consumer.accept(head());
        tail().forEach(consumer);
    }

    @Override
    public RecursiveList<T> filter(Function<T, Boolean> f) {
        if (f.apply(head())) {
            return list(head(), tail().filter(f));
        }
        return tail().filter(f);
    }

    @Override
    public Optional<T> findFirst() {
        return Optional.of(head());
    }

    @Override
    public <U> RecursiveList<U> map(Function<T, U> f) {
        return list(f.apply(head()), tail().map(f));
    }

    @Override
    public <T2> T2 foldLeft(T2 seed, BiFunction<T2, T, T2> f) {
        return tail().foldLeft(f.apply(seed, head()), f);
    }

    @Override
    public <T2> T2 foldRight(T2 seed, BiFunction<T, T2, T2> f) {
        return f.apply(head(), tail().foldRight(seed, f));
    }

}
```

```java
public class EmptyRecursiveList<T> implements RecursiveList<T> {

    public EmptyRecursiveList() {
    }

    public T head() {
        throw new RuntimeException("the list is empty");
    }

    public RecursiveList<T> tail() {
        throw new RuntimeException("the list is empty");
    }

    public boolean isEmpty() {
        return true;
    }

    @Override
    public void forEach(Consumer<T> consumer) {
    }

    @Override
    public RecursiveList<T> filter(Function<T, Boolean> f) {
        return empty();
    }

    @Override
    public Optional<T> findFirst() {
        return Optional.ofNullable(null);
    }

    @Override
    public <U> RecursiveList<U> map(Function<T, U> f) {
        return empty();
    }

    @Override
    public <T2> T2 foldLeft(T2 seed, BiFunction<T2, T, T2> f) {
        return seed;
    }

    @Override
    public <T2> T2 foldRight(T2 seed, BiFunction<T, T2, T2> f) {
        return seed;
    }

}
```

Agora, bastanto ter uma lista, podemos executar operações distintas sobre ela utilizando as funções de básicas de combinação que ajudam a criar código flexível o suficiente para mudar de acordo com uma regra de negócio específica.

Apenas para exemplificar o uso do `fold`, se liga no código que recupera os números pares de uma lista, dobra o valor de cada um e soma tudo no final:

```java
RecursiveList<Integer> values = list(1, list(2, list(3, list(4, empty()))));

Integer total = values
    .filter((n) -> n % 2 == 0)
    .map((n) -> n * 2)
    .foldRight(0, (n, m) -> n + m);
```

# Conclusão

É importante observar que a maioria das coisas que implementamos para nossa lista já está disponível no Java 8 através da API de `streams`. Porém existe uma diferença importante em tudo isso. As funções de combinação que implementamos para nossa lista não são `Lazy`, ou seja, elas executam toda a lógica sobre a lista sempre que são invocadas. Então se encadearmos um `map` e um `filter` tais operações irão executar imediatamente quando invocadas. A API de streams do Java 8 por sua vez já suporta o conceito de `Lazy` o que pode trazer benefícios de performance dependendo da situação. Caso queira saber mais sobre Laziness no Java 8, recomendo a leitura do seguinte post [Java 8 Streams API - Laziness and Performance Optimization](http://java.amitph.com/2014/01/java-8-streams-api-laziness.html#.WHuBHLGZORs).

É isso aí. Esse post é mais um resultado de alguns dias de estudo sobre programação funcional. Qualquer problema, pode mandar uma sugestão através de um comentário que eu altero o post se for preciso.
