Raspberry Pi Zero W Python Development Environment
===========================================

This project describes how to setup a Raspberry Pi Zero W for remote Python development using VSCode.  
Once setup, you can develop and debug Python scripts on your local PC and execute on a remote Raspberry Pi Zero W.

The following was adapted from:
https://bitbucket.org/jIRI/rasp-pi-zerow-vscode/src/master/

# Hardware 

This assumes you have a Raspberry Pi Zero W (W = built-in wifi/bluetooth support).
There a tons of dev kits available, e.g.:
https://www.canakit.com/raspberry-pi-zero-wireless.html?src=raspberrypi

__NOTE:__ You need a micro SD card, the RPI board-only doesn't have onboard ROM to store the OS.

__NOTE:__ You might also need a micro SD reader if you computer doesn't have one. e.g.:  
https://www.amazon.com/s?k=usb+micro+sd+card+reader


# Raspbery Pi Setup

## 1) Program Raspberry Pi OS to SD Card 

Plug the SD card in your computer's SD card reader, then  
use the Raspberry PI Imager to program RPI OS to your micro SD card:  
https://www.raspberrypi.org/software/

It's recommended to start with the default OS (with desktop support) before getting crazy with RPI OS-lite.

## 2) Enable network over USB

After programming RPI OS from setup 1, unplug the SD card, then plugin it back into your computer.
Go to your file explorer, you should see the SD card be mounted as a new drive (at least on Windows)
with a description as: `boot`.

Open the SD card 'boot' drive and edit the following files in the root of the SD card:

__config.txt:__  
Open `config.txt` and to the end add:
```
dtoverlay=dwc2
```
More details about config.txt here:  
https://www.raspberrypi.org/documentation/configuration/config-txt/

__cmdline.txt:__  
Open `cmdline.txt`, just after `rootwait` add:
```
modules-load=dwc2,g_ether
```

__ssh__   
Create empty file named `ssh` in root of the SD card

## 3) Install mDNS onto your computer

If you're using Windows, install:
https://support.apple.com/kb/DL999?viewlocale=en_US&locale=en_US

If you're using Linux, install:
```
sudo apt-get install avahi-daemon
```

## 4) Start the Raspberry Pi Zero W

Unmount the SD card an plug it into the RPI.
Then plug the USB micro into the RPI's `USB` port (_not_ PWR) and the other side into your computer.

## 5) Configure sharing on your Windows network adapter so Pi can see the Internet
Go somewhere around __Control Panel -> Network and Internet -> Network Connections__  
Double click your default network connection, click Properties..., tab Sharing

## 6) Open an SSH session to the RPI

On Windows, PuTTY is recommended: http://www.putty.org/

Connect over SSH (port 22) with connection string: `pi@raspberrypi.local`   
Accept certificate  
Default password is `raspberry`

## 7) Configure Wi-Fi

From the SSH session, issue the command:

```
sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
```
From the opened file editor, add the following to the end:

```
network={
  ssid="MY_NETWORK_NAME"
  psk="MY_NETWORK_PASSWORD"
}
```
Updating `MY_NETWORK_NAME` and `MY_NETWORK_PASSWORD` with your Wi-Fi info.

Then, `Ctrl+O` to save, `Ctrl+X` to exit nano.

Next, issue the command:

```
sudo wpa_cli reconfigure
```

Then issue the command:

```
sudo reboot
```

This will reboot the RPI and close your SSH session. 
You'll need to start a new SSH session for the subsequent steps.


## 8) Create the workspace directory 

From the RPI SSH session (step 7), issue the command:

```
mkdir workspace
chmod 0777 workspace
cd workspace
pwd
```

Which should create the directory:

```
/home/pi/workspace
```

## 9) Update Pi, install samba, config samba

From the RPI SSH session (step 7), issue the commands:

```
sudo apt-get update
sudo apt-get install -y samba samba-common-bin
```

Then issue:

```
sudo nano /etc/samba/smb.conf
```
Add to the end of file:
```
[workspace]
path=/home/pi/workspace
browsable=yes
writable=yes
only guest=no
create mask=0777
directory mask=0777
public=yes
```

Then issue:
```
sudo service smbd restart
```

In your local file explorer, you should be able to open (on Windows):
```
\\RASPBERRYPI\workspace
```


## 10) Install debugpy on the Pi to be able debug remotely

We use [debugpy](https://github.com/microsoft/debugpy) to enable Python remote debugging.

From the RPI SSH session (step 7), issue the command:

```
sudo pip3 install debugpy==1.3.0
```
(You may be able to use a newer version, but that version worked for me)

__NOTE:__ This repo uses Python3, hence `pip3`.

## 12) Resize your Pi partition to use all available space on SD card

From the RPI SSH session (step 7), issue the commands:
```
sudo raspi-config --expand-rootfs
sudo reboot
```

# Prepare VSCode Workspace

## 1) Install the dirsync Python package


We use the [dirsync](https://pypi.org/project/dirsync/) Python package
to automatically synchronize files between the local workspace and the remote RPI workspace.

Issue the following command from a local terminal to install the package:

```
pip3 install dirsync
```

After installing, the `dirsync` command should be accessbile to VSCode.

## 2) Open VSCode Workspace

Assuming you cloned this repo, open the VSCode 'workspace' that is at the root of this repo: `workspace.code-workspace` 
then install the 'Recommended Extensions'.

## 3) Map Network Drive

If using Windows, map the `\\RASPBERRYPI\workspace` network drive, more details [here](https://support.microsoft.com/en-us/windows/map-a-network-drive-in-windows-10-29ce55d1-34e3-a7e2-4801-131475f9557d)  
After this is complete, you should have a new drive, e.g. `Z:\\` that points to your RPI's `/home/pi/workspace` directory.

This is required so we can easily sync the local workspace with the RPI's workspace.

## 4) Update tasks.json

Open `<repo root>/.vscode/tasks.json`,

In the `sync-workspace` task, update the arguments as necessary for your workspace.  
At a minimum, you may need to change `z:/` to your networking mapping drive from the previous step.

# Run the debugger

That's it! Running the `Debug Python` debug configuration should:
1. Synchronize the local workspace with the RPI's workspace (assuming the network drive is properly mapped)
2. Start the `main.py` python script with remote debugging enabled
3. Cause VSCode to connect to the debug server and allow for single-stepping in the Python script as if it were running locally
