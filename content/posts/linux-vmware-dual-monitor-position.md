---
title: "Incorrect dual monitor positioning in VMWare / Virtualbox with X11 guests"
date: 2019-09-06T16:20:09+02:00
description: "For people with unconventional monitor positioning"
categories:
- linux
---

VMware Workstation has rich guest tooling and supports a range of graphical capabilities in guests, even X11 guests. I do a lot of work in a full-screen VMWare session, and I prefer to have side-by-side monitors with different orientations, which require careful positioning to line up nicely.

This works okay in the host OS, but VMware is a little special. At first everything seems fine -- moving the mouse cursor between displays indicates they seem to (magically!) be aligned correctly. However, in many setups, mouse clicks will be offset from where the mouse pointer actually is. After a little experimentation what I have determined is happening is this:

* The VMWare (or open-vm-tools, or whatever) X/input drivers are somehow rendering the mouse location based on the existing host monitor layout settings -- that is, correctly.
* Whatever you have running on top of X (KDE, XFCE, whatever), is NOT using the existing host monitor layout settings at all.

The result is that your clicks aren't going where you think they are. In the absence of a proper fix (I've seen reports of this issue that are _8 years old_), one possible solution is to carefully line up your display settings, using whatever tool available in your X environment, to _exactly match_ the layout of your Windows environment. The issue with this is, no matter how careful you are, you're still likely to have a mouse pointer that's at least a few pixels off from your click destination. The result of that is at best annoying, and at worst disastrous ("Are you sure you want to delete this folder?" "OK / Cancel").

Luckily, X has great multi monitor support in the form of xrandr. This allows you to specify absolutely pixel-perfect monitor layout, based on n displays and a "virtual" space which is the smallest possible box that combines them all. Here's an example of my personal setup:

```
0,0
  .................................
  .                  .-----------..
  .                  | Display 2 |.
  .                  |           |.
  ..----------------.|           |.
  .| Display 1      ||           |.
  .|                ||           |.
  .|                ||           |.
  .|                ||           |.
  .'----------------'|           |.
  .                  |           |.
  .                  '-----------'.
  .................................
                               4000,1440
```
That's two 2560x1440 monitors, the right-hand side in portrait onrientation. Giving me a "virtual" space of 0,0 to 4000,1440 (2560+1440 being the 4000). According to xrandr, my layout looks like this:

```
❯ xrandr --listmonitors
Monitors: 2
 0: +*Virtual1 2560/677x1440/381+0+620  Virtual1
 1: +Virtual2 1440/381x2560/677+2560+0  Virtual2
```

The last two pairs of numbers at the end there are of most interest to us: `0+620` and `2560+0` -- those tell us that `Virtual1` is offset 620 pixels down in the Y axis, and `Virtual2` is offset 2560 across in the X axis. Great! Now all we need to do is check how our guest OS is set up, and pump those exact figures into xrandr.

If you're unlucky enough to have Windows as a host OS, this isn't as easy as it seems, as Windows does _not_ have an equivalent of our xrandr command. Luckily, we do have Powershell, and using that we can access the huge `System.Windows.Forms` namespace which will allow you to find out pretty much anything about your displays if you know how to look.

Open a new Powershell session (I don't think this will work in Powershell Core, since we need to access full framework libraries), and first add the namespace:

`Add-Type -AssemblyName System.Windows.Forms`

Then query the `AllScreens` property to get the info we want:

`([System.Windows.Forms]::AllScreens | Select -ExpandProperty WorkingArea)`

And you'll hopefully see something like the following:

![Windows Sample](/WindowsTerminal_2019-09-06_13-34-34.png)

Look at that! The second display has some kind of offset there: -620. As you can tell, Windows layouts work a little differently to xrandr. Windows treats the top left corner of the primary monitor as 0,0, so the move to portrait display above that the offset must be a negative. That's okay, we can probably just put that in to xrandr as a positive offset of the left monitor instead...

`xrandr --output Virtual1 --auto --pos 0x620 --output Virtual2 --auto --pos 2560x0`

And perfect! Your X display layout should now exactly mirror your Windows layout, and your clicks will go _exactly_ where you want them to. Now all that's required is to put that xrandr command somewhere in your X startup scripts and enjoy your perfectly aligned VMware Workstation sessions.
