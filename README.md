# How to use a Huion Inspiroy 2 on Linux

## Basic setup

If you just need the pen and the main buttons, update your `hid-uclogic`
kernel driver, [configure
X](https://digimend.github.io/support/howto/drivers/wacom/) and you're set.

At the time of writing, [my patch to add Inspiroy 2
support](https://github.com/DIGImend/digimend-kernel-drivers/pull/679) has not
been merged yet, so you'll need to grab my branch and setup DKMS to install it
with every kernel update.

### DKMS setup on Debian/Ubuntu

Required packages: TBD.

From the `digimend-kernel-drivers` source directory, run:

```bash
debuild -us -uc
sudo apt install ../digimend-dkms_11_all.deb

sudo tee /etc/apt/preferences.d/digimend-dkms <<EOF
Package: digimend-dkms
Pin: version 11
Pin-Priority: 1001
EOF
```

Ideally you'd change the package version to indicate it's customized (`dch
--nmu ...`). But somehow the package is broken in that case, as the `dkms.conf`
ends up in a directory different from the driver sources.

Anyway, it's necessary after that to pin the package we just built, so that
it's not overwritten the next time you `apt upgrade` (Debian ships a `11-2`
version that would overwrite any kind of `+`ing). Pinning integrates better
with the other Apt tools than plain `apt-mark hold`.

## Advanced setup: wheel and group buttons

The three group buttons ("group keys" in Huion's docs) are meant to change the
bindings of the other buttons and wheel to match your current app or workflow.

Do note that past this point, the whole thing is basically a dump of the hack
that works for me. A dedicated application would be a better approach, possibly
improving on [existing and well-integrated
software](https://invent.kde.org/plasma/wacomtablet) to add proper support for
the Inspiroy 2.

### Requirements

This requires a few extra tools:

- `xbindkeys`.
- `xte` (from the `xautomation` package).

I assume that you configured your X server with the `wacom` X input driver as
in the Digimend link above. It can certainly be made to work with a pure evdev
or libinput setup as well, by removing all the `xsetwacom` calls and using
`xinput`/`xbindkeys` similar to the wheel's setup.

### `xbindkeys`

Create a `~/.xbindkeysrc` file with the following content:

```
"~/bin/tablet-setup bgroup1"
  b:101

"~/bin/tablet-setup bgroup2"
  b:102

"~/bin/tablet-setup bgroup3"
  b:103

# Action for Wheel Up when group 1 is active
"xte 'str ]'"
  b:104

# Action for Wheel Down when group 1 is active
"xte 'str ['"
  b:105

# Action for Wheel Up when group 2 is active
"xte 'keydown Control_L' 'str +' 'keyup Control_L'"
  b:106

# Action for Wheel Down when group 2 is active
"xte 'keydown Control_L' 'str -' 'keyup Control_L'"
  b:107

# Action for Wheel Up when group 3 is active
"pactl set-sink-volume @DEFAULT_SINK@ +200"
  b:108

# Action for Wheel Down when group 3 is active
"pactl set-sink-volume @DEFAULT_SINK@ -200"
  b:109
```

Replace the `"xte...` and `"pactl...` lines with what you want to do when the
tablet's wheel is scrolled up and down when each of the "groups" is active.

The example above will change the brush size, zoom in browser windows, and
control the PulseAudio volume, but these are shell commands, so there is no
limit on what you can do.

The `b:...` numbers are arbitrary but must match what's in the `tablet-setup`
script.

### `tablet-setup`

Create this script somewhere suitable, such as `~/bin/tablet-setup`. It is as
much configuration as it is script.

Edit the devices' names to match yours (see `xinput`).

Edit the `xsetwacom --set "$dev_pad" Button ...` lines with your own
keybindings. Do note that xsetwacom has its [own key
syntax](https://manpages.debian.org/bookworm/xserver-xorg-input-wacom/xsetwacom.1.en.html#Button)
different from xte.

Optionally, edit `on_connect` to add other commands to execute when the tablet
is connected, like fixing the height/width ratio with `xsetwacom ... Area ...`.

```bash
#!/bin/bash

set -eu

declare -r scheme="${1:-}"

declare -r dev_gbuttons='HUION Huion Tablet_H1061P Group Buttons pad'
declare -r dev_pad='HUION Huion Tablet_H1061P Pad pad'
declare -r dev_wheel_name='HUION Huion Tablet_H1061P Dial'
declare -r dev_stylus_name='HUION Huion Tablet_H1061P stylus'

# Work around the Dial appearing both as a pointer and as a keyboard, and
# confusing xinput. Likewise for the double stylus we end up with.
function extract_xinput_id() { sed -e 's/.*id=\([0-9]\+\).*/\1/'; }
declare -r dev_wheel="$( xinput list |grep "$dev_wheel_name.*pointer" |extract_xinput_id )"
: "${dev_wheel:?Did not find a "$dev_wheel_name" input device}"

function on_connect() {
    declare -r dev_stylus="$( xinput list |grep "$dev_stylus_name" |head -n 1 |extract_xinput_id )"
    : "${dev_stylus:?Did not find a "$dev_stylus_name" input device}"

    # Add any setup that should be done when the tablet is connected.
    xsetwacom --set "$dev_stylus" MapToOutput HEAD-1

    # Remap group buttons to something unique that xbindkeys can catch
    xsetwacom --set "$dev_gbuttons" Button 1 'button +101'
    xsetwacom --set "$dev_gbuttons" Button 2 'button +102'
    xsetwacom --set "$dev_gbuttons" Button 3 'button +103'
}

function load_bgroup1() {
    # Map the main buttons directly using xsetwacom
    xsetwacom --set "$dev_pad" Button  8 'key a'
    xsetwacom --set "$dev_pad" Button  9 'key b'
    xsetwacom --set "$dev_pad" Button 10 'key c'
    xsetwacom --set "$dev_pad" Button 11 'key d'
    xsetwacom --set "$dev_pad" Button 12 'key e'
    xsetwacom --set "$dev_pad" Button 13 'key f'
    xsetwacom --set "$dev_pad" Button 14 'key g'
    xsetwacom --set "$dev_pad" Button 15 'key h'

    # Catch wheel events with xinput and send them to xbindkeys
    xinput set-button-map "$dev_wheel" 1 2 3 104 105 6 7
}

function load_bgroup2() {
    xsetwacom --set "$dev_pad" Button  8 'key x'
    xsetwacom --set "$dev_pad" Button  9 'key x'
    xsetwacom --set "$dev_pad" Button 10 'key x'
    xsetwacom --set "$dev_pad" Button 11 'key x'
    xsetwacom --set "$dev_pad" Button 12 'key x'
    xsetwacom --set "$dev_pad" Button 13 'key x'
    xsetwacom --set "$dev_pad" Button 14 'key x'
    xsetwacom --set "$dev_pad" Button 15 'key x'

    xinput set-button-map "$dev_wheel" 1 2 3 106 107 6 7
}

function load_bgroup3() {
    xsetwacom --set "$dev_pad" Button  8 'key z'
    xsetwacom --set "$dev_pad" Button  9 'key z'
    xsetwacom --set "$dev_pad" Button 10 'key z'
    xsetwacom --set "$dev_pad" Button 11 'key z'
    xsetwacom --set "$dev_pad" Button 12 'key z'
    xsetwacom --set "$dev_pad" Button 13 'key z'
    xsetwacom --set "$dev_pad" Button 14 'key z'
    xsetwacom --set "$dev_pad" Button 15 'key z'

    xinput set-button-map "$dev_wheel" 1 2 3 108 109 6 7
}

case "$scheme" in
'')
    on_connect
    load_bgroup1
    ;;

bgroup1)
    load_bgroup1
    ;;

bgroup2)
    load_bgroup2
    ;;

bgroup3)
    load_bgroup3
    ;;
esac
```

### `udev` setup

And lastly, create a file `/etc/udev/rules.d/99-huion.rules` or similar, with:

```
ACTION!="remove", DRIVERS=="usb", \
   ATTRS{idVendor}=="256c", \
   ATTRS{product}=="Huion Tablet_H1061P", \
   ENV{DISPLAY}=":0", \
   ENV{XAUTHORITY}="/home/you/.Xauthority", \
   RUN+="/usr/bin/su you -c /home/you/bin/tablet-setup"
```

Replace the product name if necessary, and all instances of `you` by your
username.

By calling the `tablet-setup` script above, it will ensure that your button
bindings etc. get loaded upon device connect.


## How does it work?

### Without a Huion-aware driver

The tablet's buttons act like keyboard keys and are not configurable.

### With the hid-uclogic driver

The driver performs a special handshake when known Huion devices connect.

After this, the tablet sends proprietary "reports" whenever something happens.
There are several types, called "subreports" (one for pen activity, one for
buttons, one for wheel etc.)

The hid-uclogic driver exposes several input devices (`/dev/input/eventNN`),
one per subreport type, and translates the proprietary blob into standard
USB-HID reports.

Actually, as much as possible the hid-uclogic driver just provides the generic
hid driver with the technical description ("report descriptor") of the
proprietary report it'll route on each input device. It's only when the data
format is really non-standard (such as fragmented fields) that it massages it
into something better.

### X-level drivers for the input devices

The events from the input devices fed by the kernel driver still need
translation to make something useful happen in the graphical environment.

"Something useful" comes in two flavors: key presses (with a plausible keysym,
it's designed for keyboards), and buttons (usually for mice). Unlike key
presses, buttons do not have preset meanings beyond the first seven and leave a
lot of room for custom events.

Both key and button events are global, they cannot be filtered by the device
that emitted them.

#### wacom

The historical choice is "wacom" (xf86-input-wacom), written for the Wacom
tablets. It has unique features (such as the pressure curve), a convenient
`xsetwacom` utility, but it's not 100% compatible with non-Wacom hardware.

xsetwacom can generate key presses, button events, toggle relative/absolute
mode, scroll etc. It was written specifically for tablets, after all.

The hid-uclogic tries to expose compatible input devices, but it only works for
the pen and buttons. Wheels, notably, don't work (the wacom driver picks them
up, but does not produce any X-level events).

That's why the Digimend documentation recommends to restrict the wacom driver
with `MatchIsTablet yes` - it'll leave wheel etc. devices up for grab by
libinput.

#### libinput

Libinput has basic support for everything. It handles digitizer pens but with
limited control over the pressure, and can only substitute one button event for
another, through `xinput`.

### Generic X event rebinders

Tools like `xbindkeys` execute commands upon specific key or button events.
Together with the `xte` command which injects back events, it makes it possible
to trigger any shortcut in your favorite graphical app.

### Tablet-aware solutions

Specific tools like kcm-wacom provide a nice GUI to configure the tablet, its
keybindings etc.

They're usually based on libwacom, a repository of known tablet models and
their capabilities.

I've found kcm-wacom to be a bit finicky (dare I say, "buggy"?). As I did not
want a Gnome thing on my KDE desktop, and didn't have the time to fix whatever
was wrong with kcm-wacom, I ended up with the script hack above.

## hid-uclogic alternatives

OpenTablet is a cross-platform userspace driver for tablets and [will
provide](https://github.com/OpenTabletDriver/OpenTabletDriver/pull/2927) basic
support for the Inspiroy 2.

Or of course, one could use the official closed-source Huion drivers. Just
looking at their amateurish installer, I don't want them on my computer
though.
