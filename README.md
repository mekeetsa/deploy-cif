# deploy-cif

This document contains the software build instructions for the Cold Stage. The software is in principle platform independent and can run on MacOS or Windows. The build instructions here are for Linux on x86 hardware only. Instructions for other platforms may be added at a later date. 

## 1. Prerequisites

- Computer with Linux installed. We recommend CentOS Stream 9 or Ubuntu 22.04. Any distribution with the ability to run ```distrobox``` should work.

- The computer needs to have at least 1 USB Gen 3 port

- Before starting installation, confirm that 
   - You have root access to the machine (```sudo -s```)
   - podman installed [https://podman.io/](https://podman.io/)
   - Distrobox installed [https://github.com/89luca89/distrobox](https://github.com/89luca89/distrobox)
   - git installed [https://github.com/git-guides/install-git](https://github.com/git-guides/install-git)

## 2. Build Container 

Clone deploy-cif

```bash
[user@host ~]$ git clone https://github.com/CIF-Cold-Stage/deploy-cif.git
```

Change to ```build``` directory
```bash
[user@host ~]$ cd deploy-cif/build
```

Build container image
```bash
[user@host ~]$ podman build . -t deploy-cif:latest
```

This should finish with the statement

```
COMMIT deploy-cif:latest
--> 3e4e03c0a8c
Successfully tagged localhost/deploy-cif:latest
```

Note: It is of course possible to run the software without the container environment. In this case you will need to use Ubuntu 22.04 as distribution. If you stay with the container skip this. However, if you go that route:

1. Use the steps in the ```Containerfile``` to install the required dependencies.
2. You will need install the ```Julia``` language [https://julialang.org/downloads/](https://julialang.org/downloads/).
3. You will need to install the camera device driver from IDS [https://en.ids-imaging.com/ids-peak.html](https://en.ids-imaging.com/ids-peak.html).
4. You probably want to install VS Code [https://code.visualstudio.com/](https://code.visualstudio.com/) 
5. You will need to clone the two software repos:
   1. [https://github.com/CIF-Cold-Stage/csDAQ](https://github.com/CIF-Cold-Stage/csDAQ)
   2. [https://github.com/CIF-Cold-Stage/DropFreezingDetection.jl.git](https://github.com/CIF-Cold-Stage/DropFreezingDetection.jl.git)
6. Adapt the steps below as needed.

## 3. Initialize Distrobox

1. Login as the user that will use the instrument. \
   If you continue the earlier session, ensure that you are in the home directory:

```bash
cd
```

3. Create the box

```bash
[user@host ~]$ distrobox create --image localhost/deploy-cif:latest --name cif
Creating 'cif' using image localhost/deploy-cif:latest	[ OK ]
Distrobox 'cif' successfully created.
To enter, run:

distrobox enter cif
```

3. Enter distrobox 
```
[user@host ~]$ distrobox enter cif
Container cif is not running.
Starting container cif
run this command to follow along:

 podman logs -f cif

 Starting container...                  	[ OK ]
 Installing basic packages...           	[ OK ]
 Setting up read-only mounts...         	[ OK ]
 Setting up read-write mounts...        	[ OK ]
 Setting up host's sockets integration...	[ OK ]
 Integrating host's themes, icons, fonts...	[ OK ]
 Setting up package manager exceptions...	[ OK ]
 Setting up dpkg exceptions...          	[ OK ]
 Setting up apt hooks...                	[ OK ]
 Setting up sudo...                     	[ OK ]
 Setting up groups...                   	[ OK ]
 Setting up users...                    	[ OK ]
 Integrating host's themes, icons, fonts...	[ OK ]
 Setting up package manager exceptions...	[ OK ]
 Setting up dpkg exceptions...          	[ OK ]
 Setting up apt hooks...                	[ OK ]
 Setting up sudo...                     	[ OK ]
 Setting up groups...                   	[ OK ]
 Setting up users...                    	[ OK ]
 Executing init hooks...                	[ OK ]

Container Setup Complete!
petters@cif:~$ 
```

4. Run postinstall script. This will download the julia package dependencies and precompile them. This may take a while...
```bash
[user@cif ~]$ bash /opt/csDAQ/build/build.sh 
```

5. After finishing, the postinstall script creates two files 
```50-usb-serial.rules``` and ```99-ids-usb-access.rules```. Exit the distrobox by pressing CTRL-D so you are back on the host system. Note the change in the hostname.

```bash
[user@cif:~]$     
logout
[user@host ~]$
``` 

6. On thee host copy these files to the ```/etc/udev/rules.d/``` directory and reload the rules. This will ensure that the user has access to the serial and camera devices. 

```bash 
[user@host ~]$ sudo -s
[root@host ~]$ mv *.rules /etc/udev/rules.d/
[root@host ~]$ udevadm control --reload-rules && udevadm trigger
```

7. Reboot the machine.


## 4. Test Camera

1. Plug in the camera. **Make sure it is plugged into a USB-3 port.** The camera should show a green light.

2. Enter distrobox 
```bash
[user@host ~]$ distrobox enter cif
```

3. Start ids_cockpit software. Check that you are able to open the camera and display an image stream.
```bash
[user@cif:~]$ ids_peak_cockpit
```

## 5. Test Serial Port

1. Plug in the controller via USB cable. The controller has a USB-to-serial converter inside.

2. Run the following commands to see if the serial port shows as ```/dev/ttyUSB0```. 

```
petters@cif:~$ cd /opt/csDAQ/src/
petters@cif:.../csDAQ/src$ julia --project
               _
   _       _ _(_)_     |  Documentation: https://docs.julialang.org
  (_)     | (_) (_)    |
   _ _   _| |_  __ _   |  Type "?" for help, "]?" for Pkg help.
  | | | | | | |/ _` |  |
  | | |_| | | | (_| |  |  Version 1.8.5 (2023-01-08)
 _/ |\__'_|_|_|\__'_|  |  Official https://julialang.org/ release
|__/                   |

julia> using LibSerialPort

julia> list_ports()
/dev/ttyUSB0
	Description:	FT232R USB UART - AD0K0TCI
	Transport type:	SP_TRANSPORT_USB
    
julia> 
```

## 6. Run the DAQ software

```
cd /opt/csDAQ/src/ && julia --project main.jl && cd .
```

## Fixing python libraries

Below is the fix if you get a message "A module that was compiled using NumPy 1.x cannot be run in NumPy 2.2.6"
or similar.

Background: `ids_peak_ipl` contains a compiled native extension (`_peak_ipl_python_interface`) 
that was built against **NumPy 1.x**, while your system currently has **NumPy 2.2.6**. 
The IDS IPL Python package is being imported by `IDSpeak`, which is then loaded through Julia's `PyCall`.

The quickest and safest fix for this deployment is therefore to **downgrade NumPy to a 1.x version**.
To use NumPy 1.26.4, do:

```
sudo python3 -m pip install --force-reinstall "numpy==1.26.4"
```

Then verify:

```
python3 -c "import numpy; print(numpy.__version__)"
python3 -c "from ids_peak_ipl import ids_peak_ipl; print('IDS IPL OK')"
python3 -c "from ids_peak_ipl import ids_peak_ipl; from ids_peak import ids_peak; print('IDS Peak OK')"
```

You should get:

```
1.26.4
IDS IPL OK
IDS Peak OK
```
