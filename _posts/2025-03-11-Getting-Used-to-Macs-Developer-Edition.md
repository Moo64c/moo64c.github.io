---
title:  "Getting Used to Macs, Developer Edition"
author: "Amir Arbel"
categories: ["Devlog", "how-tos"]
date: "2025-03-11"
tags: [MacOS, software-development]
excerpt: Switching to MacOS: The tooling developers need when using MacOS. The good, the bad, and the Terminal Emulators.
---

Switching to Mac for the first time is a big drain on a developer's productivity.  After almost six years, I've learned a whole lot - between grievances to workarounds, tips, tricks and a bit of software. I decided it is time to write some of it down.

You might have run into this page because you are making the switch yourself, probably because you are forced to do so by an employer, or you are (sanely) thinking to yourself "why not join a closed-end ecosystem of really expensive products?".

Whatever the reason, here's s a set of things to know, software and general principals that I use to be as productive as possible with MacOS. These fit my use case, which is mostly keyboard-centric.  I tend to switch parts of it from time to time when I feel its no longer comfortable or slows me down.

With that introduction out of the way, let's start with some gripes I faced when I switched:

# The Painful Switch
Coming from Windows or Linux, the biggest difference you will face immediately - most of the keyboard shortcuts you are used to, now use the `command` key (`cmd`) instead of `ctrl`. `cmd` is actually the `windows` key in pretty much any keyboard, and difference in position is physically tiny but mentally huge.

`cmd` is used for the shortcuts the muscle memory points to `ctrl`, like copy/pasting (`cmd+c/cmd+v` instead of `ctrl+c/ctrl+v`).  Whatever you were used to do with `ctrl` (except maybe escape sequences), you need to re-learn as `cmd`. Another good example would be `cmd+p` instead of `ctrl+p` for the VSCode "jump to file" shortcut.

If you are coming from Linux, (or already used to a lot of terminal usage) this mostly allows for more shortcuts to be available inside any terminal emulator (and makes a lot more sense than what Windows is doing for copy pasting in the terminal). I believe having `cmd+c/v` instead of the dreaded `ctrl+shift+c/v` for terminals in Linux make a bit more sense, but I'm already biased.

This was actually the hardest switch for me, due to muscle memory built over the years, and avoiding the Mac keyboard layout where possible (using Windows-like keyboards). Even working on a Linux machine for a day and coming back to Mac is still painful - for both sides - after almost six years.

## Apple Hates Users as a Policy (or: Using the Finder File Manager)
The second most painful switch was Finder, the file manager. Only five years into working with MacOS, I learned that Finder has `cmd+shift+g` to jump to any folder is all I needed to make it less painful, together with `cmd+1/2/3/4` to change the display layout. But that leads me to another good point about MacOS for new comers:

 ## ... And Hides Keyboard Shortcuts Where Possible
**You have to manually find the keyboard shortcuts you need.**
Finding out about keyboard shortcuts in MacOS (without online help) is one of two: opening up menus and looking for the action you need, and if it has a shortcut - it will be greyed out in the menu item to the right, or just going through every possible settings tab and pane you can navigate into (and there are many).

There's no one settings pane and no interactive way to learn what keyboard shortcuts exist. The easier method is to [look them up online](https://support.apple.com/en-us/102650). Changing anything from at the system-wide level is a straight out pain. For example, try to change the spotlight shortcut (`cmd+space`) and see what I mean.

Finder, iTerm2 (later on in this guide), and system-wide shortcuts are all hidden in multiple menus and option panes. Finding out about shortcuts is tricky enough, and it feels like everything is being done to hide the _useful_ ones as much as possible.

# Enough Pain Points. Some good stuff:

## Spotlight (but really, [Raycast](https://www.raycast.com/) )
Spotlight is the best productivity tool you would get from switching to MacOS. You can get the same thing in [Linux](https://ulauncher.io/) and [Windows](https://learn.microsoft.com/en-us/windows/powertoys/)  with additional tooling, and in MacOS  [Raycast](https://www.raycast.com/)  expand on the Spotlight concept, but the idea is the same:
`cmd+space` and type anything you want. A program, a command, a setting's name, a webpage - and you are there. Calculator? Built-in. Currency conversions, move a window to the left two thirds of the screen, quick searches, plugins for searching through Jira or Github.

One-liner for anything you need. Powerful, fast, and absolutely necessary for getting anything done with MacOS. Having it around pretty much cancels the need for launcher bar at the bottom. It is a big change from going through the start menu in Windows, but makes things so much easier. With Aerospace (discussed later), a lot of functionality that forced you to use the mouse is eliminated, which I love.

## The MacOS Package Manager for Developers - [Homebrew](https://brew.sh/)
Package managers are nothing new, and Homebrew isn't the best one that exists. But in Macs anything development-y would require you to run `brew install`  for something.

Homebrew is actually a very helpful tool. It updates only when you ask it to, has a thriving community, and most (free or open sourced) software for Macs will have a one-liner to install them highlighted on the main product or readme page.

## Hipster Terminal Emulator, or a Useful One?
Macs ship with *Terminal*, which fulfill the required ability of installing something else. Unless you are living under a rock, gone are the days terminals are sluggish, ASCII only command line runners. They became a [pretty serious pieces of technology](https://github.com/ghostty-org/ghostty) with cross-platform native support and GPU acceleration, performant and beautiful, all while trying their best not to be something that slows you down at all.

Ask any hipster and they'll point you to any of the *cutting-edge, blazingly fast* category of terminal emulators, there's a whole lot of options. [Ghostty](https://ghostty.org), [Kitty](https://sw.kovidgoyal.net/kitty/), [Alacritty](https://alacritty.org) and others (some also available in other OS). The option I've been given when I was just starting out with Macs was [iTerm2](https://iterm2.com). iTerm2 is the most feature-rich as far as I can tell - snippets to password managers, custom keybindings (that need a bunch of work to make them reasonable) and much much more.

A terminal emulator is something developers spend a lot of time using, and replacing it is pretty difficult. A missing feature or unexpected behavior or rendering can just send you running back to the familiar one. Make sure you have the features you need in the emulator you choose and it makes sense to you. I've been with iTerm2 since the beginning and couldn't make a change (so far).

What I'm trying to say here, and will say again when it comes to IDEs - it comes to personal preference. If you never touched a terminal emulator before and will miss nothing I'd suggest taking Ghostty, as it looks the most promising down the line. If you are like me and no longer remember your SSH key's password and need a in-terminal password manger one keyboard shortcut away, iTerm2 is probably the way to go (or maybe set up your password manager as an extension in Raycast, and use it from there).

## A [Window Manager](https://github.com/nikitabobko/AeroSpace) - Aerospace
One thing that MacOS sucks at is the `alt+tab` to switch between _windows_. But MacOS switches between _apps_ instead (and it's `cmd+tab`). To switch between different windows in the same app, you need to ```cmd+` ``` like that makes any sense. Why is this a problem?

Say you are just developing away like one does, and end up having to use 2 different windows of VSCode, say one for a backend of a project and one for the frontend. Switching between them and a browser window becomes a pain really, really fast, since you need to use both `cmd+tab` and ```cmd+` ``` to reach the correct VSCode window.

_Aerospace_ makes that switch quite easy - Backend gets the "b" workspace with `shift+option+b` and Frontend gets the "f" workspace with `shift+option+f`. Now jumping between them is just `option+b` and `option+f`.

Alternatively, we can put both in the same workspace, and either have them stack on top of one another (only one visible at a time) or split the screen left/right or top/bottom, moving between them with vim-style movements (`super+h/j/k/l`).

The other option is to install [something to replicate](https://alt-tab-macos.netlify.app) the known `alt+tab` behavior from Windows or Linux. To me it makes little sense, because I already invested myself into how Macs behave, and made the jump over to Aerospace. `alt+tab` is dead to me.

Also glossed over this before, but the `option` key is just the `alt` key. Don't let Tim Cook fool you.

# (Mostly) Platform Agnostic Things:

## Note Taking, to the Max
If you are not noting down things while you work, you should start. Trying to keep every little detail in your mind while developing, chatting, having meetings and designing software is a feat too great for me. Luckily, computers exist and they can store a lot of data relatively easily. Who would've guessed?

[Obsidian](obsidian.md) fills the niche for me. Macs come with Apple Notes, a with a very clean UI and a left pane for switching between notes using a small preview, but I find it hard to keep track of things there. [Notion](https://www.notion.com) is also a common option. The killer feature for Obsidian is linking between notes, a feature not unique to Obsidian but it makes the connection easy and flowing, and also has a relevant Neovim extension). They also [recently removed](https://help.obsidian.md/teams/license) the requirement to buy a commercial license if, say, you have a job or make money in any way, and write anything about it in Obsidian.

## Any Browser (But Safari)
Another one of the flavor-specific item is a browser. Unless you are using a very old or specific browser, you can use the same browser you have used before. But avoid Safari, there is really no reason to use it (anymore than Microsoft's Edge). It is not reliably supported, and Safari has only the downsides of the Mac ecosystem, without real upside.

Maybe take the time to switch to a more [privacy-focused](https://vivaldi.com) browser (or at least one that claims to be?). As a [previous user](https://connect.mozilla.org/t5/discussions/information-about-the-new-terms-of-use-and-updated-privacy/m-p/87735) of [Firefox](https://www.mozilla.org/en-US/firefox/new/), it and other non-chromium browsers tend to have more difficulties in certain sites.

Whatever you choose (hopefully not Chrome) - make sure to have [uBlock Origin](https://ublockorigin.com) has an extension for proper sanity. Youtube enthusiasts might love [SponsorBlock](https://sponsor.ajay.app) (pro tip: "skip all non-music segments" in the settings makes all the difference). And a [password manager](https://bitwarden.com/download/) since nobody should remember more than two or three passwords in their lives (With Bitwarden -`cmd+shift+l` for auto-fill).

## What About an IDE? What about Docker?
Like browsers, modern (and sane) IDEs no longer tie themselves to one operating system. Choose what you are familiar with, and keep looking for better options when they come around. I have been working on **VSCode** with a whole lot of extensions for years - and from time to time try my luck with a bit of [Lazy](https://github.com/folke/lazy.nvim) [Neovim](https://neovim.io) (which takes a bit time to learn).

As for containers, docker-desktop, Rancher or Podman will work just the same as on Linux. Remember M-series Macs use a different architecture (ARM/AARCH) so you might need to add `platform: linux/x86_64` to your services in the relevant compose files to make some containers work.

Hope this was helpful - and good luck!
