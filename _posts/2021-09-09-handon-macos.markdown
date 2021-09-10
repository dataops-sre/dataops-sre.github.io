---
title:  "Mac is POSIX they said, 'It shouldn't be too bad!' they said..."
date:   2021-09-09 10:36:37 +0100
categories: MacOS Docker
tags:
  - Devops
  - MacOS
  - Docker
---

## Introduction
Claim: This article is not all about complaining, I hope to provide some usefull tips for people working with MacOS.

I joined a new company : [miro](https://miro.com/app/dashboard/) this month, new start, new adventure(sort of, it's not like I am going to north pole though). First thing is to order hardwares, naturally I asked if I may have a linux PC, sadly, the answer is negative. Engineer department can only have Apple. I know nowadays Mac just become a standard among developpers, you appears almost as an amateur if you dare to show up to a meetup with a PC. Me, old school, use Archlinux for years, undeniably have some prejudice against Mac, Nevertheless, the internal IT guy told me, "Mac is POSIX system, look, they have terminal and bash, it shouldn't be too bad!", I was like, hmm, well so many developpers, include my past colleagues use it, it can't be that bad, right? okay I don't like Apple as a company, but let's not be so narrow minded, give it a try.

## Small glitches here and there
I recieved a MacBook Pro 16 two weeks prior to my start date, the laptop looks good, nice design, the whole aluminium coating inside out, My XPS laptop looks like trash now with its cheap plastic inside, I never understand why Dell have Alu coating outside and try to save some dollars to have plastic inside.. Sound quality is undeniably way better than any of my old PC laptop, AND finally I felt as a real engineer with `Pro` seal on my forehead, with this thought, I farted a little, and immediately realized even my fart smells better with Apple on my hand. is it..?! is it so called the magic of Apple?!! I was stunned.

It's a happy start, then I set up my working environment, it is quite minimal, a browser, terminal + tmux + vim, sublime text, usually I just use these 3 components for working. As I am a poor coder, I don't have a multi-screens setup, I like to have my terminal transparent so I can see codes in sublime or cat videos playing in the background while running some docker builds, so I don't feel needs to have a Tiling window manager. MacOS's [Rectangle](https://github.com/rxhanson/Rectangle) is enough for my humble need. I miss [rofi](https://github.com/davatorium/rofi) on linux, as a good samaritan, I decided just to live with spotlight.

Nothing much to say about setting up browser and sublime text. For terminal, I used to have Termite, I am super happy to learn its successor [alacritty](https://github.com/alacritty/alacritty) support MacOS plateform. So just import my linux tmux/zsh/vim dotfiles then everything would work as expected right? well, of course not, Mac has its own key mapping! I blind type with french PC keyboard, some extra configurations are needed. Here's list of glitches on Alacritty(with french PC keyboard):

* Special characters such as ~/#,{},[],@,^ etc do not work
* Ctl - left/right to jump one word left/right on terminal do not work
* vim keybindings on tmux vi-mode do not work with [tmux configuration](https://github.com/gpakosz/.tmux)
* `ls --color=auto` and many other commands does not work anymore.

All issues get resolved with some googling.

Resolve first two issues by adding following lines in `~/.config/alacritty/alacritty.yml`
``` yaml
# for french pc keyboard see https://github.com/alacritty/alacritty/wiki/Keyboard-mappings#macos 
key_bindings:
  - { key: 19,             mods: Alt,                       chars: "~"                     }
  - { key: 20,             mods: Alt,                       chars: "#"                     }
  - { key: 21,             mods: Alt,                       chars: "{"                     }
  - { key: 22,             mods: Alt,                       chars: "|"                     }
  - { key: 23,             mods: Alt,                       chars: "["                     }
  - { key: 24,             mods: Alt,                       chars: "}"                     }
  - { key: 25,             mods: Alt,                       chars: "^"                     }
  - { key: 26,             mods: Alt,                       chars: "`"                     }
  - { key: 27,             mods: Alt,                       chars: "]"                     }
  - { key: 28,             mods: Alt,                       chars: "\\"                    }
  - { key: 29,             mods: Alt,                       chars: "@"                     }
 # for ctl-left/right, now become option-left/right see https://github.com/alacritty/alacritty/issues/1408
  - { key: Right,          mods: Alt,                       chars: "\x1BF"                 }
  - { key: Left,           mods: Alt,                       chars: "\x1BB"                 }%

``` 

Third issue is actually a mistake of my part, just need to set the `EDITOR` env var to vim, it was automatically setted as such in my linux.

Fourth issue is common, MacOS use BSD coreutils, linux use GNU coreutils, there are some minor differences, if you want to have similar cli experience as in linux, checkout [linuxify](https://github.com/fabiomaia/linuxify) repo, they did a great job to bring Gnu/Linux experience to MacOS.

If you miss your i3w set up on linux, I found a great setup [article](https://cbrgm.net/post/2021-05-5-setup-macos/), have a look!

## Docker nightmare
Everything is fine until I start to work with docker and minikube. First thing I noticed is -- slow ! For example, This blog is a jekyll project, locally built and run with [docker](https://github.com/dataops-sre/dataops-sre.github.io/blob/master/Taskfile.yml#L7), first time run download few ruby gems, it took almost 4 minutes to launch the local server in docker, I don't remember taking that long in my ugly XPS.. Later I lauched minikube, few docker-compose and my Mac is simply on fire, CPU temperature raise to 70/80, fan ran as crazy, I am pretty sure that I can cook a nice omlette on my MAC touchpad.. Suddenly I have flashback to the moment when I heard Macbooks of my colleague's fan spin as crazy, I thought they were mining bitcoins, the whole time they had been actually working with docker/minikube !!

MacOS/Windows docker desktop uses virtual machine technology, it spins up a linux VM as docker engine server and set the host machine docker client points to vm's docker socket. As all VM technology, shared filesystem perform super bad. Docker in linux, on the other hand, shared filesystem work natively. Out of curiousity I compared time to build/run this blog under MacOS(shared volume) to under a linux vm(native) : cpu 3m33.81s vs 1m15.151s almost 300% difference.. this issue is quite known in MacOS, many attempts and tools are developped try to [beat the problem](https://medium.com/homullus/beating-some-performance-into-docker-for-mac-f5d1e732032c). what a waste of people's time and energy.. Eventhough, they are just workaround.. There's nothing much you can do about it.

## Afterword  
I think MacBook is a quite good laptop in term of design and functionality, sold way over its real price by scammer sellmens, but if it's your company that they rip off, I think you should be fine. 

As an engineer, if you never deal with docker or kubernetes or you work with video/sound editting and your native tools are in Mac, You should stick with it.

For developpers who always work with docker on MacOS, you are basically climbing Everest everyday with an extra 200kg bag, you deserve to experience quiet and cool computer with docker/kubernetes local run.

For developpers with linux, don't change and I envy you.
