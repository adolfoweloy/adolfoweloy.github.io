---
layout: post
title:  "Creating a terminal emulator from scratch"
date:   2025-05-03 19:00:00 +1100
ref: terminal
comments: true
categories: terminal linux xterm
description: Understanding how a terminal emulator works under the hoods
---

In this post I write about how I implemented an over simplified terminal emulator that helped me understanding how it works behind the scenes while also learning many interesting things about Linux along the way. Writing a full blown terminal is far from an easy task, so this post is just the tip of the iceberg. Although, interesting enough for me to spark the curiosity about how things work.

![Man typing on a terminal]({{ "/assets/terminals_distilled/image.png" | absolute_url }})

This idea about writing my own simplified terminal started when I stumbled upon some weird characters sequence that I found when setting up a command prompt variable used for formatting purposes. Moving back from MacOS to Linux as a desktop led me to decide using [bash](https://www.gnu.org/software/bash/) again instead of [zsh](https://www.zsh.org/) and I noticed that my prompt was missing git branch info as well as some nice color formatting. Since I actually have [my dotfiles on GitHub](https://github.com/adolfoweloy/cmdcenter), my first reaction was: "why is this missing from my .bashrc file?". Fortunately I have this saved as a GitHub [gist](https://gist.github.com/adolfoweloy/64983c65457f04a947c3), and that is what I found:

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

Just for the records, this can be easily generated using [Bash Prompt Generator](https://bash-prompt-generator.org/), however in any case, I just updated my "dotfiles" with the bash prompt formatting that I like ([here is how I setup my dotfiles](https://github.com/adolfoweloy/cmdcenter)).

Before proceeding to the next sections, I want to mention that there a lot of similar blog posts out there and some are really good (expect some overlaps). My favourite, which helped me with the first steps is the [Terminal anatomy](https://poor.dev/blog/terminal-anatomy/). Nevertheless, I find it easier to read now that I did my own research on things that were not clear to me when I first found the Terminal anatomy blog post. The other two posts that I think are worth reading in case you want to go deeper (I just skimmed through them for now), are listed in the references at the bottom of this page. And last but not least, I think [The Secret Rules of the Terminal](https://jvns.ca/blog/2025/06/24/new-zine--the-secret-rules-of-the-terminal/) by Julia Evans might also be pretty cool. I haven't read that yet, but I am considering buying it at some point. 

## The weird PS1 variable

First of all, this PS1 thing is weird enough for people unfamiliar with the Linux shell and terminal (yes shell and terminal are not the same thing and I will dissect this in this post). To start off, `PS1` is not even an environment variable. It is a shell variable (more specifically, a bourne shell variable that bash uses). From the [bash manual](https://www.gnu.org/software/bash/manual/html_node/Bourne-Shell-Variables.html):

> The primary prompt string. The default value is ‘\s-\v\$ ’. See Controlling the Prompt, for the complete list of escape sequences that are expanded before PS1 is displayed.

But once this variable purpose is demystified, look at these weird stuff like `\[\033[38;5;4m\]\u@h\[\033\[0m\]`.
There's a lot going on there, so let me break it down:

| **Element**       | **Description**                                                                                         |
|-------------------|---------------------------------------------------------------------------------------------------------|
| **\\[**           | Starts a string of non-printing characters.                                                             |
| **\\]**           | Closes this non-printing characters string.                                                             |
| **\\033[**        | Starts the SGR parameters. `\033` can also be represented as `\E` or `\x1b` in different contexts.|
| **38;5;4m**       | The SGR parameters. This is formatting the foreground color using the 256 color style (as opposed to ANSI colors) with light green (number 004 hexa) |

SDR stands for Select Graphic Rendition and it is an ANSI instruction used to define graphical attributes of text in the terminal.
It is used with the `m` finalizer and accepts multiple parameters such as color, bold, and reset. Look that all of these escape characters aren't used only for the purposes of formatting the prompt, but to format any character anywhere in the terminal. Just try the following command on the terminal:

```bash
echo -e "\033[38;5;4mhello\033[0m \033[38;5;205mworld\033[0m"
```

The output should look like follows
![Formatting hello world with colors]({{ "/assets/terminals_distilled/hello-world.png" | absolute_url}})

My purpose with this post is far from going deep on how this notation actually works. There are good documentation -- actually source of truth -- available as well (in case you want to validate what ChatGPT tells you). For more about the syntax of this formatting sequence, check [Bash Manual / Controlling the Prompt](https://www.gnu.org/software/bash/manual/html_node/Controlling-the-Prompt.html).

In relation to the colors, I ended up creating a bash script just for fun to print a 256 terminal color palete for me which prints something like follows in the terminal when I type `colors` (if you want it, just have a look at the source code at [colors.sh in cmdcenter GH repository](https://github.com/adolfoweloy/cmdcenter/blob/main/scripts/colors.sh)):

![256 Color palete]({{ "/assets/terminals_distilled/color-palete.png" | absolute_url }})

This helps me picking up the right color number to set as part SGR parameters and this is also fun to see them printed on my terminal :)

PS: I just realised that someone writing about building your own command line also did something similar but written in Python: [Build your own command line with ANSI escape codes](https://www.lihaoyi.com/post/BuildyourownCommandLinewithANSIescapecodes.html]). Sorry, but I prefer mine :D

## Where is the terminal?

Fair enough, whoever is reading this might be a bit impatient to start typing code to create a very simple terminal on their own. However, some concepts must be understood before I jump into the code. I think that the best way to understand all the moving parts of a terminal emulator is to start with simpler mental models first. The diagram below shows a very simplified mental model that should help building up confidence before coding.

![alt text](image.png)

![1000ft view of terminal architecture]({{ "/assets/terminals_distilled/terminal-architecture-overview.png" | absolute_url }})

## The main components

#### The terminal emulator
The terminal emulator is a graphical application that allows you to type in the commands and interact with your __*nix__ operational system. Examples of a terminal emulator are gnome-terminal, Alacrity, ST ([Simple Terminal](https://st.suckless.org/)), iterm from MacOS, xterm and many many more. At the time of this writing I found a page with a list of [30+ Linux Terminal Emulators](https://linuxblog.io/linux-terminal-emulators/).

The name terminal emulator may sound odd, but it comes from early days —before computers— when TTYs (teletypewriters) were _used to distribute stock prices over long distances in realtime_ [^1]. These devices evolved into CRT-and-keyboard terminals connected to time-shared computers, where the mainframe managed text manipulation like backspacing and cursor movement. Unix later abstracted these roles into software, enabling both physical and "pseudo-terminals" to interact with the system.

#### The shell
The shell, on the other hand is a process that you run on your terminal and there are many flavours such as bash, zsh, sh, etc.
However, the terminal emulator architecture allows any program/process to be bound to the terminal such as ssh, tmux, language REPLs, etc. The shell basically receives input (commands) from the terminal emulator, processes them and send back results that can come back formatted with, guess what? ANSI escape sequences to format text color, style, positioning, etc.

#### The kernel
This is the Linux OS kernel and here is where all the black magic happens to allow the terminal emulator to connect with any process such as bash, ssh, tmux, etc. This is the hardest part to understand, at least it was for me.

## Connecting the dots

When the terminal emulator starts, it creates a pseudo-terminal (each new session, new tab creates one) which is represented by the following files (actually they are character devices on Linux -- not physical devices though):

| Path              | Description                                                                 |
|-------------------|-----------------------------------------------------------------------------|
| `/dev/pts/ptmx`   | A file/device to which the terminal emulator gets a handler that it uses to read and write to. |
| `/dev/pts/N`      | Where N is a number bound to a process responding to the terminal emulator (e.g., bash). |

The terminal emulator will then write to `ptmx` in order to send commands to the shell and will read from it in order to parse and display the output (yes, the shell will send formatted text using the ANSI escape sequence previously described). On the other hand, the shell will interact with the corresponding `/dev/pts/N` via `stdin`, `stdout` and `stderr` communication channels. 

From the official docs, the PTY side that handles `ptmx` is the master and the PTY side that is bound with the process (e.g. bash) is known as the slave. I will keep these terms as bad as they are just to avoid cognitive load in case one has to map names between what is written here and what is written on Linux man pages and official docs. Sorry about that.

Think of the PTY as an abstract entity that is made of physical files. It is actually defined as code in the Linux Kernel which is what provides functions like `ptyfork`, `ptyopen` and more to manipulate PTY. The source code for the PTY can be found at /`drivers/tty/pty.c` under [Linux Kernel GitHub repo](https://github.com/torvalds/linux/tree/479058002c32b77acac43e883b92174e22c4be2d).

### A short experiment with PTY

As an experiment, if you type tty on the terminal, you will get the corresponding `/dev/pts/N` file, e.g. `/dev/pts/19`. Then if I go to another terminal and `cat /dev/pts/19`, whatever I type on the previous terminal gets printed by cat output.

## Expanding the mental model

Now I am ready to improve the diagram with a few more pieces:



The PTY master, typically accessed by the terminal emulator via `/dev/ptmx`, receives data written by the emulator. This data is handled by the TTY subsystem in the kernel, where the TTY driver manages the low-level I/O logic and interacts with the **line discipline**, which processes the data (e.g., echoing, buffering, signal generation). On the other end, the PTY slave (e.g., /`dev/pts/N`) is connected to a process like bash, which communicates through standard input and output. The kernel links both ends, making the shell believe it's connected to a real physical terminal.


## References

- [Terminal anatomy](https://poor.dev/blog/terminal-anatomy/)
- [The TTY demystified](https://www.linusakesson.net/programming/tty/)
- [Build your own Command Line with ANSI escape codes](https://www.lihaoyi.com/post/BuildyourownCommandLinewithANSIescapecodes.html)
- [The Secret Rules of the Terminal](https://jvns.ca/blog/2025/06/24/new-zine--the-secret-rules-of-the-terminal/)
- [List of 30+ Linux Terminal Emulators](https://linuxblog.io/linux-terminal-emulators/)
- [IBM - Time-Sharing](https://www.ibm.com/history/time-sharing)

## Footnotes

[^1]: [The TTY Demystified](https://www.linusakesson.net/programming/tty/) (see history section)