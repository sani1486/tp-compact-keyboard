


sani1486: ``cd tp-compact-keyboard/tp-compact-usb-keyboard/``  
	  ``sudo ./tp-compact-usb-keyboard /dev/hidraw1``




FnLk switcher for ThinkPad Compact Keyboards with TrackPoint
===============================================================

The Lenovo Thinkpad Compact Keyboard with TrackPoint is a repackaging
of a Thinkpad laptop keyboard into a stand-alone case with a Bluetooth
wireless or cabled USB connection to the computer. Following the
current fashion, Lenovo has made the top row of keys serve two
purposes, using them for both the traditional Fn keys and for
"hotkeys" intended to control various functions such as volume and
screen brightness.

Pressing FnLk (Fn+Esc) switches these keys between generating Fn
keypresses and hotkey keypresses, but this mode change is handled not
by the keyboard itself but by Lenovo's keyboard driver for Windows.

On the Bluetooth version of this keyboard the default mode is to
generate hotkey keypresses; this package will help you make it
generate Fn keypresses instead. (The USB keyboard has FnLk set by
default and so starts out generating Fn keypresses.)

Pairing the Keyboard
--------------------

The Bluetooth keyboard uses [BT 2.1 Simple Secure Pairing](https://en.wikipedia.org/wiki/Bluetooth#Pairing_mechanisms) (SSP),
which should be supported by modern Linux distributions. Several different utilities can be used to pair the keyboard.

If your distribution does not support SSP, or if you have problems pairing with SSP, you can try disabling it with ``hciconfig hci0 sspmode 0``. [See this note on a Logitech keyboard](https://wiki.archlinux.org/index.php/Bluetooth#Logitech_keyboard_does_not_pair).

### bluetoothctl

Pairing can be done with the ``pair`` command. You should be prompted for the PIN code to type
as part of the process.

See: https://wiki.archlinux.org/index.php/Bluetooth_keyboard#Pairing_process

### bluetooth-wizard

Select the device (keyboard) and click "Continue". Watch for the PIN code in
the hcidump window and enter that.

See: https://bugzilla.redhat.com/show_bug.cgi?id=1019287#c3

### bluez-test-*

Versions of Bluez that ship with these scripts don't support SSP and thus don't report
the PIN code the keyboard requires you to use.

To get around this, run the following in a separate console before you start
pairing:

    sudo hcidump -at | grep -i passkey -A1

During the pairing process, you are looking out for lines like:

    > HCI Event: User Passkey Notification (0x3b) plen 10
    bdaddr 90:7F:61:01:02:03 passkey 123456

In this case, ``123456`` is the PIN you need to enter.

Put the keyboard into discoverable mode by holding down the power button until
the light starts flashing. Then use ``hcitool scan`` to find out what the
address of your keyboard is. It should resemble ``90:7F:61:01:02:03``.

Firstly, create the device:

    bluez-test-device create 90:7F:61:01:02:03

Then start trying to pair:

    bluez-simple-agent hci0 90:7F:61:01:02:03

You should see the passkey appear in the hcidump window. Type that passkey at
the prompt (using a keyboard already connected to your computer) and press
enter. You should now be paired. Finally:

    bluez-test-device trusted 90:7F:61:01:02:03 1
    bluez-test-input connect 90:7F:61:01:02:03

Toggling FnLk
-------------

### Linux 3.17 and Above

Linux kernels 3.17 and above (or any kernel with the [kernel-patches](https://github.com/lentinj/tp-compact-keyboard/tree/master/kernel-patch)
applied) have built in support for the keyboards. This means all keys should
work out the box, and you can control whether fn_lock is enabled by:

    echo 0 > /sys/bus/hid/devices/*17EF\:604*/fn_lock 

The kernel can't currently switch fn_lock automatically for you, however you
can use [esekeyd](https://sites.google.com/site/blabdupp/esekeyd) to map
KEY_FN_ESC to a script such as:

    { grep -q 1 /sys/bus/hid/devices/*17EF\:604*/fn_lock && echo 0 || echo 1; } \
        > /sys/bus/hid/devices/*17EF\:604*/fn_lock

...to toggle it.

### Apple OS X (MacOS)

[tpkb](https://github.com/unknownzerx/tpkb) is a Mac userland tool
similar to ``tp-compact-keyboard``. It uses the cross-platform hidapi
library.

### Linux Pre-3.17

To use the keyboard in an older kernel, you have several options. From
easiest to most difficult they are:

1. Use the ``tp-compact-keyboard`` program supplied with this package.
2. Load a kernel module such as [tp-compact-keyboard-backport](
   https://github.com/mithro/tp-compact-keyboard-backport).
3. Apply the patches in [kernel-patch](
   https://github.com/lentinj/tp-compact-keyboard/tree/master/kernel-patch)
   to the source code for your kernel and recompile it.

We cover only the first option here.

``tp-compact-keyboard`` is a small utility
to control some features of the keyboard, most
importantly to enable FnLk. It works only with the Bluetooth keyboard,
whilst the USB keyboard accepts the same commands but not in the same way,
one has to tweak the hidraw ioctls which (AFAIK) can't be done in a Bash script.

#### Requirements

This was developed under a Debian unstable kernel, 3.12-1-amd64, and has been
reported to work on kernels as early as 3.10 (CentOS 7). However older kernels (3.6, for example)
don't send the report; I'm not currently sure why.

#### Using

To enable or disable FnLk on all connected ThinkPad Bluetooth
keyboards, run ``./tp-compact-keyboard --fn-lock-enable`` or
``./tp-compact-keyboard --fn-lock-disable``. The program has a few
other functions as well, but they're not very useful on their own.
Have a look at the source.

I'm assuming what you really want to do is force FnLk on and forget about it
though. To do this, do the following as root:

    mkdir /etc/udev/scripts
    cp tp-compact-keyboard /etc/udev/scripts/tp-compact-keyboard
    cp tp-compact-keyboard.rules /etc/udev/rules.d/tp-compact-keyboard.rules

Then udev will enable FnLk every time the keyboard is connected.

#### Other options

There are a few other options, however they are mostly useless without a custom kernel
module handling the keyboard:

    --sensitivity xx
        Set sensitivity of TrackPoint. xx is hex value 01--09 (although values
        up to FF work).
    --native-fn-enable
    	By default, F7/F9/F11 generate a string of keypresses that might be
	useful under Windows. This stops this, and instead only generates
    	custom events.
    --native-mouse-enable
        The middle button generates 2 events by default, but Linux only
        understands one of them. This disables the event that Linux does
        understand, leaving you with no middle button.
    --native-mouse-disable
        Restore middle button to a working state.

Pair/unpair isn't enough to reset the keyboard, you need to power down the
keyboard to get it back to it's original state.

How I Did It
------------

I worked out the custom commands by sniffing what the Windows driver did.
First I set up a Windows XP virtual machine in KVM/QEMU and installed the drivers. Then I
disconnected the keyboard from my computer and ensured that the user running
QEMU could write to the ``/dev/bus/usb`` node associated with my Bluetooth
dongle.

Wireshark was readied for capture:

    # modprobe usbmon
    # wireshark

Then, I let Windows use the Bluetooth dongle:

    (qemu) usb_add host:0a5c:217f

If you capture the whole association process then you should be able to view
HID packets to/from the keyboard (filter for ``bthid``). The particuarly
interesting ones here are the ones going to the keyboard.

Disassembling the Keyboard
--------------------------

The upper part of the keyboard case is just clipped on; run something
around the seam on the underside of the keyboard to release it. The
keyboard itself is fastened to the lower part of the case with
double-sided tape. Very gentle persuasion (probably some heat too)
removes it.

The keyboard itself looks like it comes out of a Thinkpad, with metal plates stuck
to it to add heft. The keyboard connects to a controller board in the top right.

[Images of keyboard internals](images/)

Repairing the Trackpoint
------------------------

Some users of the Bluetooth keyboard have found that the TrackPoint stops
working whilst the rest of the keyboard is fine. This isn't a software issue;
apparently the chip that controls the TrackPoint isn't correctly soldered down.

See: http://bdm.cc/2016/06/lenovo-bluetooth-keyboard-repairs/

Replacement trackpoint caps
---------------------------

The keyboards use the "ThinkPad Low Profile TrackPoint Caps" (FRU 0A33908 for a 10-pack) by default. The original-size will fit also, but stick out more. This is fine for the cat-tongue, but with the others your finger can catch the edge.

Other Keyboards: Thinkpad Multi-Connect Keyboard
------------------------------------------------

There is also another physically-identical keyboard, the Thinkpad
*Multi-Connect* Keyboard, which allows you to toggle between 3 paired devices.

It appears to be China-only, and only available through sites like AliExpress.
There are no drivers for any OS (iOS/Android or Windows), just a single-sheet
manual is available here:
http://webdoc.lenovo.com.cn/lenovowsi/new_cskb/att/147372/%E5%BF%AB%E9%80%9F%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97.pdf

Unlike the Compact keyboards, the keyboard does not have (known) built-in
Fn-Lock functions, Fn, Fn-F1 and F1 are all separate scancodes, so they can be
remapped using a ``hwdb`` database.

Other Keyboards: Thinkpad Tablet 2 Keyboard
-------------------------------------------

Older than all the above, it was sold with the Tablet 2 and has a fold-out stand
for said tablet.

It doesn't have a trackpoint, rather a Blackberry-esque tiny trackpad which you
use like a trackball.

Dissassembly:

* Unclip the frame around the keyboard, the power slider just pokes the real button underneath.
* Undo the 4 screws holding the keyboard down (including the tri-wing screw with ground connection)
* Gently pry the keyboard from the metal backing plate it's glued to, remove 2 keyboard ribbon cables. Do not pry the backing plate away from the bottom plastic.
* Undo all the metal backing plate screws.
