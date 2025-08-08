---
layout: post
title:  "Criando um emulador de terminal do zero"
date:   2025-05-03 19:00:00 +1100
lang: pt
ref: terminal
comments: true
categories: terminal linux xterm
description: Entendendo como funciona um emulador de terminal
---

Neste post eu escrevo sobre como implementei um emulador de terminal super simplificado que me ajudou a entender como ele funciona por trás dos bastidores, enquanto também aprendi muitas coisas interessantes sobre o Linux ao longo do caminho. Escrever um terminal completo está longe de ser uma tarefa fácil, então este post é apenas a ponta do iceberg. Ainda assim, foi interessante o suficiente para despertar ainda mais a minha curiosidade sobre como as coisas funcionam.

Todo o código fonte deste projeto está disponível no [GitHub](https://github.com/adolfoweloy/termini).

![Homem digitando no terminal]({{ "/assets/terminals_distilled/image.png" | absolute_url }})

A ideia de escrever meu próprio terminal simplificado começou quando me deparei com algumas sequências de caracteres estranhas que encontrei ao configurar uma variável de prompt de comando usada para fins de formatação (a tal PS1). Voltar do MacOS para o Linux como desktop me levou a decidir usar [bash](https://www.gnu.org/software/bash/) novamente em vez de [zsh](https://www.zsh.org/) e percebi que no meu prompt faltavam informações sobre o branch do git, bem como a formatação de cores que eu usava. Como eu tenho [meus dotfiles no GitHub](https://github.com/adolfoweloy/cmdcenter), minha primeira reação foi: "por que isso está faltando no meu arquivo .bashrc?". Felizmente, eu salvei isso como um [gist](https://gist.github.com/adolfoweloy/64983c65457f04a947c3) no GitHub, e isso é o que eu encontrei por lá:

```bash
function git_branch_name () {
    branch=$(git symbolic-ref --short HEAD 2>/dev/null)
    if [ -n "$branch" ]; then
        echo "($branch) "
    fi
}

PS1="\[\033[38;5;4m\]\u@\h\[\033[0m\]:"
PS1=$PS1"\[\033[38;5;083m\]\w\[\033[0m\] "
PS1=$PS1"\[\033[38;5;122m\]\$(git_branch_name)\[\033[0m\]$ "
```

A meu ver, o conteúdo do PS1 não é muito legível e esse foi o começo de tudo sobre essa jornada.

Só para constar, isso pode ser facilmente gerado usando o [Bash Prompt Generator](https://bash-prompt-generator.org/), no entanto, ainda assim eu atualizei meus "dotfiles" com a formatação de prompt bash que eu gosto ([aqui você pode ver como eu configurei meus dotfiles](https://github.com/adolfoweloy/cmdcenter)).

---
Antes de prosseguir para as próximas seções, quero mencionar que existem muitos blog posts semelhantes por aí e alguns são realmente bons (com algumas intersecções de conteúdo com meu post). Meu favorito, que me ajudou com os primeiros passos, é o [Terminal anatomy](https://poor.dev/blog/terminal-anatomy/). No entanto, acho mais fácil de ler aquele post agora que fiz minha própria pesquisa sobre coisas que não estavam claras para mim naquele momento. Os outros dois posts que acho que valem a pena ler, caso você queira se aprofundar em outros aspectos do terminal, estão listados nas referências no final desta página. E por último, mas não menos importante, acho que [The secret rules of the terminal](https://jvns.ca/blog/2025/06/24/new-zine--the-secret-rules-of-the-terminal/) de Julia Evans também pode ser muito legal. Eu ainda não li o Zine dela, mas estou considerando comprá-lo em algum momento.

---

## A estranha variável PS1

Primeiramente, essa coisa do PS1 é estranha o suficiente para pessoas não familiarizadas com o shell e o terminal no Linux (sim, shell e terminal não são a mesma coisa e eu vou dissecar isso neste post). Para começar, `PS1` não é nem mesmo uma variável de ambiente. É uma variável de shell (mais especificamente, uma variável do bourne shell que o bash usa). 

Do [manual do bash](https://www.gnu.org/software/bash/manual/html_node/Bourne-Shell-Variables.html) em tradução livre:
> PS1 é uma string de prompt primária. O valor padrão é ‘\s-\v\$ ’. Veja Controlando o Prompt, para a lista completa de sequências de escape que são expandidas antes que PS1 seja exibido.

Mas uma vez que o propósito dessa variável é desmistificado, olhe para essas coisas estranhas como `\[\033[38;5;4m\]\u@h\[\033\[0m\]`. Há muito acontecendo ali, então deixe-me explicar:

| **Elemento**       | **Descrição**                                                                                         |
|-------------------|---------------------------------------------------------------------------------------------------------|
| **\\[**           | Inicia uma string de caracteres não imprimíveis.     |
| **\\]**           | Fecha essa string de caracteres não imprimíveis. |
| **\\033[**        | Inicia os parâmetros SGR. `\033` também pode ser representado como `\E` ou `\x1b` em diferentes contextos.|
| **38;5;4m**       | Os parâmetros **SGR**. Isso formata a cor do primeiro plano usando o estilo de 256 cores (em oposição às cores ANSI) com verde claro (número 004 hexa) |

**SGR** significa Select Graphic Rendition e é uma instrução ANSI usada para definir atributos gráficos de texto no terminal.
É usado com o finalizador `m` e aceita vários parâmetros, como cor, negrito e até blink. Note que todos esses caracteres de escape não são usados apenas para formatar o prompt, mas para formatar qualquer caractere em qualquer lugar do terminal. Como um experimento rápido, sugiro tentar o seguinte comando no terminal:

```bash
echo -e "\033[38;5;4mhello\033[0m \033[38;5;205mworld\033[0m"
```

A saída deve se parecer com isto:
![Formatando hello world com cores]({{ "/assets/terminals_distilled/hello-world.png" | absolute_url}})

Meu objetivo com este post está longe de aprofundar como essa notação realmente funciona. Existem boas documentações - na verdade, documentações oficiais - disponíveis também (caso você queira validar o que o ChatGPT venha a te explicar sobre o assunto). Para mais informações sobre a sintaxe dessa sequência de formatação, consulte [Bash Manual / Controlling the Prompt](https://www.gnu.org/software/bash/manual/html_node/Controlling-the-Prompt.html).

Em relação às cores, acabei criando um script bash apenas por diversão para imprimir uma paleta de 256 cores no terminal pra mim, e que desenha algo como segue quando digito `colors` no terminal (se você quiser, basta dar uma olhada no código-fonte em [colors.sh no repositório GH cmdcenter](https://github.com/adolfoweloy/cmdcenter/blob/main/scripts/colors.sh)):

![256 Color palete]({{ "/assets/terminals_distilled/color-palete.png" | absolute_url }})

Isso me ajuda a escolher o número da cor certa para definir como parâmetros SGR e eu acho legal vê-las no meu terminal :)

Enquanto eu estava quase concluindo este post, percebi que outra pessoa também escreveu um script parecido, embora escrito em Python: [Build your own command line with ANSI escape codes](https://www.lihaoyi.com/post/BuildyourownCommandLinewithANSIescapecodes.html]). Confesso que prefiro meu próprio script.

## E o terminal?

Justo, acho que quem estiver lendo isso já pode estar impaciente a essa altura querendo começar a codar logo e ver o tal emulador de terminal funcionando. No entanto, alguns conceitos devem ser entendidos antes de sair codando. Eu acho que a melhor maneira de entender todas as diferentes partes de um emulador de terminal é começar com modelos mentais mais simples primeiro. O diagrama abaixo mostra uma abstração muito simplificada da arquitetura do emulador de terminal que deve ajudar a construir esse modelo mental de que estou falando.

![1000ft view of terminal architecture]({{ "/assets/terminals_distilled/terminal-architecture-overview.png" | absolute_url }})

## Os principais componentes

#### O emulador de terminal
O emulador de terminal é um aplicativo gráfico que permite que você digite comandos e interaja com seu sistema operacional __*nix__. Exemplos de emuladores de terminal incluem gnome-terminal, Alacrity, ST ([Simple Terminal](https://st.suckless.org/)), iterm do MacOS, xterm e muitos outros. No momento em que escrevia esse post, encontrei uma página com uma lista de mais de [30 Emuladores de Terminal Linux](https://linuxblog.io/linux-terminal-emulators/).

O nome emulador de terminal pode parecer estranho, mas vem de muito tempo atrás —antes dos computadores— quando TTYs (teletypewriters) eram _usados para distribuir preços de ações a longas distâncias em tempo real_ [^1]. Esses dispositivos evoluíram para terminais CRT e teclado conectados a computadores usando compartilhamento de tempo, onde o mainframe gerenciava a manipulação de texto, como backspace e movimentos do cursor. O Unix mais tarde abstraiu esses papéis com software, permitindo que terminais físicos e "pseudo-terminais" interagissem com o sistema.

#### O shell
O shell é apenas um tipo de processo que você pode executar em um terminal, com exemplos populares incluindo bash, zsh e sh. Mas os emuladores de terminal não se limitam a shells — eles podem hospedar qualquer programa interativo, como ssh, tmux ou REPLs de linguagem como Python ou Ruby (i.e. irb). Esses programas recebem a entrada do usuário no emulador de terminal, processam e enviam a saída de volta — muitas vezes formatada usando sequências de escape ANSI para controlar a cor do texto, estilo, posicionamento do cursor e mais.

#### O kernel
Eis o Kernel do Linux. O kernel lida com todos os detalhes de baixo nível que permitem que os emuladores de terminal se comuniquem com processos como bash, ssh, tmux e outros. Num nivel detalhe um pouco maior, essa interação é gerenciada pelo subsistema TTY usando a abstração PTY (pseudo-terminal), que faz a ponte entre o emulador de terminal (no lado primário) e o processo (no lado secundário). É uma parte complexa e fascinante do sistema, e vamos explorá-la em mais detalhes mais adiante.

PS: Estou usando as palavras primário e secundário para me referir aos lados do PTY, até mesmo para manter a consistência com outros posts que li como é o caso no post do blog [Terminal anatomy](https://poor.dev/blog/terminal-anatomy/). No entanto, algumas fontes se referem a eles como _master_ e _slave_, o que está ultrapassado.

## Criando o emulador de terminal

Das três caixas que estão sendo usadas como o simples modelo mental de um emulador de terminal, o emulador de terminal em si é o que é implementado nesse post. Lembre-se, esta é uma aplicação gráfica, então a primeira coisa a criar é uma janela que possa renderizar caracteres que um usuário digita e caracteres que vêm do shell. Para conseguir isso, o código abaixo inicializa uma janela SDL, que servirá como tela para o nosso mini terminal.

```c
#include <SDL2/SDL.h>

int main() {
    SDL_Init(SDL_INIT_VIDEO);

    SDL_Window* win_sdl = SDL_CreateWindow(
      "Mini Terminal",
      SDL_WINDOWPOS_CENTERED, 
      SDL_WINDOWPOS_CENTERED,
      1200, 600, SDL_WINDOW_SHOWN);

    int running = 1;
    SDL_Event event;
    while (running) {

        while (SDL_PollEvent(&event)) {
            if (event.type == SDL_QUIT) {
                running = 0;
            } else if (event.type == SDL_KEYDOWN) {
                if ((event.key.keysym.mod & KMOD_CTRL) 
                  && event.key.keysym.sym == SDLK_c) {
                    running = 0;
                }
            }
        }
        
        SDL_Delay(10);
    }

    SDL_DestroyWindow(win_sdl);
    SDL_Quit();
    return 0;
}
```

Compilar este código requer o a biblioteca SDL2, que pode ser instalada no Ubuntu com `sudo apt install libsdl2-dev`. Depois disso, você pode compilá-lo usando:

```bash
gcc -o mini_terminal mini_terminal.c -lSDL2
```

Então execute o mini terminal usando `./mini_terminal`. Você deve ver uma janela vazia como a seguir:

<p align="center">
  <img src="{{ "/assets/terminals_distilled/terminal.png" | absolute_url }}" width="500" height="250" alt="Empty terminal">
</p>

## Criando a ponte para o shell

Neste ponto, esta janela do emulador de terminal tem apenas uma funcionalidade: ela pode ser fechada. Não é exatamente um emulador de terminal ainda, mas agora posso começar a conectar os pontos entre o Kernel e o shell.

O próximo passo envolve fazer com que o emulador de terminal leia e escreva caracteres vindos do shell. Para conseguir isso, vou usar a biblioteca `pty` que está disponível no Linux. O PTY (pseudoterminal) é uma abstração que representa uma ponte entre o emulador de terminal e o shell. Esses dois programas se comunicam através de um _canal de comunicação assíncrono bidirecional fornecido pelo PTY_[^2]. O emulador de terminal escreve no lado primário do PTY, enquanto o shell lê do lado secundário do PTY. Isso permite que o emulador de terminal envie comandos para o shell e receba a resposta de volta. Vamos dar mais um passo melhorando o modelo mental com mais detalhes.

![Terminal architecture with PTY]({{ "/assets/terminals_distilled/terminal-architecture-with-pty.png" | absolute_url }})

Em termos mais concretos, o PTY é um par de dispositivos virtuais: o primário e o secundário. O lado primário é com o que o emulador de terminal interage, enquanto o lado secundário é com o que o shell interage. O lado primário do PTY é representado pelo arquivo (device virtual) `/dev/ptmx`, que é um arquivo usado para multiplexação permitindo que vários emuladores de terminal se conectem ao mesmo PTY. O lado secundário é representado por `/dev/pts/N`, onde `N` é um número atribuído a cada PTY secundário. Isso tudo fará mais sentido assim que adicionarmos mais detalhes nesse código (o trecho abaixo isso é executado antes que o emulador de terminal comece a ouvir eventos SDL).

```c
pid = forkpty(&primary_fd, NULL, &term, &win);
if (pid == -1) {
    perror("Error creating pseudo-terminal");
    return 1;
}

if (pid == 0) {
    printf("Initializing the shell...\n");

    int res = create_pid_file();

    if (res > 0) {
      return res;
    }

    execlp(getenv("SHELL"), getenv("SHELL"), NULL);

    perror("execlp");
    exit(1);
}
```

A ideia principal por trás da criação do PTY vem da utilização da função `forkpty`, que cria um pseudo-terminal e faz o fork de um processo filho. Essa função abre o arquivo `/dev/pts/ptmx`, que é o lado primário do PTY, e retorna um descritor de arquivo que pode ser usado para ler e escrever no PTY. Por outro lado, a entrada e saída padrão do processo filho estão diretamente conectadas ao lado secundário do PTY, que é representado por `/dev/pts/N`.

| Path              | Description                                                                 |
|-------------------|-----------------------------------------------------------------------------|
| `/dev/pts/ptmx`   | Arquivo (device virtual) usado pelo terminal para ler e escrever no PTY |
| `/dev/pts/N`      | N é um número atribuído a um processo secundário que responde ao emulador de terminal (por exemplo, bash). |


---
**Um experimento com PTY**

Como um experimento, se você digitar `tty` no terminal, você obterá o arquivo correspondente `/dev/pts/N`, por exemplo, `/dev/pts/19`. Então, se eu for para outro terminal e digitar `cat /dev/pts/19`, tudo o que eu digitar no terminal anterior será impresso na saída do cat.

---

## Renderizando caracteres lidos do PTY

A parte que falta para termos um emulador de terminal funcional é lidar com a comunicação entre o emulador de terminal e o shell, além de renderizar os caracteres na tela. O código abaixo mostra como ler do lado primário do PTY e renderizar os caracteres na janela SDL. A primeira coisa a fazer é criar um buffer para armazenar os caracteres lidos do lado primário do PTY, ou seja, `/dev/pts/ptmx` e anexá-lo à variável global do buffer que armazena todas as linhas de texto já escritas no terminal (algo feito pela função `append_line_to_lines_buffer()`).

```c
int running = 1;
SDL_Event event;
char buf[256];

while (running) {
    // reading the shell output from the primary file descriptor
    fd_set fds;
    FD_ZERO(&fds);
    FD_SET(primary_fd, &fds);
    struct timeval tv = {0, 10000}; // 10ms

    if (select(primary_fd + 1, &fds, NULL, NULL, &tv) > 0) {
        ssize_t n = read(primary_fd, buf, sizeof(buf) - 1);
        if (n > 0) {
            buf[n] = '\0';
            append_line_to_lines_buffer(buf);
        }
    }
    // ommitting the rest of the code for brevity
```

Uma vez que o buffer é atualizado com a próxima linha lida do PTY, o emulador de terminal pode renderizar os caracteres na tela. Isso envolve processar o buffer de linhas e desenhar os caracteres usando a biblioteca gráfica SDL da seguinte forma:

```c
SDL_SetRenderDrawColor(renderer, 0, 0, 0, 255);
SDL_RenderClear(renderer);

int y = 0;
for (int i = 0; i < num_lines; ++i) {
    SDL_Surface* surface = TTF_RenderText_Solid(
      font, lines[i], (SDL_Color){255, 255, 255});
    SDL_Texture* texture = SDL_CreateTextureFromSurface(renderer, surface);
    SDL_Rect dst = {10, y, surface->w, surface->h};
    SDL_RenderCopy(renderer, texture, NULL, &dst);
    SDL_FreeSurface(surface);
    SDL_DestroyTexture(texture);
    y += FONT_SIZE;
}

SDL_RenderPresent(renderer);
SDL_Delay(10);
```

Note que `lines[i]` é o buffer que contém todos os caracteres lidos do PTY, que é anexado pela função `append_line_to_lines_buffer()` declarada da seguinte forma:

```c
void append_line_to_lines_buffer(const char* text) {
    if (num_lines < MAX_LINES) {
        snprintf(lines[num_lines++], MAX_LINE_LENGTH, "%s", text);
    }
}
```

Agora, executar este código deve resultar em algo que se pareça mais com um emulador de terminal, embora seja completamente básico. Por enquanto, ele apenas renderiza caracteres vindos do shell, mas ainda não lida com a entrada do usuário.

![Barebone terminal emulator]({{ "/assets/terminals_distilled/barebone-terminal.png" | absolute_url }})

## Tratando a entrada do usuário

Para lidar com a entrada do usuário, o emulador de terminal precisa ler os caracteres digitados pelo usuário e escrevê-los no lado primário do PTY. Isso é feito capturando eventos de teclado do SDL e escrevendo os caracteres no descritor de arquivo primário do PTY. O código abaixo mostra como lidar com eventos de teclado e escrever os caracteres no PTY (isso é apenas uma melhoria no loop que foi criado anteriormente para lidar com eventos SDL):

```c
while (SDL_PollEvent(&event)) {
    if (event.type == SDL_QUIT) {
        running = 0;
    } else if (event.type == SDL_TEXTINPUT) {
        write(primary_fd, event.text.text, strlen(event.text.text));
    } else if (event.type == SDL_KEYDOWN) {
        if ((event.key.keysym.mod & KMOD_CTRL) 
          && event.key.keysym.sym == SDLK_c) {
            running = 0;
        } else if (event.key.keysym.sym == SDLK_RETURN) {
            write(primary_fd, "\n", 1);
        } else if (event.key.keysym.sym == SDLK_BACKSPACE) {
            write(primary_fd, "\x7f", 1);
        }
    }
}
```

O emulador de terminal agora renderiza caracteres digitados pelo usuário e texto que vem do shell. Embora esteja longe de ser perfeito (se você testá-lo, notará que não lida com movimento do cursor, quebra de linha ou qualquer outro recurso avançado), é um bom começo para entender como um emulador de terminal funciona. Mas é hora de fechar o ciclo aqui e conectar tudo com a pergunta inicial sobre a variável `PS1` e o emulador de terminal.

## Onde estão as sequências de escape ANSI?

Olhando de perto para o que está sendo renderizado, dá pra perceber alguns caracteres estranhos impressos como caixinhas ou pontos de interrogação. Isso ocorre porque o emulador de terminal está tentando imprimir caracteres não imprimíveis. Eu preciso lidar com eles de uma maneira que eles possam ser visíveis para o usuário para fins educacionais. O código abaixo mostra como lidar com sequências de escape ANSI e renderizá-las como caracteres visíveis no emulador de terminal.

```c
void escape_and_append(const char* input) {
    char escaped[MAX_LINE_LENGTH * 4]; // buffer maior para acomodar escapes
    int j = 0;

    for (int i = 0; input[i] != '\0' && j < MAX_LINE_LENGTH - 4; ++i) {
        unsigned char c = input[i];
        if (c == '\n') {
            escaped[j++] = '\\';
            escaped[j++] = 'n';
        } else if (c == '\t') {
            escaped[j++] = '\\';
            escaped[j++] = 't';
        } else if (c < 0x20 || c >= 0x7f) { // não imprimível
            j += snprintf(&escaped[j], 5, "\\x%02x", c);
        } else {
            escaped[j++] = c;
        }
    }

    escaped[j] = '\0';
    append_line_to_lines_buffer(escaped);
}
```

Agora eu substituo a utilização da função `append_line_to_lines_buffer()` por `escape_and_append()` e é isso que o usuário deve ser capaz de ver agora:

![Terminal emulator with ANSI escape sequences]({{ "/assets/terminals_distilled/the-output.png" | absolute_url }})

Olhando esse screenshot de perto, você verá que antes do nome de usuário, há a sequência de escape `\x1b[38;5;4m`, que é o parâmetro SGR para a cor verde claro. Esta é a mesma sequência que foi usada na variável `PS1` para formatar o prompt. O mesmo se aplica às outras sequências de escape que são impressas no emulador de terminal.

Detalhe: Ao declarar a variável `PS1`, na verdade eu utilizo `\033` como uma representação do caractere de escape, que é equivalente a `\x1b`.

Todas essas sequências de escape são então interpretadas pelo emulador de terminal para aplicar a formatação correspondente (por exemplo, mudar a cor do texto) ao renderizar a saída. Alguns emuladores de terminal podem depender de bancos de dados `terminfo` ou `termcap` para mapear essas sequências de escape para _capabilities_ específicas do terminal, mas neste caso, o emulador de terminal está lidando com elas diretamente (Alacrity faz o mesmo).

## Um pouco mais sobre o PTY e o kernel

Neste ponto, alguém ainda pode perguntar: tá mas, onde e como a comunicação realmente acontece no kernel? A resposta para essa pergunta é que o subsistema TTY no kernel é responsável por gerenciar dispositivos de terminal, incluindo PTYs. O subsistema TTY fornece uma estrutura para drivers de terminal, que lidam com os detalhes de baixo nível de leitura e gravação de dados em dispositivos de terminal.

Os principais componentes do subsistema TTY que tive a chance de explorar no código-fonte do kernel Linux são:

1. **Drivers TTY**: Esses são responsáveis por interagir com o hardware de terminal real ou dispositivos de terminal emulados (como PTYs). Eles implementam as funções necessárias para ler e escrever no terminal.

2. **Line discipline**: Esta é uma camada que fica entre o driver TTY e o usuário do terminal. Ela é responsável por processar dados de entrada e saída, lidando com coisas como edição de linha, sinais de controle e mais.

O _workflow_ do subsistema TTY pode ser resumido da seguinte forma:

O PTY primário, geralmente acessado pelo emulador de terminal via `/dev/ptmx`, recebe dados escritos pelo emulador. Esses dados são tratados pelo subsistema TTY no kernel, onde o driver TTY gerencia a lógica de I/O de baixo nível e interage com o **Line discipline**, que processa os dados (por exemplo, executando um ECHO, armazenando em buffer, processando sinais de terminal). Na outra extremidade, o PTY secundário (por exemplo, `/dev/pts/N`) está conectado a um processo como o bash, que se comunica através da entrada e saída padrão. O kernel vincula ambas as extremidades, fazendo com que o shell acredite que está conectado a um terminal físico real.

## Proximos passos

Como parte de trabalhos futuros, planejo implementar algumas das funcionalidades que estão faltando neste emulador de terminal, como:
- Manipulação de movimento do cursor e quebra de linha
- Implementação de recursos básicos de edição de texto (por exemplo, copiar, colar e excluir)
- Adição de suporte a sequências de escape ANSI para controlar a formatação do texto (por exemplo, negrito, sublinhado e cores)

Toda essa exploração também despertou minha curiosidade sobre o funcionamento interno do ptmx e como registrar dispositivos no kernel. Estou planejando escrever um post sobre isso no futuro, então fique ligado!

## Considerações finais

Ao explorar a arquitetura do emulador de terminal, achei fascinante quantas camadas de abstração estão envolvidas no processo de renderização de caracteres na tela. Ler o trabalho de outros neste espaço me proporcionou insights valiosos sobre as complexidades da emulação de terminais e os sistemas subjacentes em jogo.

Além disso, gostaria de mencionar o quanto algumas ferramentas de IA me ajudaram, especialmente na compreensão dos componentes do código-fonte do kernel Linux relacionados ao subsistema TTY. Nada se compara a ler a partir do código fonte. Se você quiser explorá-lo por conta própria, pode encontrar o código do subsistema TTY no código-fonte do kernel do Linux em `drivers/tty/`. A implementação do PTY está em `drivers/tty/pty.c`, e o _core_ do TTY está em `drivers/tty/tty_io.c`.

## Referências

- [Terminal anatomy](https://poor.dev/blog/terminal-anatomy/)
- [The TTY demystified](https://www.linusakesson.net/programming/tty/)
- [Build your own Command Line with ANSI escape codes](https://www.lihaoyi.com/post/BuildyourownCommandLinewithANSIescapecodes.html)
- [The Secret Rules of the Terminal](https://jvns.ca/blog/2025/06/24/new-zine--the-secret-rules-of-the-terminal/)
- [List of 30+ Linux Terminal Emulators](https://linuxblog.io/linux-terminal-emulators/)
- [IBM - Time-Sharing](https://www.ibm.com/history/time-sharing)
- [Linux kernel source](https://github.com/torvalds/linux/)

## Footnotes

[^1]: [The TTY Demystified](https://www.linusakesson.net/programming/tty/) (veja também a seção "History" no artigo)
[^2]: [Terminal anatomy](https://poor.dev/blog/terminal-anatomy/)