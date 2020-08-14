---
layout: post
title: Tmux and Vim
description: Configure Tmux with VIM keys. Works on Windows and Linux
---

Tmux is one of the most useful applications to use on Linux, 
and if you read my [last post]({{ site.baseurl }}/Tmux-on-Windows/), 
you now know how to get Tmux up and running on Windows too.
As a VIM user (addict), I need to use the VIM keys everywhere possible, 
and in this post, we will configure Tmux to do exactly that.

## Prerequisites

----

You will need Tmux installed and working.

**Linux**: Install as you would any other package

```bash
# Example for Debian based distros
sudo apt install tmux
```

**Windows**: Follow the guide in my 
[previous post]({{ site.baseurl }}/Tmux-on-Windows/) and then continue from here.


### Disclaimer

----

As a disclaimer, this guide will do more than just add VIM keys to Tmux;
it will change the default Tmux prefix `Ctrl + b` to the backtick '```'. 
The new configuration makes use of Tmux plugins that make Tmux even better.
I would recommend backing up your existing Tmux.conf file before continuing:

```bash
# Backup your existing config if necessary
cp ~/.tmux.conf ~/.tmux.conf-backup
```

## Set the tmux.conf file

----

I have a pre-set Tmux.conf file stored as a Gist in GitHub for Linux and Windows.

[Linux Tmux config](https://gist.github.com/briansworth/9da664f15e51ca48ab5d7a0ac4a73cb2)

Run the following to quickly install it on Linux.

```bash
TMUX_CONF=~/.tmux.conf
GIST_URL=https://gist.githubusercontent.com/briansworth/9da664f15e51ca48ab5d7a0ac4a73cb2/raw/ea2f7da743887e345dbddeccd7eedb1fd2271ba6/.tmux.conf

# Backup the existing tmux.conf if it exists before overwriting
if [ -f $TMUX_CONF ]; then
  cp $TMUX_CONF ~/.tmux.conf.bak_CodeAndKeep
fi
curl $GIST_URL -o $TMUX_CONF
```

[Windows Tmux config](https://gist.github.com/briansworth/b12f28f9a9e7bd42d9d7b67160079188)

Run the following to quickly install it on Windows.

*NOTE: You will need to be in a bash shell on your WSL Distro*

```bash
TMUX_CONF=~/.tmux.conf
GIST_URL=https://gist.githubusercontent.com/briansworth/b12f28f9a9e7bd42d9d7b67160079188/raw/61bcca39aed0b68fb2891c26ac4bf92c20733bcd/windows.tmux.conf

# Backup the existing tmux.conf if it exists before overwriting
if [ -f $TMUX_CONF ]; then
  cp $TMUX_CONF ~/.tmux.conf.bak_CodeAndKeep
fi
curl $GIST_URL -o $TMUX_CONF
```


PowerLines tmux.conf:
https://gist.github.com/briansworth/bd3d5d44b7e23982edf1847214ab1551

tmux.conf
https://gist.github.com/briansworth/9da664f15e51ca48ab5d7a0ac4a73cb2

Windows tmux.conf
https://gist.github.com/briansworth/b12f28f9a9e7bd42d9d7b67160079188

