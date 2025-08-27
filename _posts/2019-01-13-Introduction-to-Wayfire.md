---
layout: post
title: Introduction to Wayfire
---

Hello all, I've recently been spending a lot of time developing [Wayfire](https://github.com/WayfireWM/wayfire), so with the help of Drew DeVault (a huge thanks for setting up the blog) I decided to start telling more about it.
<!--more-->

So what is Wayfire? Wayfire is a 3D floating wayland compositor, utilizing [wlroots](https://github.com/swaywm/wlroots). For those of you unfamiliar with wayland, a wayland compositor is similar to compositing window managers in the X11 world. It is that one piece of software that coordinates all of your input and output devices and manages all of your opened applications.

<iframe
  width="640"
  height="440"
  src="https://www.youtube-nocookie.com/embed/videoseries?list=PLb7YRKEhWEBUIoT-a29UoJW9mhfzjpNle"
  frameborder="0"
  allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture"
  allowfullscreen
></iframe>

# Why Wayfire?

You might wonder, why start yet another project? Why not use the already existing GNOME, KDE, Enlightenment (all of which support wayland to at least some extent)?

In short: Because there is a need of a common base to build a lightweight, but fully-functional wayland-based DE with minimal effort.

The *long* answer: because not everybody is willing to use a huge and resource-intensive (and for some people, bloated) DE. And because creating a compositor from scratch is not as easy as it may seem. Actually, most of what the Xserver used to do is the compositors' job now. That's why porting DEs to wayland is a challenge.

Having each DE/X window manager write their own compositor would mean a lot of useless, duplicated effort, not to mention most of them just don't have the resources for such a huge undertaking.  [wlroots](https://github.com/swaywm/wlroots) actually does a great job in this respect - it provides a very flexible base for building a wayland compositor, abstracting away many of the low-level details. However, it still cannot make writing a compositor too simple, and for a good reason: there is a lot of desktop-specific functionality which just can't go into a generic library.

Here's the place where Wayfire fits in: it aims to be a base on which a lightweight wayland DEs can be built. By being split into plugins (which mostly control the appearance/window-management part) and core (providing rendering, workspaces, etc) Wayfire makes it possible to build a DE with minimal effort. Since the beginning of the project, Wayfire's goals have also expanded to provide a rather minimalistic (but functional!) desktop environment.

# What are some of the best features Wayfire has?

1. Wayfire is heavily inspired by the Compiz window manager, and one of the implicit goals is to make your desktop look as nice as possible. So, Wayfire supports a lot of eye-candy: Cube, Expo, Wobbly windows, Animations(Burn, Zoom, Fade), 3D window switcher and more. See the [demo videos](https://www.youtube.com/watch?v=Ban7wspkrNQ&t=0s&index=2&list=PLb7YRKEhWEBUIoT-a29UoJW9mhfzjpNle) if you haven't!

2. Wayfire is very modular. You use almost any combination of plugins you want to. You can choose the panel/background/etc clients (unlike GNOME, where you're stuck with the built-in panel). By combining such "small" programs, and possibly enhancing Wayfire with your own plugins, you can use it to easily build a custom DE suited to your preferences.

3. Wayfire supports fully independent outputs. This means, each output has its own set of workspaces, windows, etc. You can play a movie on one output, and continue working on another, as usual (switching windows, workspaces, ...). Nothing you do on the second output (short of `kill`) won't affect the player on the first.

4. Wayfire is almost fully dynamically configured: Every modification you make to the config file is applied on-the-fly, without restarting the compositor and losing your opened applications.

# Conclusion

Wayfire has been in the works for several years now, so although I've been the only developer working on it for most of that time, I think it is starting to approach some kind of maturity. So, go ahead, give it a try! See the [README.md](https://github.com/WayfireWM/wayfire/blob/master/README.md) for more information on how to build and install Wayfire, and consult the Wiki([configuration](https://github.com/WayfireWM/wayfire/wiki/Configuration), [core options](https://github.com/WayfireWM/wayfire/wiki/Core-options), [external tools](https://github.com/WayfireWM/wayfire/wiki/External-tools)) in order to get Wayfire in a fully-functional state. Reach out in [Matrix](https://matrix.to/#/#wayfire:matrix.org) or `#wayfire` at `libera.chat` if you encounter problems or want to develop on top of Wayfire. Any feedback is appreciated!
