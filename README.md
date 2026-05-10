# tutorial-how-to-set-up-your-own-darcs-hosting
A step by step how-to tutorial for setting up your own remote server for darcs, an open source version control software

Darcs can be thought of as a git alternative - it handles version control. It's open source software. Any version control software is only as useful as its ability to be hosted remotely: you'll want to have the ability to have off-site backups. 

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

## Install the Webmin control panel
Webmin is an open source GUI control panel for your server. It just makes admin-ing your server a breeze. I recommend using it although it's not an absolute must-have. Any of the steps we do later which utilize Webmin may be accomplished via the command line if you know how.

