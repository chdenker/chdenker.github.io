# Wacom Intuos S problems on Xubuntu 20.04 LTS

### Date: 2021-04-06

I frequently use the Wacom Intuos S graphics tablet in combination with Xournal++ for
note-taking. Yesterday, I installed Xubuntu 20.04 LTS since the end of life for Xubuntu 18.04 LTS is on April 29, 2021.
This is often an annoying task for me since some things stop working after a distro upgrade.

One of the hiccups was that there were issues with the graphics tablet.
The command

```bash
chris@chris-TUXEDO:/home/chris$ xsetwacom --list devices
Wacom Co.,Ltd. Intuos S stylus   	id: 10	type: STYLUS    
Wacom Co.,Ltd. Intuos S eraser   	id: 18	type: ERASER
```

didn't show the pad and the cursor device for example.
Trying to write with the tablet was also pretty difficult because the
area of the pad was not getting mapped correctly to the monitor
resolution.
Additionally, the sensitivity of the stylus was way off.

Executing

```bash
chris@chris-TUXEDO:/home/chris$ xsetwacom set 10 MapToOutput 1920x1080+0+0
```

didn't change anything.

After some googling, I stumbled upon [this](https://github.com/linuxwacom/libwacom/wiki/Troubleshooting#special-cases)
entry in the libwacom wiki. Apparently, the current Intuos S has Android support which leads the detection mechanism to sometimes
identify a Linux system as Android. I'm not sure why this didn't happen for me previously on my Xubuntu 18.04 installation.

A quick check shows that this is indeed the case on my new installation:

```bash
chris@chris-TUXEDO:/home/chris$ libwacom-list-local-devices 
# Device node: /dev/input/event6
[Device]
Name=Wacom Intuos S (Android Mode)
ModelName=CTL-4100
DeviceMatch=usb:2d1f:0374;
Class=Bamboo
Width=2
Height=4
IntegratedIn=
Layout=intuos-s-p3.svg
Styli=0x862;
...
```

The entry in the wiki explains how we can solve this. We simply press
buttons 1 and 4 (the leftmost and the rightmost one) and hold them for a
few seconds.
Now, the issue is fixed:

```bash
chris@chris-TUXEDO:/home/chris$ libwacom-list-local-devices 
# Device node: /dev/input/event6
[Device]
Name=Wacom Intuos S
ModelName=CTL-4100
DeviceMatch=usb:056a:0374;
Class=Bamboo
Width=6
Height=4
IntegratedIn=
Layout=intuos-s-p3.svg
Styli=0x862;
...
```

Furthermore, the remaining devices are detected now:

```bash
chris@chris-TUXEDO:/home/chris$ xsetwacom --list devices
Wacom Intuos S Pen stylus       	id: 10	type: STYLUS    
Wacom Intuos S Pen eraser       	id: 18	type: ERASER    
Wacom Intuos S Pen cursor       	id: 20	type: CURSOR    
Wacom Intuos S Pad pad          	id: 21	type: PAD
```

Lastly, I like to modify the pressure curve (I also did this on Xubuntu 18.04):

```bash
chris@chris-TUXEDO:/home/chris$ xsetwacom --set 10 PressureCurve 0 100 100 100
```

... and now we can take notes using Xournal++ just like before.
