# dotfiles

Node: Since I switched to Artix Linux, some of the configurations used are no longer needed because Artix does not have systemd. I leave the notes here so that it can be used by someone that uses Arch.

### Backlight Control
inside ```/etc/default/grub```, append ```GRUB_CMDLINE_LINUX_DEFAULT="acpi_osi='!Windows 2012'"```.

then regenerate the config file with
```grub-mkconfig -o /boot/grub/grub.cfg```

### Hibernation
inside
```/etc/default/grub```

append 
```GRUB_CMDLINE_LINUX="resume=/dev/sdXX"```
(sdXX = SWAP)

then regenerate the config file with
```grub-mkconfig -o /boot/grub/grub.cfg```

edit how systemd handles sleep and hibernation with
```sudo $EDITOR /etc/systemd/logind.conf```

append

```
HandleLidSwitch=suspend
HandleLidSwitchDocked=ignore
HandleLidSwitchExternalPower=HandleLidSwitch
```
to apply any changes signal ```systemd-logind``` with ```HUP```
```systemctl kill -s HUP systemd-logind```

to hibernate on low battery level make sure to have ```acpi``` installed and append 
```
# Suspend the system when battery level drops to 8% or lower
SUBSYSTEM=="power_supply", ATTR{status}=="Discharging", ATTR{capacity}=="[0-8]", RUN+="/usr/bin/systemctl hibernate"
```

to
```/etc/udev/rules.d/99-lowbat.rules```

testing events: one way to test udev rules is to have them create a file when they are run. 

to 
```/etc/udev/rules.d/98-discharging.rules```

apppend
```SUBSYSTEM=="power_supply", ATTR{status}=="Discharging", RUN+="/usr/bin/touch /home/example/discharging"```

This creates a file at 
```/home/example/discharging``` 
when the laptop charger is unplugged

Hybrid-sleep on suspend or hibernation request

It is possible to configure systemd to always do a hybrid-sleep even on a suspend or hibernation request.

The default suspend and hibernation action can be configured in the ```/etc/systemd/sleep.conf``` file. To set both actions to hybrid-sleep, append

```
[Sleep]
# suspend=hybrid-sleep
SuspendMode=suspend
SuspendState=disk
# hibernate=hybrid-sleep
HibernateMode=suspend
HibernateState=disk
```

to ```/etc/systemd/sleep.conf```

### Brave Video Acceleration
install ```libva-intel-driver``` and launch brave with ``` --use-gl=desktop``` and ```--ignore-gpu-blacklist```

then add ```h264ify``` extension to brave

### Touchpad
in ```/etc/X11/xorg.conf.d/50-synaptics.conf```
add the following:

```
Section "InputClass"
        Identifier "touchpad"
        MatchProduct "SynPS/2 Synaptics TouchPad"
        # MatchTag "lenovo_x230_all"
        Driver "synaptics"
        # fix touchpad resolution
        Option "VertResolution" "100"
        Option "HorizResolution" "65"
        # disable synaptics driver pointer acceleration
        Option "MinSpeed" "1"
        Option "MaxSpeed" "1"
        # tweak the X-server pointer acceleration
        Option "AccelerationProfile" "2"
        Option "AdaptiveDeceleration" "16"
        Option "ConstantDeceleration" "16"
        Option "VelocityScale" "20"
        Option "AccelerationNumerator" "30"
        Option "AccelerationDenominator" "10"
        Option "AccelerationThreshold" "10"
	# Disable two fingers right mouse click
	Option "TapButton2" "0"
        Option "HorizHysteresis" "100"
        Option "VertHysteresis" "100"
        # fix touchpad scroll speed
        Option "VertScrollDelta" "500"
        Option "HorizScrollDelta" "500"
EndSection
```

### Power Saving

#### TLP
install tlp with ```sudo pacman -S tlp```

enable startup
```systemctl enable tlp.service```

run tlp
```systemctl start tlp.service```

in ```/etc/default/tlp```
add the following:

```
START_CHARGE_THRESH_BAT0=67
STOP_CHARGE_THRESH_BAT0=100
```

### Charge thresholds

```
# echo 40 > /sys/class/power_supply/BAT0/charge_start_threshold
# echo 80 > /sys/class/power_supply/BAT0/charge_stop_threshold
```
Note: after battery removal, default value will be loaded.

### Fan control
install AUR ```thinkfan``` and configure ```/etc/thinkfan.conf``` with:

```
tp_fan /proc/acpi/ibm/fan
hwmon /sys/class/thermal/thermal_zone0/temp

(0, 0,  60)
(1, 53, 65)
(2, 55, 66)
(3, 57, 68)
(4, 61, 70)
(5, 64, 71)
(7, 68, 32767)
```

### Kernel
in ```/etc/mkinitcpio.conf```, add ```MODULES="i915"```. Regenerate initram image with ```sudo mkinitcpio -p linux```.

### Dual Screen
` xrandr --output LVDS-1 --auto --output HDMI-1 --above LVDS-1`

### GRUB Background
inside
```/etc/default/grub```

append 
```GRUB_BACKGROUND="path_to_png"```

then regenerate the config file with
```grub-mkconfig -o /boot/grub/grub.cfg```

### Broadcom Bluetooth firmware for Linux kernel
```yay -S broadcom-bt-firmware-git```

### Firefox Hardware Acceleration

install ```libva-intel-driver``` and ```libva-utils```

run ```vainfo``` command and if vainfo gives a non-error output (i.e. a list of codecs it can decode and encode) you're set.

install ```h264ify``` firefox extension 

You can test that it's working by launching Firefox with ```MOZ_LOG="PlatformDecoderModule:5"``` - for example by running ```MOZ_LOG="PlatformDecoderModule:5"``` firefox in a terminal. While this does restrict you to x264 (and thus 1080p) on YouTube, it's great for low powered laptops etc running on battery (not so much on my Threadripper desktop system, but meh).

Firefox needs launching with ```MOZ_X11_EGL=1``` and you can do this either with the launcher, or set it as an environmental variable in ```/etc/environment``` or ```.bashrc/.zshrc/whatever (export MOZ_X11_EGL=1)```.

Firefox ```about.config```

```
gfx.webrender.all:true
media.ffmpeg.dmabuf-textures.enabled:true
media.ffmpeg.vaapi.enabled:true
media.ffvpx.enabled:false
```

Test it with ```MOZ_WEBRENDER=1 MOZ_X11_EGL=1 MOZ_LOG="PlatformDecoderModule:5" firefox 2>&1 | grep 'VA-API'``` and play a video. It either says something like

```41:[Child 18537: MediaController #2]: D/PlatformDecoderModule DMA-Buf/VA-API can't be used, WebRender/DMA-Buf is disable```

Then it does not work, if it dumps a lot of video encoding infos like

```06:[Child 20521: MediaPDecoder #3]: D/PlatformDecoderModule DMABUF/VA-API Got one frame output with pts=21533333dts=21533333 duration=33333 opaque=-9223372036854775808```

then it works.

### IntelliJ or other Java based apps blank screen
add ```export _JAVA_AWT_WM_NONREPARENTING=1``` to ```.xinitrc```.
