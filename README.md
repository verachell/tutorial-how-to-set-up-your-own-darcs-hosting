# tutorial-how-to-set-up-your-own-darcs-hosting
A step by step how-to tutorial for setting up your own remote server for darcs, an open source version control software

Darcs can be thought of as a git alternative - it handles version control. It's open source software. Any version control software is only as useful as its ability to be hosted remotely: you'll want to have the ability to push your files off-site as backups or as a central storage repository. 

The problem : It's almost impossible to find pre-made hosting services for darcs aside from the flagship darcs hosting instance at `hub.darcs.net`. 

The solution: Set up your own private host for darcs. It's affordable (with a tiny VPS) and not too difficult at all. Here is the step by step tutorial of how to do it

## What you'll need
A tiny VPS server - you won't need a ton of RAM or space, just the lowest specced VPS you can find, ideally well under $5 per month.

If you prefer to test out the tutorial first before committing to an ongoing hosting subscription somewhere, consider doing your test at an hourly computing provider with low top-up minimums, e.g. RamNode, DigitalOcean, or Clouding.io. You can try it out for a day or 2 which should cost much less than $1, then destroy (not suspend or pause, but destroy) the instance to prevent you from getting any further charges. 

This tutorial was developed on RamNode with a Linux Debian VPS, but the principles would work on other hourly computing provider or any other VPS host.

## Step 0 - getting started
Sign up for your tiny VPS with at least a root password, and preferably an SSH key as well. It's possible your provider will set your password for you, but you do need to know the password. You'll also need your servier IP address. All of this info will be on the onboarding info given by your VPS host, or you can find it in the dashboard of your hourly computing provider. I recommend Debian but the same principles would work for Ubuntu. 

## Step 1 - SSH into your server
Assuming you have your root password and your server's IP address, open a terminal on your computer and type

`ssh root@your.server.ip.address`

it will ask you about fingerprint, say yes, then it will prompt for the root password. Once logged in, you are ready to proceed with setting up your server.

## Step 2 - Tidy up a few essentials
If your server gives a notice that your locale is not set, do the following:

`nano /etc/locale.gen`

and uncomment out the en_US.UTF-8 line. After exiting nano, on the command line do:

`locale-gen`

Then update software
```
apt-get update
apt-get upgrade
```

Then install firewall and set which ports to allow
```
apt install ufw
ufw allow 10000
ufw allow 22
ufw allow 53/tcp
ufw allow 53/udp
ufw enable
```
## [optional] Install any network based intrusion detection system (NIDS)
Although outside the scope of this tutorial, this is when you would install and configure whatever network based intrusion detection system you want, e.g. CrowdSec, fail2ban, psad etc. It's highly recommended you use one unless this is a brief tryout just for a day or two. 

## Step 3: Install the Webmin control panel
Webmin is an open source GUI control panel for your server. It just makes admin-ing your server a breeze. I recommend using it although it's not an absolute must-have. Instead of Webmin, any of the steps mentioned later in this tutorial may be accomplished via the command line if you know how. However, there is a lot that is just easier with Webmin. 

Here are the steps for installation at the time of writing. You may want to double check this against the official Webmin documentation in case anything has changed since then at https://webmin.com/download/ 

```
curl -o webmin-setup-repo.sh https://raw.githubusercontent.com/webmin/webmin/master/webmin-setup-repo.sh

sudo sh webmin-setup-repo.sh
```
and then (do not want to install usermin, just webmin)
```
apt-get install webmin --install-recommends
```

Then check it's working - go to https://your.server.ip.address:10000
Click to get past your browser's security warning, due to the fact of Webmin using a self-signed SSL certificate. The self-signed certificate is necessary, if you want a regular certificate you'd need to install one separately later on, which would require adding a domain name to your host. That's outside the scope of this tutorial, although if you'd like to do it you can follow the instructions here https://github.com/verachell/Tutorial-Easy-customizable-Apache-server-setup-without-email/blob/main/Connect-first-domain-and-set-up-ssl.md 

Log into webmin as root and using the root password you specified when you set up your VPS server. Go to notifications and install any software updates. If the server requires reboot from within webmin, do it - it will take several minutes but it will work.

On the Webmin menu, go to hardware and System time, change timezone to your local timezone.

## Step 4: Create the non-privileged sudo user
The new user you are creating will NOT be the darcs user (that will be done a little later). For security reasons, you want to do as little as possible as root - in fact, you don't want root to be able to SSH into the server at all. Having a non-privileged user that can sudo means that you can use this user to SSH into the server and then do sudo to issue root commands.

If you rebooted the server in Webmin in the previous step, you'll need to either SSH into the server again, or simply go to the Webmin menu -> Tools -> Terminal and issue these commands to create a new user
```
adduser newuser
```
replacing `newuser` with whatever username you want.Follow prompts to add user password. Then, add user to sudo:

```
usermod -aG sudo newuser
```
again, replacing newuser with the username you made.

### Add the new user to webim
Go to the Webmin menu, click on Webmin -> Webmin Users (do NOT go to System -> Users)

On the Webmin Users window, click on the tab "Create a new safe user"

For username, type in the name of your existing non-privileged user e.g. newuser (don't worry, this will not overwrite any of your user's existing files)

Password: choose "set to" and type in a password for your user to use with Webmin. If this is for a long-term website, this password should be different to the user's regular password, because it can otherwise get confusing when you change passwords. For a quick tryout it's OK to keep both passwords the same.

Then scroll down a bit to "Available Webmin Modules" and pick what your user should have access to. I recommend clicking Select All toget everything. At the very least, you'll definitely want File Manager, Terminal, upload and download. But I recommend everything. If you change your mind later you can always edit this later on.

Now click create.

Then log out of Webmin and check you can log in again to Webmin https://your.server.ip.address:10000 but as your new user with the new Webmin password.
