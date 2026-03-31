---
title: "macOS is POSIX, they said. “It shouldn’t be too bad,” they said..."
date: 2021-09-09 10:36:37 +0100
categories: macOS Docker Developer-Experience
tags:
  - DevOps
  - SRE
  - macOS
  - Docker
  - Developer Experience
---


Disclaimer: this article is not only about complaining. I also hope to share a few useful tips for people working with MacOS.

I joined a new company, [Miro](https://miro.com/app/dashboard/), this month. New start, new adventure — sort of, it is not like I am going to the North Pole though. The first thing to do was order hardware. Naturally, I asked whether I could get a Linux laptop. Sadly, the answer was no. Engineers could only get Apple machines.

I know that nowadays Macs have more or less become the standard among developers. You almost look like an amateur if you dare to show up at a meetup with a PC. Me, old school, I have used Arch Linux for years, so I undeniably had some prejudice against Macs. Still, the internal IT guy told me, “Mac is a POSIX system. Look, it has a terminal and bash. It shouldn’t be too bad!” I thought, well, so many developers use it — it cannot be that bad, right? Fine. I do not like Apple as a company, but let’s not be too narrow-minded. Let’s give it a try.

## Small glitches here and there

I received a 16-inch MacBook Pro two weeks before my start date. The laptop looked good: nice design, aluminum everywhere inside and out. My XPS suddenly looked like trash with its cheap plastic interior. I never understood why Dell put aluminum on the outside and then tried to save a few dollars with plastic on the inside. The sound quality was also undeniably better than on any of my old PC laptops.

And finally, I felt like a real engineer, with the `Pro` seal stamped onto my forehead. With that thought, I farted a little, and immediately realized even my fart smelled better with Apple in my hand. Is this... is this the famous Apple magic?! I was stunned.

It was a happy start. Then I began setting up my working environment, which is fairly minimal: a browser, terminal + tmux + vim, and Sublime Text. Usually, I only need those three components to work. As a poor coder, I do not have a multi-screen setup. I like to have my terminal transparent so I can see code in Sublime or cat videos running in the background while I launch Docker builds, so I do not really feel the need for a tiling window manager. On MacOS, [Rectangle](https://github.com/rxhanson/Rectangle) is enough for my humble needs. I do miss [rofi](https://github.com/davatorium/rofi) on Linux, but as a good Samaritan I decided to just live with Spotlight.

There is not much to say about setting up the browser and Sublime Text. For the terminal, I used to use Termite, and I was very happy to learn that its successor, [Alacritty](https://github.com/alacritty/alacritty), supports MacOS. So, I thought, I will just import my Linux tmux/zsh/vim dotfiles and everything will work as expected, right?

Well, of course not. MacOS has its own key mapping. I touch-type with a French PC keyboard, so a few extra configurations were needed. Here is the list of glitches I hit with Alacritty on macOS using a French PC keyboard:

- Special characters such as `~`, `#`, `{}`, `[]`, `@`, `^`, etc. do not work
- `Ctrl + Left/Right` to jump one word left or right in the terminal does not work
- Vim keybindings in tmux vi-mode do not work with the [tmux configuration](https://github.com/gpakosz/.tmux)
- `ls --color=auto` and many other commands do not work anymore

All of these issues were resolved with a bit of Googling.

## Alacritty fixes on macOS

The first two issues can be fixed by adding the following lines to `~/.config/alacritty/alacritty.yml`:

```yaml
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
  # for ctrl-left/right, now become option-left/right see https://github.com/alacritty/alacritty/issues/1408
  - { key: Right,          mods: Alt,                       chars: "\x1BF"                 }
  - { key: Left,           mods: Alt,                       chars: "\x1BB"                 }
```

The third issue was actually my own mistake. I just needed to set the `EDITOR` environment variable to `vim`. On Linux, it was already set automatically in my environment, so I never had to think about it.

The fourth issue is a common one. MacOS uses BSD coreutils, while Linux generally uses GNU coreutils. There are small differences here and there. If you want a CLI experience closer to Linux, check out the [linuxify](https://github.com/fabiomaia/linuxify) repository. They did a great job bringing a more GNU/Linux-like experience to macOS.

And if you miss your i3wm setup on Linux, I found a good [setup article](https://cbrgm.net/post/2021-05-5-setup-macos/) worth checking out.

## The Docker nightmare

Everything was fine until I started working with Docker and Minikube.

The first thing I noticed was: it is slow. For example, this blog is a Jekyll project, built and run locally with [Docker](https://github.com/dataops-sre/dataops-sre.github.io/blob/master/Taskfile.yml#L7). The first run had to download a few Ruby gems, and it took almost four minutes to launch the local server in Docker. I do not remember it taking that long on my ugly XPS.

Later I launched Minikube, a few `docker-compose` setups, and my Mac simply caught fire. CPU temperature went up to 70–80°C, the fan spun like crazy, and I am pretty sure I could have cooked a decent omelet on the Mac touchpad. Suddenly, I had a flashback to the times I heard my colleagues’ MacBooks spinning like jet engines. I thought they were mining Bitcoin. Turns out they had just been working with Docker and Minikube all along.

The reason is fairly simple. Docker Desktop on macOS and Windows relies on virtualization. It spins up a Linux VM as the Docker engine server, and the host machine’s Docker client talks to the VM’s Docker socket. As with most virtual machine setups, shared filesystem performance is terrible. On Linux, Docker runs natively, so filesystem sharing works much more naturally.

Out of curiosity, I compared the time needed to build and run this blog on macOS with shared volumes versus inside a Linux VM with native behavior: `3m33.81s` vs `1m15.151s` — almost a 300% difference. This issue is well known on macOS. Many attempts and tools have been developed to [mitigate the problem](https://medium.com/homullus/beating-some-performance-into-docker-for-mac-f5d1e732032c), but in the end most of them are only workarounds. There is not much you can do about the underlying architecture.

## Final thoughts

I think the MacBook is actually a pretty good laptop in terms of design and general usability, sold at a wildly inflated price by very talented salespeople. But if it is your company getting ripped off, you should be fine.

As an engineer, if you never deal with Docker or Kubernetes, or if you work with video or sound editing and your native tools live on MacOS, then yes, you should probably stick with it.

But for developers who work with Docker on macOS every day, you are basically climbing Everest daily with an extra 200 kg backpack. You deserve to experience what it feels like to have a quiet, cool computer while running Docker and Kubernetes locally.

And for developers still on Linux: do not change. I envy you.
