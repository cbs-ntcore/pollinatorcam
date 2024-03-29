Installation notes
-----

# Install OS

Install latest Pi OS (Desktop: tested March 2020)
Setup locale, timezone, keyboard, hostname, ssh

# Environment variables

Several environment variable are used for configuration/running. Please set
the following in your ~/.bashrc (or wherever else is appropriate). Note this
must be at the TOP of your bashrc (before the 'If not running interactively'... line).
You may have to use nano in the terminal to make these edits:

```bash
export PCAM_USER="camera login user name"
export PCAM_PASSWORD="camera login password"
export PCAM_NAS_USER="ftp server user"
export PCAM_NAS_PASSWORD="ftp server password"
```

# Clone this repository

Prepare for and clone this repository
```bash
. ~/.bashrc
mkdir -p ~/r/cbs-ntcore
cd ~/r/cbs-ntcore
git clone https://github.com/cbs-ntcore/pollinatorcam.git
```

# Install pre-requisites

```bash
sudo apt update
sudo apt install python3-numpy python3-opencv python3-requests python3-flask python3-systemd nginx-full vsftpd virtualenvwrapper apache2-utils python3-gst-1.0 gstreamer1.0-tools nmap
```

# Setup virtualenv

```bash
. ~/.bashrc
mkvirtualenv --system-site-packages pollinatorcam -p `which python3`
workon pollinatorcam
echo "source ~/.virtualenvs/pollinatorcam/bin/activate" >> ~/.bashrc
```

# Install tfliteserve

```bash
mkdir -p ~/r/braingram
cd ~/r/braingram
git clone https://github.com/braingram/tfliteserve.git
cd tfliteserve
pip3 install --no-deps https://dl.google.com/coral/python/tflite_runtime-2.1.0.post1-cp37-cp37m-linux_armv7l.whl
# install edge support
echo "deb https://packages.cloud.google.com/apt coral-edgetpu-stable main" | sudo tee /etc/apt/sources.list.d/coral-edgetpu.list
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt-get update
sudo apt-get install libedgetpu1-std
pip3 install -e .
# get model (TODO 404 permission denied, host this in repo or publicly)
wget https://github.com/braingram/tfliteserve/releases/download/v0.1/200123_2035_model.tar.xz
tar xvJf 200123_2035_model.tar.xz
```

# Install this repository

```bash
cd ~/r/cbs-ntcore/pollinatorcam
pip install -e .
pip install uwsgi
```

# Setup storage location

This assumes you're using an external storage drive that shows up as /dev/sda1. One option is to setup the drive as ntfs.
To format the drive as ntfs (to allow for >2TB volumes) in fdisk you will need to:
```bash
# confirm /dev/sda is your external drive before proceeding
# open fdisk
sudo fdisk /dev/sda
# switch to gpt: g
# delete all partions: d (for each partion)
# make a new partion that takes up all disk space: n (use all defaults)
# switch the partion type to microsoft basic data: t 11
# write fdisk: w
# make ntfs filesystem
sudo mkfs.ntfs -f /dev/sda1
```

Mount storage location

```bash
echo "/dev/sda1 /mnt/data auto defaults,user,uid=1000,gid=124,umask=002  0 0" | sudo tee -a /etc/fstab
sudo mkdir /mnt/data
sudo mount /mnt/data
```

# Setup FTP server

```bash
echo "
write_enable=YES
local_umask=011
local_root=/mnt/data" | sudo tee -a /etc/vsftpd.conf

sudo adduser $PCAM_NAS_USER --gecos "" --disabled-password
sudo adduser $PCAM_NAS_USER ftp
echo -e "$PCAM_NAS_PASSWORD\n$PCAM_NAS_PASSWORD" | sudo passwd $PCAM_NAS_USER
sudo mkdir -p /mnt/data/logs
sudo chgrp ftp /mnt/data
sudo chown pi /mnt/data
sudo chmod 775 /mnt/data
```

# Setup web server (for UI)

```bash
sudo htpasswd -bc /etc/apache2/.htpasswd pcam $PCAM_PASSWORD
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s ~/r/cbs-ntcore/pollinatorcam/services/pcam-ui.nginx /etc/nginx/sites-enabled/
```

# Setup systemd services

```bash
cd ~/r/cbs-ntcore/pollinatorcam/services
for S in \
    tfliteserve.service \
    pcam-discover.service \
    pcam-overview.service \
    pcam-overview.timer \
    pcam@.service \
    pcam-ui.service; do \
  sudo ln -s ~/r/cbs-ntcore/pollinatorcam/services/$S /etc/systemd/system/$S
done
# enable services to run on boot
for S in \
    tfliteserve.service \
    pcam-discover.service \
    pcam-overview.timer \
    pcam-ui.service; do \
  sudo systemctl enable $S
done
# start services
for S in \
    tfliteserve.service \
    pcam-discover.service \
    pcam-ui.service; do \
  sudo systemctl start $S
done
sudo systemctl restart nginx
```

# Network configuration

The lorex box will try to act as a gateway so if you want to use a different
interface (than eth0) for internet (like wlan0 or eth1) you will need to tell
the pi to not use eth0 as a gateway by adding the following to /etc/dhcpcd.conf

```
interface eth0
nogateway
```

# Configure cameras

In the background, pcam-discover will run network scans to find new cameras.
You can run the following to see what devices were found.

```bash
python3 -m pollinatorcam discover -p
```

When new cameras are connected, they will need to be configured. If this is
the first time the camera is configured, you may need to provide a different
username and password (like the default admin/admin).

```bash
# if camera ip is 10.1.1.153
python3 -m pollinatorcam configure -i 10.1.1.153 -u admin -p admin
```

# (optional) Setup pymicroclimate weather logging

Install and setup the [pymicroclimate](https://github.com/braingram/pymicroclimate) code.

```bash
# clone the repository
cd ~/r/braingram
git clone https://github.com/braingram/pymicroclimate.git

# install pymicroclimate into the pollinatorcam virtualenv
# if not already active, activate the virtualenv: workon pollinatorcam
cd ~/r/braingram/pymicroclimate
pip install -e .

# setup the pymicroclimate service
cd ~/r/cbs-ntcore/pollinatorcam/services
sudo ln -s ~/r/cbs-ntcore/pollinatorcam/services/pymicroclimate.service /etc/systemd/system/pymicroclimate.service
sudo systemctl enable pymicroclimate
sudo systemctl start pymicroclimate
```
