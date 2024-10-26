---
layout: post
title:  "Seting up Nixos on Dell XPS 16 (with working webcam)"
date:   2024-10-26 14:30:00 +0000
categories: nixos dell_xp_16
---

I recently started a new job and got to pick a laptop to use as my daily driver. Somewhat naively picked the Dell XPS 16 since I had previously heard that Dell had ok support for Linux with their Dell XPS 13. That was probably a few years ago. Now the machine itself is of high quality and a solid build, but once I got it I immediately ran into issues with sound and webcam not working.

I was originally using Xubuntu 24.04 and with a bit of tinkering I did get the sound to work back in July by switching kernel version and some additional dark arts I don't fully recall. Howeer, this weekend I had a few hours, and I've been curious about seting up NixOS properly, so now was the time to make it work! With NixOS I figured I could get the basics set up and then not have to worry too much if my tinkering failed since I could always just go back to a working previous state. 

Now I am by no means an expert on neither Linux nor NixOS, but I did manage to set things up so the webcam work both in Firefox, Slack and using `ffplay` which I used for testing.

So what did I do? I started with a plain install of NixOS using the main ISO. I made sure that I left my original Xubuntu unchanged in case things didn't pan out. Then after booting up I 
initially sett up `flakes`, by following [vimjoyer's starter config guide](https://github.com/vimjoyer/flake-starter-config). Once that was set up I updated my kernel to the latest version (6.11.5 at the time of writing) by adding a single line to my `/etc/nixos/configuration.nix`:

```
boot.kernelPackages = pkgs.linuxPackages_latest;
```

This could probably be pegged to `pkgs.linuxPackages_6_11` in case one wants more controll over changes to the kernel. 

I also added the following system packages to facilitate testing:

```
environment.systemPackages = with pkgs; [
    webcamoid
    v4l-utils
];
``` 

Now running `v4l2-ctl --list-devices` only displayed a long list of ipu6 "phantom"(?) devices that none of them worked with neither ffplay (temporarily added via `nix-shell -p ffmpeg_6`) nor webcamoid. I still had some hardware config to do, so towards the end of my `/etc/nixos/hardware-configuration.nix` I added:

```
hardware.ipu6.enable = true;
hardware.ipu6.platform = "ipu6epmtl";
```

After invoking
```
$ sudo nixos-rebuild boot --flake /etc/nixos/#default
```

I rebooted the system and webcamoid, firefox, and slack could all find and utilize the built in webcam! I'm omitting a bunch failed attempts (e.g. using "ipu6ep" as the platform for ipu6), but all in all I have to say it was much less painfull than trying the same thing on my original Xubuntu install (which I still have on a separate partition until I've migrated all the settings to the NixOS install). There I'd be much more worried of wrecking some setting unintentionally. If I'd used a filesystem with snapshots I could have tried using that as a rollback mechanism, but alas that's not how I'd set it up. Besides, I did want to learn more NixOS so all well and good!

So if you have a Dell XPS 16 and have had trouble getting the webcam to work maybe this will help you.

Cherio!
