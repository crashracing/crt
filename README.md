# CRT

Crash's main repository.

All you need to get Crash running is described in the next few sections.

## Git

### Clone this repository

Note: If you want to use HTTPS instead of SSH, replace `git@github.com:` with `https://github.com/` in the file `.gitmodules` and in the following line:

```
$ git clone git@github.com:crashracing/crt
$ cd crt
```

### Get submodules

This populates the `./crt-*` directories.

```
$ git submodule init
$ git submodule update --remote
```

### Limitations

The directory `./debug-server` is not shipped in this release as it is not open-sourced yet.
It contains the TURAG Debug Server which is not needed to run the ROS components.

## Installation of Developer VM

This sets up a VM for development, which includes everything you need like ROS and the Arduino toolchain.

Note: Change the `vb.memory` variable in `Vagrantfile` if you have less than 4 GB free RAM.

### Install dependencies

```
$ sudo apt install vagrant virtualbox
```

### Create and deploy VM.

```
$ vagrant up
```

### Develop stuff

Continue in the `development` section.

### Stop VM.

```
$ vagrant halt
```

## Installation of Raspberry Pi 3

### Download Ubuntu and copy to SD card

```
$ wget https://ubuntu-mate.org/raspberry-pi/ubuntu-mate-16.04.2-desktop-armhf-raspberry-pi.img.xz
$ echo "dc3afcad68a5de3ba683dc30d2093a3b5b3cd6b2c16c0b5de8d50fede78f75c2  ubuntu-mate-16.04.2-desktop-armhf-raspberry-pi.img.xz" > sha256sum.txt
$ sha256sum -c sha256sum.txt
```

Make sure the last line returns "OK".

Then copy to SD card (where `sdX` is the device name of your SD card).

```
$ xz -dc ubuntu-mate-16.04.2-desktop-armhf-raspberry-pi.img.xz | sudo dd of=/dev/sdX
```

### First boot

* configure timezone, keymap, user to your liking
* in a terminal, do:

```
$ sudo systemctl enable ssh
$ sudo systemctl start ssh
```

Configure SSH for pubkey authentication. A suggestion:

```
# ~/.ssh/config
Host raspi_root
        HostName 192.168...
        User root
        PreferredAuthentications publickey,keyboard-interactive,password
        PubkeyAuthentication yes
```

Then copy your SSH key to the raspberry:

```
% ssh-copy-id raspi_root
```

If this fails due to missing SSH key, generate it:

```
% ssh-keygen -t ecdsa -b 521
```

Finally, check that login using SSH works without password:

```
% ssh raspi_root
```

### ROS installation with Ansible

Install Ansible on your host machine:

```
% apt-get update && apt-get install ansible
```

Then start the Ansible playbook on your host machine which connects to the raspberry via SSH and configures everything:

```
% ansible-playbook -i provisioning/hosts_pi provisioning/install-pi.yml
```

When the last step fails see the next section.

### Clone CRT repository

The Ansible playbook creates the directory `/vagrant` to keep file paths consistent with our Vagrant VM.

When using Vagrant for setup this directory automatically contains the CRT git repository. For installation on the Raspberry Pi you need to clone it manually:

```
% ssh raspi
% cd /vagrant
% git clone git@github.com:crashracing/crt
% cd crt
% git submodule init
% git submodule update --remote
```

### Install CRT dependencies

Just run the Ansible playbook (again) as in the ROS installation section.

### Setup UART communication with Arduino

The Arduino Nano's hardware UART (pins RXD/PD0, TXD/PD1) is connected to the Raspberry Pi 3 UART0 (GPIO14, 15).

* disable serial kernel log: remove `console=serial0,115200` from `/boot/cmdline.txt`
* disable Bluetooth: run `sudo systemctl disable hciuart`
* add the following to `/boot/config.txt`:

```
enable_uart=1
dtoverlay=pi3-disable-bt
```

After a reboot you should be able to open the serial port (`/dev/serial0` should link to `/dev/ttyAMA0`):

```
picocom /dev/serial0 -b 115200 -l
```

## Development

### Build ROS workspace (Vagrant)

Enter the VM and go to this directory. It is available as a shared folder under `/vagrant`, so the file you're reading right now would be located at `/vagrant/README.md`.

```
$ vagrant ssh
# -> You're now inside the VM.
$ cd /vagrant/crt-ros
```

Compile the CRT ROS packages:

```
$ catkin_make
```

See further instructions in `./crt-ros/README.md`.

### Workflow

Develop stuff. Commit and push in the subdirectories.

Example (if you added something in crt-ros):

```
$ cd crt-ros
$ git commit -am 'developed stuff'
$ git push
```

### Run

On Crash:

```
$ cd /vagrant/crt/crt-ros
$ roslaunch crt_launch all_robot.launch
```

On PC (maybe in different terminals):

```
$ roslaunch crt_launch all_pc.launch
$ roslaunch crt_launch viz.launch
```

## Credits

* `provisioning/roles/ansible-role-keyboard` source: https://github.com/gantsign/ansible-role-keyboard (MIT)
* `provisioning/roles/ansible-role-ros` source: https://github.com/jalessio/ansible-role-ros (MIT/BSD)
