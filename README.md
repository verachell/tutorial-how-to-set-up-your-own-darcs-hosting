# tutorial-how-to-set-up-your-own-darcs-hosting
A step by step how-to tutorial for setting up your own remote server for darcs, an open source version control software

Darcs can be thought of as a git alternative - it handles version control. It's open source software. Any version control software is only as useful as its ability to be hosted remotely: you'll want to have the ability to push your files off-site as backups or as a central storage repository. 

The problem : It's almost impossible to find pre-made hosting services for darcs aside from the flagship darcs hosting instance at `hub.darcs.net`. 

The solution: Set up your own private host for darcs. It's affordable (with a tiny VPS) and not too difficult at all. In fact, all you need is a server you can SSH into, and install `darcs` on it, and it will "just work". Here is the step by step tutorial of how to do it

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

Then check it's working - go to `https://your.server.ip.address:10000` Click to get past your browser's security warning, due to the fact of Webmin using a self-signed SSL certificate. 

The self-signed certificate is necessary at this point, if you want a regular certificate you'd need to install one separately later on, which would require adding a domain name to your host. That's outside the scope of this tutorial, although if you'd like to do it you can follow the instructions here https://github.com/verachell/Tutorial-Easy-customizable-Apache-server-setup-without-email/blob/main/Connect-first-domain-and-set-up-ssl.md 

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

Then log out of Webmin and check you can log in again to Webmin `https://your.server.ip.address:10000` but as your new user with the new Webmin password.

## Step 5. Change the SSH port away from the default of 22

This tactic will stop a lot of brute force attacks
```
nano /etc/ssh/sshd_config
```
in the file, uncomment the port line and change the port from 22 to some other port not in use. So, avoid ports 53, 80, 443, 10000 and 22. For example, change to port 65000

in the same file, add a line saying
```
Protocol 2
```
then after saving the file, open that port
```
ufw allow 65000
```
Then you'll need to restart ssh:
```
systemctl restart ssh
```
## Step 6. Set up keys for new user (and root if you didn't sign up with one), disallow root ssh
Now, keeping your existing terminal open, go to a new terminal and open a fresh connection to the server under your non-privileged user, specifying the new port number like this:
```
ssh -p 65000 newuser@your.server.ip.address
```
Check that this works and allows you to log in as the new user with the password. Now on local Linux machine, set up SSH keys for new user:
```
ssh-keygen -t ed25519 -f ~/.ssh/newuser_ed25519 -C "non-priv user key for temp server"
```

As non-priv user in remote machine (should be there in SSH already in a terminal window),do the following:
```
mkdir .ssh
```
change perms of .ssh to 0700
```
chmod 0700 .ssh
cd .ssh
nano authorized_keys 
```
Then copy and paste only the public key from your local machine (its file extension ends with .pub) into the remote server's `authorized_keys` file that you just made.

Change permissions of the authorized_keys file on the remote server so that no-one besides you can read them:
```
chmod 0600 authorized_keys
```
IMPORTANT: If you didn't supply an SSH key when you signed up for your service, you need to set up the keys for root as well. Simply repeat the steps above that you did, but as the server root user. So the making of keys on your local machine will be the same (just pick a different filename for the keys). At the server end, I recommend SSH-ing into your server as root from the command line `ssh root@your.server.ip.address` so that you'll be in the root home directory when you make the .ssh home directory.

At this point, both root and newuser should have SSH public keys copied in and stored on the server. The next step is to check that newuser can SSH in.

Without closing any existing terminals, open yet another terminal on your local machine and connect to the server via ssh as follows. This method specifies exactly which key file you're using, which is important, it will be something like `~/.ssh/newuser_ed25519` or wahtever you called it. 

If you don't specify the key file (and if you have more than 1 key file on your local machine) you might wind up with an authentication error and a refusal to connect. This is because your local machine might default to using the wrong key when there is a choice of multiple. We'll also have to specify the new port every time too, otherwise it will default to port 22. Here is how to specify everything on your local machine:
```
ssh -i ~/.ssh/newuser_ed25519 -p 65000 newuser@your.server.ip.address
```
Now, if everything has been done correctly, it will ask for a passphrase (not a password!) and this is where you type in the one from when you generated the keys.

At this point it should let you log in. If it doesn't, go back through these steps. Make sure you're able to log in as the non-privileged user through your SSH keys before moving on to the next step.

Now disable password logins for SSH access and only allow authentication keys, disable root login. Only do this step after you're able to log in through your SSH keys as your non-privileged user.

Edit the ssh config file:
```
sudo nano /etc/ssh/sshd_config
```
uncomment the `PasswordAuthentication` line and set it to no

Look for the following and change as follows:
```
PermitRootLogin no 
```
Make sure the following line is there - you may need to add it yourself
```
AllowUsers newuser
```
Be sure to type in your new username correctly or you could be locked out! Then save and exit. Then restart ssh
```
sudo systemctl restart ssh
```
and now log in on a new terminal as new user with SSH keys as before. This should work. Try a new connection and try to log in as root, it should be denied.
## Installing darcs on the VPS and creating a darcs user
```
sudo apt install darcs
```
The instructions here come from https://darcs.net/SSH  First, create a new account for the user that will own the repos. In this case I will NOT make it my normal non-privileged user because I don't want a user that can sudo, all they need to do is use darcs.
```
sudo adduser newuser2
```
Do NOT add this user to sudo group! Log into webmin as root. Go to tools -> file manager. Navigate to `/home/newuser2` Then go to file-> create new directory, and create a directory called `bin` and also (still in the `newuser2` directory) make another new directory for your first repo directory e.g. `myfirstrepo` - both will show as being owned by newuser2

Instructions from the darcs link above say to copy the `darcs-wrapper.pl` script below into the `bin` directory. IMPORTANT Please note that the perl script below does not prevent users from uploading and running arbitrary programs with the darcs account privileges.

then handle addition of public key. Making an ed25519 key on local machine for darcs  Basically, do what you did to generate key for the regular unpriv user newuser, but do it for newuser2 
```
ssh-keygen -t ed25519 -f ~/.ssh/newuser2darcs_ed25519 -C "key for hosting as darcs user"

```

##### darcs-wrapper.pl
original code is at 
```
!/usr/bin/perl

sub fail {
	my ($msg) = @_;
	print STDERR "account restricted to darcs: ", $msg, "\n";
	exit 1;
}

# Since this script is called as a forced command, need to get the
# original command given by the client.

($command = $ENV{SSH_ORIGINAL_COMMAND})
	|| fail "environment variable SSH_ORIGINAL_COMMAND not set";

open LOG, '>>', '/home/darcs/wrapper-log';
$now = localtime;
print LOG $now, ": ", $command, "\n";
close LOG;

# Split the command string to make an argument list, and remove the first
# element (the command name; we'll supply our own);

@orig_argv = split /[ \t]+/, $command;

while (1) { 
	$orig_command = shift @orig_argv;

	if ($orig_command eq "cd") {
		$dir = shift @orig_argv;
		fail "bad cd sequence" 
			unless (shift @orig_argv) eq '&&';
		fail "illegal repo $dir"
			unless $dir =~ /^'(repos\/[a-zA-Z0-9\/]+)'$/;
		chdir $1;
	}
	elsif ($orig_command eq "darcs") {
		foreach my $arg (@orig_argv) {
			$arg =~ s/^(['"])(repos\/[a-zA-Z0-9\/]+)\1$/$2/;
		}
		last;
	}
# NB: there's no need to whitelist these if you have darcs 2 on both
# the client and the server side
#      elsif ($orig_command eq "scp") {
#          $ok = 0;
#          foreach $arg (@orig_argv) {
#              if ($arg eq '-t' || $arg eq '-f') {
#                  $ok = 1;
#                  last;
#              }
#          }
# 
#          fail "bad scp command"
#              unless $ok;
# 
#          last;
#      }
#      elsif ($orig_command =~ "sftp-server") {
#      $orig_command = '/usr/libexec/openssh/sftp-server';
#          last;
#      }
	else {
		fail "$orig_command not allowed"
	}
}

# Wipe the environment as a security precaution.  This might conceivably
# break something, but if it does you can filter the environment more
# selectively here.

%ENV = ();

exec $orig_command, @orig_argv;
```

