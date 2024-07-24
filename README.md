# Raspberry Pi Zero W Python Development Environment

This project describes how to set up a Raspberry Pi Zero W for remote Python development using vscode.  
Once setup, you can develop and single-step debug Python scripts from your local PC and execute on a remote Raspberry Pi Zero W.

**The following was adapted from [driedler/rpi0w_dev_env](https://github.com/driedler/rpi0w_dev_env) which was adapted from
[jIRI/rasp-pi-zerow-vscode](https://bitbucket.org/jIRI/rasp-pi-zerow-vscode/src/master/)**. Thank you!

CHANGES FROM [driedler/rpi0w_dev_env](https://github.com/driedler/rpi0w_dev_env): This version does not require `dirsync` from python or `putty` and simply uses `robocopy` and `ssh.exe` to replace these third-party utilities respectively, and simplifies configuration by using a `settings.json` file instead of manual replacement.

This setup also does not use mDNS and you must specify the pi's static IP address, since I did not want to go through the trouble of setting this up with my unique network configuration.

## Hardware

This assumes you have a Raspberry Pi Zero W (W = built-in Wi-Fi/Bluetooth support).

## Raspberry Pi Setup

### 1) Create a key and add it to the pi during the setup

There is a section to add a public key for SSH. Also enable ssh on the pi.

You may also add a key to `~/.ssh/authorized_keys` after installation (I had to do this to add a key without a passphrase to make developing less annoying since every ssh command will require passphrase authentication if you have one set.)

### 2) (Optional) Enable network over USB

Follow a guide like [adafruit's tutorial](https://learn.adafruit.com/turning-your-raspberry-pi-zero-into-a-usb-gadget/ethernet-gadget) on how to enable the Ethernet gadget. There are some parts of which are out of date since the tutorial was created for Windows 7 (such as installing the RNDIS driver), if you are stuck you may want to look at my experience setting this up on my website.

### 3) (Optional) Configure sharing on your Windows network adapter so Pi can see the Internet

If you are using USB ethernet as described in the first step, follow this step to allow the Pi to access the network.

Go somewhere around **Control Panel -> Network and Internet -> Network Connections**  
Double click your default network connection, click Properties..., tab Sharing

NOTE: **This will likely make any VMs or WSL instances unable to see the internet while sharing is active**

### 4) Create the workspace directory

From the RPI SSH session (step 7), issue the command:

```bash
mkdir workspace
chmod 0777 workspace
cd workspace
pwd
```

Which should create the directory:

```txt
/home/pi/workspace
```

If you are using a different directory, change `"ws_rpi_remote_path"` in `settings.json`.

### 5) Update Pi, install samba, config samba

From the RPI SSH session (step 7), issue the commands:

```bash
sudo apt-get update
sudo apt-get install -y samba samba-common-bin
```

Then issue:

```bash
sudo nano /etc/samba/smb.conf
```

Add to the end of file:

```bash
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

```bash
sudo service smbd restart
```

In your local file explorer, you should be able to open (on Windows):

```batch
\\<STATIC IP>\workspace
```

### 6) Install debugpy on the Pi to be able debug remotely

We use [debugpy](https://github.com/microsoft/debugpy) to enable Python remote debugging.

From the RPI SSH session (step 7), issue the command:

```bash
sudo pip3 install debugpy
```

I am using `debugpy` version 1.8.2 as of time of writing.

### 7) Open a port for `debugpy` access

By default, this workspace uses port 5678. Change as required.

Install `ufw` and allow this port.

```bash
sudo apt install ufw
sudo ufw allow 5678
```

### 8) Resize your Pi partition to use all available space on SD card

From the RPI SSH session (step 7), issue the commands:

```bash
sudo raspi-config --expand-rootfs
sudo reboot
```

## Prepare VSCode Workspace

### 9) Open VSCode Workspace

Clone this repository to a directory on your host machine.

Open the VSCode 'workspace' that is at the root of this repo: `workspace.code-workspace`
then install the 'Recommended Extensions'.

### 10) Map Network Drive

If using Windows, map the `\\<PI STATIC IP ADDRESS>\workspace` network drive, more details [here](https://support.microsoft.com/en-us/windows/map-a-network-drive-in-windows-10-29ce55d1-34e3-a7e2-4801-131475f9557d)  
After this is complete, you should have a new drive, e.g. `Z:\\` that points to your RPI's `/home/pi/workspace` directory.

This is required so we can easily sync the local workspace with the RPI's workspace.

### 11) Clone this repository and configure

Open `.vscode/settings.json`:

```json
{
  "ws_rpi_address": "<YOUR PI STATIC IP ADDRESS HERE>",
  "ws_rpi_share_drive_letter": "Z",
  "ws_rpi_entrypoint_file": "./main.py",
  "ws_rpi_user": "pi", // Default username is `pi`
  "ws_rpi_remote_path": "/home/pi/workspace",
  "ws_rpi_debug_port": 5678
}
```

Change these parameters as required. This is an explanation of each parameter and what step it applies to:

| Parameter                     | Step                                                                                 | Explanation                                                                                                                                                                                     |
| ----------------------------- | ------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `"ws_rpi_address"`            | Always                                                                               | **You must alwayas** manually set this to your pi's static ip, which can be found with `ip a`.                                                                                                  |
| `"ws_rpi_share_drive_letter"` | [10](#10-map-network-drive)                                                          | Windows explorer may assign a different drive letter to the samba share than `Z:\\`, in this case change `"ws_rpi_share_drive_letter"` to the **letter only** (Do not include colon or slashes) |
| `"ws_rpi_entrypoint_file"`    |                                                                                      | Change this if you have decided to change the entrypoint file, `main.py`                                                                                                                        |
| `"ws_rpi_user"`               | Installation                                                                         | Change this if you have set a username in the imager wizard. The default name is `pi`.                                                                                                          |
| `"ws_rpi_remote_path"`        | [4](#4-create-the-workspace-directory), [5](#5-update-pi-install-samba-config-samba) | Change this if you are using a different directory to store the code on the raspberry pi side.                                                                                                  |
| `"ws_rpi_debug_port"`         | [7](#7-open-a-port-for-debugpy-access)                                               | Change this if you are not using the port of `5678` for the `debugpy` server on the side of the raspberry pi                                                                                    |

## Run the debugger

That's it! Running the `Debug Python` debug configuration should:

1. Synchronize the local workspace with the RPI's workspace (assuming the network drive is properly mapped)
2. Start the `main.py` python script with remote debugging enabled
3. Cause VSCode to connect to the debug server and allow for single-stepping in the Python script as if it were running locally
