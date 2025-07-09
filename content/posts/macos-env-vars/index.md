---
title: Setting system-wide Environment Variables on MacOS
date: 2025-07-09
draft: false
summary: "A little documented trick to set environment variables globally on a mac"
tags: ["macos", "tips"]
---

{{<lead>}}
A little documented trick to set environment variables globally on a mac
{{</lead>}}

This will be a nice quick one, something I've discovered recently that is barely documented!

## Problem

You need to set environment variables *at the system level* on MacOS, so that **all** processes running after boot up will have access to those variables.

To make the point concrete, let's use a motivating example: `XDG_CONFIG_HOME`

The value of this environment variable points to a particular directory that applications can use to store settings and other configuration files. Most people end up setting
this to `~/.config` (which is the default on Linux for this variable anyway). So an application called "blah" would store it's config under `~/.config/blah/`

{{<alert "lightbulb">}}
If you don't set this variable, applications running on MacOS default to storing these things under `~/Library/Application Support/<application>`
{{</alert>}}

I like the `~/.config` convention far more than `~/Library/Application Support` and it fits well with what a lot of command line programs and other dev tools assume as defaults
so I want to set my `XDG_CONFIG_HOME` to this for my mac.

## The Solution that kinda works

You *could* just put the following in your shell startup file like `~/.zshrc` or `~/.config/fish/config.fish` or choose your favourite shell's path:

```shell
# ~/.zshrc
# $HOME expands to your user home dir e.g. /Users/<you>/ or ~/
export XDG_CONFIG_HOME=$HOME/.config
```

And you could get away with this for most things, it would work most of the time. Certainly any programs you run from within a terminal session (like most command line dev tools)
would respect this value now.

But what if you want `XDG_CONFIG_HOME` to be set for *your shell itself*... or any other programs that run outside of a terminal session where `~/.zshrc` hasn't been sourced yet? 🤔

## Why only "kinda"

This is where I was stuck!

I use [nushell] (it's actually great you should check it out if you haven't yet!) and [nushell] actually looks for `XDG_CONFIG_HOME` *before* it starts up to
source it's config (stored by default on macs under `~/Library/Application Support/nushell`)

So setting `XDG_CONFIG_HOME` in `~/Library/Application Support/nushell/env.nu` would work for everything started in a terminal session *after* [nushell] starts, but crucially **not** [nushell]
itself

{{<alert "circle-info">}}
Quick sidebar on *why* I wanted this. I manage my [dotfiles](https://github.com/FollowTheProcess/dotfiles) from a repo with symlinks managed by [GNU Stow](https://www.gnu.org/software/stow/), which requires the structure of the repo to replicate the structure
of your desired home directory. And it's so much more convenient to have everything under `~/.config` rather than scattered across different locations
{{</alert>}}

There's even a note on this in the docs in [Additional Startup Configuration]:

> As discussed below, variables in this section must be set before Nushell is launched.

So I *had* to find a way to set `XDG_CONFIG_HOME` *before* my shell (or any other user-space program for that matter) is launched

## The Real Solution

Turns out there's a program in MacOS called `launchctl`, which allows us to programmatically interact with `launchd`, a "System wide and per-user daemon/agent manager".

This is how we can get MacOS to do things for us on boot, user-login etc. Crucially, buried in the man page of `launchctl`... is the `setenv` subcommand:

```plaintext
setenv key value
    Specify an environment variable to be set on all future processes
    launched by launchd in the caller's context.
```

This is getting somewhere! Now we need to get MacOS to run this *for us* on startup.

We do this by putting `plist` files in the correct places to make things happen. The place we want for this task is `~/Library/LaunchAgents`, and the file is pretty simple!

```xml
<!-- ~/Library/LaunchAgents/setenv.XDG_CONFIG_HOME.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
  <plist version="1.0">
  <dict>
  <key>Label</key>
  <string>setenv.XDG_CONFIG_HOME</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/launchctl</string>
    <string>setenv</string>
    <string>XDG_CONFIG_HOME</string>
    <string>/Users/tomfleet/.config</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
</dict>
</plist>
```

You can see our `launchctl setenv` command there, along with the `RunAtLoad` key set to `true` (this is the crucial bit 👌🏻).

We save this file in `~/Library/LaunchAgents/setenv.XDG_CONFIG_HOME.plist` and our job is done! On a restart, `launchctl` will arrange this to be run for us and `XDG_CONFIG_HOME` will point to the right place for all user-space
programs 🎉

If you look at my [dotfiles] repo, I've even got this bit managed with GNU Stow so it's automatically put in the right place 🤓 Now if I evaluate `$nu.default-config-dir` it
gives me:

```shell
❯ $nu.default-config-dir
/Users/tomfleet/.config/nushell
```

Success! ✅

[nushell]: https://www.nushell.sh
[Additional Startup Configuration]: https://www.nushell.sh/book/configuration.html#additional-startup-configuration
[dotfiles]: https://github.com/FollowTheProcess/dotfiles
