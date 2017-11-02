---
layout: post
title:  "Escrevendo um livro para explicar OAuth 2.0 (Em Português)"
date:   2017-03-05 18:08:00 -0300
lang: pt
ref: livro_casa_do_codigo
comments: true
categories: livro casadocodigo oauth2
description: como foi escrever um livro técnico didático para explicar como funciona o protocolo OAuth 2.0 em detalhes
---

Hoje, quero compartilhar como foi minha jornada durante a escrita do meu primeiro livro, que explica em detalhes como o OAuth 2.0 funciona: 

__OAuth 2.0 - *Proteja suas aplicações com Spring Security OAuth2*__. 

Veja mais sobre o livro no [site da Casa do Código](https://www.casadocodigo.com.br/products/livro-oauth).

## Introdução

Algumas pessoas tem me perguntado como é escrever um livro e como foi que tudo começou. Nesse post, vou falar justamente sobre isso. Aqui eu mostro quais foram as motivações que me levaram a escrever sobre o assunto, quanto tempo gastei, como organizei meu tempo e quais são as atividades envolvidas durante a escrita de um livro técnico.

Antes de tudo, muito do que eu mostro nesse post, foi medido com uma planiha em Excel que criei, onde eu lançava as horas que trabalhava em cada tipo de atividade realizada durante a escrita. A idéia de criar essa planilha, foi inspirada no [post](https://pothix.com/post/escrevendo-o-desconstruindo-a-web/) que li do PotHix sobre como ele escreveu o livro Desconstruindo a Web (que por sinal é um livro muito bom que estou lendo agora que tenho mais tempo livre).

## Motivação

A motivação sobre escrever sobre OAuth 2.0 surgiu quando comecei a trabalhar em um projeto que demandava a construção de um Servidor de Autorização OAuth 2.0. Como o projeto usava Spring Boot, acabamos usando o Spring Security OAuth2 para proteger algumas APIs. Naquela época, tive dificuldade em entender como tudo funcionava olhando apenas para a documentação do Spring Security OAuth2. O que me faltava era um pouco mais de conhecimento teórico do framework de autorização: OAuth 2.0. 

Para isso, busquei a teoria em livros e nas próprias RFCs relacionadas ao OAuth 2.0 (principalmente a RFC 6749). Então, enquanto eu procurava entender o OAuth 2.0, ora eu estava muito focado na teoria, ora estava mais focado na prática com o framework sendo usado. Além disso, lembro que eu e um colega do trabalho acabavamos _debugando_ o código fonte do Spring Security OAuth2 muitas vezes, o que me deixava frustrado. Eu ficava frustrado, pois na minha opinião ninguém deveria ter que _debugar_ um framework para poder usá-lo.

Dessa mistura de frustração e dificuldades para entender o assunto, acabei decidindo estudar com mais profundidade, tanto a teoria apresentada nas especificações, quanto a parte prática com o Spring Security OAuth2. Acabei então decidindo condensar o resultado desse estudo em um material, que acabou sendo um livro publicado pela [Casa do Código](https://www.casadocodigo.com.br/). 

**A idéia principal era escrever o livro que eu gostaria de ler.**

## Como tudo começou

Antes de realmente começar a escrever o livro, eu estava prestes a escrever um material mais resumido. Porém, tendo em vista o quanto o assunto poderia ir longe, compartilhei a idéia com um amigo que trabalhava comigo (grande [Jorge Acetozi](https://www.jorgeacetozi.com/)) que acabou me incentivando a escrever um livro.

Após alguns dias, pensei em entrar em contato com a [Casa do Código](https://www.casadocodigo.com.br/) e propor a idéia do livro. Decidido a escrever um livro, falei com o [Newton Angelini](https://twitter.com/newtonbeck) que me indicou o contato do [Adriano Almeida](https://twitter.com/adrianoalmeida7). Uma vez que tudo ficou combinado com a [Casa do Código](https://www.casadocodigo.com.br/), comecei a pensar em como seria o livro.

## 20 dias antes de começar a escrever

Assim que tudo ficou certo com a [Casa do Código](https://www.casadocodigo.com.br/), eles criaram um repositório no GitHub e me indicaram um material muito legal contendo várias dicas importantes sobre didática e sobre como o livro deveria ser estruturado. Além disso, também me indicaram o [post](https://pothix.com/post/escrevendo-o-desconstruindo-a-web/) do PotHix onde ele conta como foi escrever o livro Desconstruindo a Web. 

Nesse post, o autor fala sobre o livro [Deep Work](https://www.amazon.com.br/Deep-Work-Focused-Success-Distracted/dp/1455586692), que se trata de um livro que mostra a importância de trabalhos realizados com profundidade enquanto critica tarefas que o autor classifica como rasas (_shallow tasks_). Como exemplo de shallow tasks, podemos citar a leitura de feeds em redes sociais como Facebook e Twitter. Muitas vezes gastamos muitas horas nessas atividades sem produzir algo significativo para nossas vidas, levando a frustrações, procrastinação e dificuldade de concentração. 

Como eu estava saindo de férias por 30 dias e iria viajar por 20 dias, pensei em ler esse livro na viagem durante os trajetos que eu faria de trem. Na minha opinião, a decisão de ler esse livro durante a viagem foi perfeita. Ao mesmo tempo em que eu descansava e conhecia coisas novas, eu também podia refletir com calma em tudo o que estava lendo. Dentre muitas coisas que esse livro me ajudou, acho que a mais importante foi definir uma rotina de trabalho.

Além de ler o __Deep Work__, assim que cheguei de viagem, estava em uma livraria onde encontrei o livro: [Não há tempo a perder](https://goo.gl/sDbqgJ) do Amyr Klink. Como ainda tinha mais alguns dias de férias, aproveitei e li mais esse livro. Eu queria muito ter bem clara a idéia de que todo o tempo livre que eu teria, seria precioso para trabalhar no livro sobre OAuth 2.0. Ler o livro do Amyr Klink, acabou dando um incentivo maior com relação a essa idéia.

## Métricas

Enquanto eu escrevia o livro, coletei algumas métricas para identificar quanto tempo eu gastaria em cada atividade envolvida na escrita de um livro, bem como o tempo geral que seria gasto em todo o processo. Basicamente, organizei essas métricas em uma tabela parecida com a mostrada na próxima imagem:

![Metrics]({{ "/assets/book1/excel.pt.png" | absolute_url }})

Apesar da imagem acima parecer uma folha de caderno, eu fui cadastrando essas métricas no Excel, o que permitiu que eu pudesse inferir mais informações como o tempo médio gasto em cada dia da semana e até mesmo classificar qual era o melhor horário para trabalhar. Analisando as métricas depois de algumas semanas, percebi que trabalhar à noite quando voltava do trabalho não era muito produtivo, pois minha energia intelectual já estava bem esgotada. 

Além disso, a noite era um momento importante para eu poder aproveitar algumas horas do dia com minha noiva. Percebi então, que o melhor horário para começar a trabalhar era entre as 5:30 e as 6:00 horas da manhã (atenção, pois os horários indicam a hora em que comecei a trabalhar).

![Horário de trabalho durante a semana]({{ "/assets/book1/worktime-durantesemana.png" | absolute_url }})

Como a escrita de um livro é um processo longo, não adianta se afobar e tentar escrever tudo em um dia só. O importante é ter um ritmo e mantê-lo até o final (assim como em uma corrida). Decidi então, seguir a seguinte rotina, baseado nas métricas que coletei e no fato de que durante a manhã eu ainda não estaria preso a nenhuma preocupação:

- Acordar aproximadamente as 5:30 da manhã
- Tomar um café
- Ler minhas próprias anotações a respeito do livro ou algum tópico relacionado ao assunto sendo abordado durante uns 10 minutos (isso ajudava a definir o _staging_ para começar a trabalhar)
- Tomar um banho e ficar pronto para trabalhar a partir das 6:00 da manhã
- Trabalhar por pelo menos 2 horas. 

Essa sequência de atividades, eu segui me baseando em um conceito apresentado no livro **Deep Work**, chamado por Carl Newport de _Eudaimonia Machine_. Essa idéia propõe um [design arquitetural](https://goo.gl/nvn9wo) que promove um ambiente de estudo e trabalho capaz de levar as pessoas a um estado de "_eudaimonia_" (basicamente um estado muito bom para produzir trabalhos que demandam concentração ou profundidade). Como eu não tenho um ambiente construído seguindo os padrões da Eudaimonia machine, acabei simulando esse ambiente com uma espécie de ritual (que é simplesmente a rotina que detalhei mais acima).

> [Eudaimonia](https://pt.wikipedia.org/wiki/Eudaimonia) é um termo grego geralmente traduzido como felicidade e que pode ser usado para expressar o que seria a plenitude de um ser.

Eram duas horas trabalhando focado sem distração (celular, redes sociais, e barulho - essas são horas em que as pessoas fazem menos barulho no mundo). O legal dessa experiência, foi perceber que depois de ficar totalmente focado por aproximadamente 90 minutos, as horas seguintes pareciam passar muito rápido. Era como se após 90 minutos concentrado eu parasse de perceber o tempo. Acredito que esse estado é o que costumam chamar de _Flow_ ou _the zone_ (existe uma apresentação [TED Talk](https://www.ted.com/talks/mihaly_csikszentmihalyi_on_flow) bem legal sobre esse assunto). 

Olhando novamente o gráfico anterior, dá pra perceber que também trabalhei em alguns horários estranhos, como durante a madrugada (3 da manhã) e alguns horários como 10 da manhã ou 14 horas. Nesse momento você deve se perguntar: mas entre as 10 da manhã e as 19 você não está no escritório da empresa? Sim, a resposta é sim. Mas esses horários existem, pois houveram dias em que não foi possível ir para o escritório por algum motivo especial ou porque era feriado.

Já durante o final de semana, eu acordava um pouco mais tarde e geralmente começava a trabalhar as 10 da manhã. Conforme pode ser observado no gráfico a seguir, no horário da tarde, algumas vezes comecei a trabalhar aproximadamente as 16 horas. 

![Horário de trabalho durante o final de semana]({{ "/assets/book1/worktime-fimdesemana.png" | absolute_url }})

Foi difícil estabelecer uma rotina bem regular durante o final de semana, por conta de algumas tarefas como fazer uma compra, ajudar nas tarefas de casa, ou até mesmo alguma manutenção.

Através das métricas que fui coletando, também foi possível gerar um gráfico bem interessante que mostra o tempo médio que gastei trabalhando no livro por dia da semana (dá pra ver que o tempo em que eu trabalhava no livro durante o final de semana foi bem superior ao tempo médio gasto durante a semana).

![Tempo médio de trabalho por dia da semana]({{ "/assets/book1/time-by-weekday-media.png" | absolute_url }})

Seguindo no ritmo apresentado, foi possível finalizar a primeira versão do livro (contando com a revisão técnica do [Rafael Chinelato](https://twitter.com/RafaDelNero)) em meados de 23 de Julho de 2017. Em seguida, foram feitas as revisões de Português pela [Casa do Código](https://www.casadocodigo.com.br/). 

No total, contando com escrita, revisão técnica, revisão didática, revisão de Português, pesquisa, escrita de código e criação dos desenhos, foram **295 horas de trabalho** (aqui não estou contabilizando o tempo gasto pelos revisores). O tempo total gasto em cada uma das atividades também foi contabilizado conforme mostra o gráfico a seguir:

![Tempo gasto por atividade]({{ "/assets/book1/category-pt.png" | absolute_url }})

Escrita, revisão, pesquisa e escrita de código foram as principais tarefas executadas durante a produção do livro. Apesar da atividade de escrita ter ficado em primeiro lugar, considerando a quantidade de tempo gasto no processo, é interessante notar que a revisão ficou em segundo lugar. A experiência de trabalhar nesse livro me mostrou o quão importante é a revisão de um livro. 

Além da ajuda dos revisores, percebi que também é muito importante que o próprio autor faça uma revisão depois de algum tempo de trabalho. Se possível, quanto mais tempo o autor demorar pra fazer a revisão melhor, pois estará menos enviesado no conteúdo (e poderá encontrar mais problemas).

## Revisando o livro

A [Casa do Código](https://www.casadocodigo.com.br/) ajudou muito no processo de revisão didática e na revisão de Português. Atualmente é possível trabalhar na escrita de um livro de forma independente, por exemplo através do [Leanpub](https://leanpub.com/). Entretanto, trabalhar com uma editora como a Casa do Código me permitiu aprender bastante sobre didática e sobre como um livro é produzido. Além disso, existem muitas outras vantagens de se trabalhar com uma editora, como por exemplo ter o livro impresso e distribuído pelo país.

Quanto à revisão técnica, foi muito importante ter a ajuda do Rafael Chinelato, pois muitos pontos que estariam complicados para o leitor foram melhorados para facilitar o entendimento e execução dos exemplos.

## Revisão e ferramentas

Como citei anteriormente, também fiz uma revisão técnica do próprio livro. Para revisar o livro, conforme eu ia escrevendo, utilizei o aplicativo do **Kindle** para mobile. Dessa forma eu aproveitava o caminho de casa para o trabalho para ir lendo o que eu já havia escrito e ir anotando o que poderia ser melhorado. Através desse aplicativo ficava fácil adicionar anotações e marcações no texto.

Também utilizei o aplicativo **One Note** da Microsoft no celular para ir adicionando idéias que eu ia ganhando na medida que também estudava mais sobre OAuth 2.0. Durante a leitura de outros materiais ganhei bastante _insights_ para a escrita do livro. Um dos _insights_ mais importantes que anotei foi quanto ao formato que o código fonte deveria ser apresentado e como a aplicação de exemplo deveria ser.

## Problemas encontrados

Apesar de tudo, nem tudo são flores. Durante a escrita do livro houveram momentos em que eu ficava bastante preocupado pensando se realmente conseguiria escrever o livro inteiro. Acabei tendo dificuldades em alguns assuntos e outras vezes o Spring Security OAuth2 não funcionava da forma como eu esperava. Nesses momentos, eu não tinha pra onde correr. 

O jeito que encontrei foi sair (sair significava no máximo fazer uma caminhada), relaxar um pouco e voltar ao assunto com toda paciência possível, debugando o projeto de exemplo e o framework sendo usado, fazendo desenhos no papel ou até mesmo desenhando alguns esquemas na janela do quarto.

Outro problema que encontrei, foi a respeito da estrutura do livro. Apesar de ter estruturado tudo no início do projeto, percebi que teria que fazer uma refatoração em como apresentar os exemplos quando já estava na metade do livro. Ao mesmo tempo que isso me deixou bastante preocupado, voltar atrás e refatorar o que foi preciso, me deu mais segurança para continuar trabalhando até o final.

## Conclusão

Eu sempre quis poder contribuir com um trabalho desse tipo, porém percebi que não era algo que poderia ser feito sem uma motivação principal. Conforme citei no início, a minha principal motivação foi escrever exatamente o livro que eu gostaria de ler e principalmente que o resultado desse trabalho também pudesse ajudar outras pessoas. O livro que eu gostaria de ler sobre OAuth 2.0 era um livro didático onde eu pudesse aprender os detalhes da especificação sem deixar de lado a parte prática. 

Agora que esse livro existe, espero que ele seja útil para pessoas que se encontrem na mesma situação em que eu já estive quando comecei a trabalhar com OAuth 2.0.
