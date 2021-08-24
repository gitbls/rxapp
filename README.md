# rxapp &mdash; Seamless password-less remote X11 starter

## Overview

*rxapp* is a bash script that makes it really easy to seamlessly start X11 applications across hosts on a LAN. For instance, if you have an X Server running on your desktop and you want to have xterms running across several other systems, all displaying on your desktop X Server, *rxapp* is the easiest way to do this, since they can be easily started from the bash command line, or in a script.

To me, seamless means that it works silently with no typing and no extra or undesirable output. However, when things *don't* work, getting *some* output is critical to figuring out the problem. If you have problems, *rxapp* has built-in capabilities to provide debug output that can help you get your X11 applications started.

By default, *rxapp* starts apps using SSH X11 Forwarding. This means that the communication between the X11 app and the X server is encrypted over the SSH channel. Apps are started using SSH X11 Forwarding by default.

Apps can also display using direct TCP connections, either for performance, or for displaying on a third system (rxapp-starting system, X11 client app-running system, and X server system).

Most of the details below are around setting up SSH password-less login, which you should really have anyhow. 

## Topics
* *rxapp* Installation
* Set up SSH for password-less login
* SSH forwarding
* Example bash aliases
* Command line and Special *rxapp* control features
* Configuring the X Server for direct TCP connections
* Problem-solving hints
* Notes for *putty* users

## *rxapp* Installation

Installation is super-simple. Download https://github.com/gitbls/rxapp/blob/master/rxapp and put it someplace convenient on your system. The best places to put it are /usr/local/bin (so it's available to all users) or ~/bin (available for the current user only).
```
curl -L https://raw.githubusercontent.com/gitbls/rxapp/master/rxapp -o /usr/local/bin/rxapp
   - OR -
curl -L https://raw.githubusercontent.com/gitbls/rxapp/master/rxapp -o ~/bin/rxapp
```
## Set up SSH for password-less login

**NOTE:** There is nothing in this section that is specific to *rxapp*. If you have used another method to generate your SSH keys and password-less login already works, then you can skip the *Create the key* and *Propagate the public key* steps. You do, however, need to ensure that you're set up to use `ssh-agent` or `keychain` for totally seamless operation.

Password-less rxapp use requires that ~/.ssh on the rxapp-starting host and the X11 client system
both have ssh set up correctly as follows:
* The rxapp-starting host has the public and private keys in ~/.ssh, and ssh-agent or keychain installed and configured
* The client where the X11 app runs has ~/.ssh/authorized_keys enabled with the correct SSH public key

The steps below will guide you through this.

* **Create the key** This can be done on any host, but the keys must be deployed correctly
      on both the rxapp-starting host and the target host. Easiest to do this on the rxapp-starting host.
      * `ssh-keygen -t rsa -C keycomment`
        * Creates an ssh key pair with the comment *keycomment*
          * By default the comment in the public key will be 'user@host'
          * Specifying a useful *keycomment* makes it easier to identify the key when you look at 
            `~/.ssh/authorized_keys` and is critical if you use multiple keys so you can easily identify each one.
      * When you create the key you'll be prompted for a passphrase for the key. This is highly recommended for increased security. `keychain` and/or `ssh-agent` make using passphrase-protected keys quite tolerable.
      * If you use `keychain` or `ssh-agent` you'll typically only enter it once per system reboot
      * Make sure that you save the new key in `~/.ssh` (aka `/home/yourusername/.ssh`)
      * Expert tip: Can also add -O source-address=ip.ad.dr.0/24 for instance, to restrict the key to those
        source IP addresses
      * See `man ssh-keygen` for a complete list of options and the complete ssh-keygen help

* **Propagate the public key** as needed to the target host(s) (X11 app running host)
      * `ssh-copy-id -i identity-file [user@]otherhost`
      * The rxapp-starting host must have both the public (keyname.pub) and private (keyname) keys in ~/.ssh
      * If needed, copy the private key manually to `~/.ssh` on the rxapp-starting host and protect it 600

* If you want to use the rxapp *run as different user* feature, use `ssh-copy-id` to propagate the public key to those accounts as well

* **Use ssh-agent** on the rxapp-starting host to enable password-less ssh key usage
      *keychain is a simple way to manage and use ssh-agent*. Either use keychain, or ssh-agent by itself. But not both.

* **Using keychain** to simplify use of ssh-agent
    * `sudo apt install keychain`
    * Add to your .bashrc at the correct spot (try the end if you're not sure). If your keys have a password on them (and they should), you will be prompted for the password. This is true whether you're using `keychain` or `ssh-agent/ssh-add` directly.
```
    # keystart and keystop aliases make it easy to stop and restart ssh-agent
    # Replace 'sshkeyfile' with the filename(s) of the keys you want to use (just the filename, not full path)
    alias keystart='keychain -q --nogui [--ignore-missing] --agents ssh sshkeyfile1 sshkeyfile2 ; source ~/.keychain/$(hostname)-sh'
    alias keystop='keychain -q --stop all'  # Use keystop to stop the running ssh-agent (it's ok to not stop it)
    keystart                                # If your private keys have passwords, you will be prompted to enter them
                                            # The optional `--ignore-missing` switch eliminates messages about missing keys.
```
* **Using ssh-agent by itself** without keychain
    * There are many tutorials and permutations for using ssh-agent
    * One way is to add to your .bashrc at an appropriate location. As with `keychain`, if your keys have a password on them you will be prompted for the password.
```
eval `ssh-agent`    # Start ssh-agent
ssh-add sshkeyfile  # Add the private key identity to ssh-agent
 (For example  ssh-add ~/.ssh/id_rsa)
```
If you have more than one *sshkeyfile*, you can list them all on the `ssh-add` command line

* **Test your work!**
If you have set up password-less ssh correctly, you will be able to simply:
```
ssh otherhost 
```
This should seamlessly login to *otherhost* with no password prompt for the account or the ssh key.

**NOTE:** Once set up, you will still be prompted by SSH when you connect to a host by name or IP address that is not in your ~/.ssh/known_hosts file. Once the host is added to your known_hosts file there will be no further prompts for that system unless it's SSH host key changes (e.g., you rebuild the system).

## SSH Forwarding

   * **SSH X11 Forwarding is used by default with *rxapp***. The remote application will be displayed on the X Server of the rxapp-starting host
   * *rxapp* will add the ssh switch "-X" to enable X11 Forwarding
   * *Expert tip:* Define the environment variable RXSSHSW with any desired additional ssh switches:
```
RXSSHSW="-X -Y" rxapp myhost myapp
```
   * **To use a direct TCP connection to an X Server**, see 5) below and set the environment variable RXDISPLAY:
```
RXDISPLAY=somehost.local:0 rxapp otherhost someX11app
```

Since it's more secure, **why would I not use SSH X11 Forwarding**? Additional CPU cycles are consumed encrypting and decrypting the X11 network packets. I've observed about 10% overhead on the X Server system for a single app continuously producing output. Of course, running without encryption on anything other than a Home Network is not recommended.

## Example .bashrc aliases

* Add something like this in your .bashrc on the rxapp-starting host
```
function rxterm() { (rxapp $1 xterm $2 $3 $4 $5 $6 $7 $8) ; } ; declare -fx rxterm
```
* Similarly, add functions to .bashrc for other commonly-used X11 apps. For instance, to add chromium:
```
function rxchromium() { (rxapp $1 chromium $2 $3 $4 $5 $6 $7 $8) ; } ; declare -fx rxchromium
```
* Optionally, include RXUSER to run a remote X11 service as a different user (See 2a below)
```
function rxotheru() { (declare -x RXUSER="other" ; rxapp $1 xterm $2 $3 $4) ; } ; declare -fx rxotheru
function rxtsu() { (declare -x RXUSER="root" ; rxapp $1 xterm $2 $3 $4) ; } ; declare -fx rxtsu
```
After editing your .bashrc, you can test your new commands after by first using `source .bashrc` to update your definitions in your current bash session.

## Command line Special rxapp control features

* Use this on the command line to run an X11 application an a remote host
```
rxapp hostname cmdname arg1 arg2 arg3 arg4 ... arg8
```

These environment variables may be used to control rxapp's operation. You can either prefix your *rxapp* command line with the definition so that it only affects that command, or you can put define them globally on the command line or in your .bashrc by using the `declare -x` command.

For instance, to prefix a command with an environment variable use:
```
RXDISPLAY=myxserver.local:0 rxterm myothernode
RXUSER=alterego rxterm someothernode
```
To add definitions to your .bashrc you could add lines similar to this:
```
declare -x RXDISPLAY="myxserver.local:0"
declare -x RXUSER="otherme"
```

* **RXDISPLAY**&mdash;The address of the X display in the form host:0
    * Using RXDISPLAY creates an unencrypted TCP network connection without using SSH X11 Forwarding; this is generally OK for a 'home LAN'
    * The default is to use SSH X11 Forwarding over the SSH connection to the current host (where rxapp is running). RXDISPLAY is not used with SSH X11 Forwarding.
    * RXDISPLAY can be set to be any host with X server running that has been configured per the section *Configuring the X Server for direct TCP connections*
* **RXGappname**&mdash;Specify the geometry for an app if defined
    * e.g., RXGXTERM for *xterm *, RXGCHROMIUM for *chromium*, RXGXEYES for *xeyes
    * Default is whatever the X defaults are for that app
    * Standard X11 apps use "-geometry WIDTHxHEIGHT". e.g., `RXGXTERM=-"geometry 140x60"`
    * Other apps might have a different syntax, for example
        * lxterminal uses `--geometry=WIDTHxHEIGHT` so `RXGLXTERMINAL="--geometry WIDTHxHEIGHT"`
    * See the man page or help for the app in question
* **RXSSHSW**&mdash;Specify SSH switches to use for X11 forwarding
    * Default is "-X"
* **RXUSER**&mdash;The username for the system where the X client app will be run
    * Default is same as current user

These environment variables can help debug rxapp/ssh issues
* **RXDBGOUT**&mdash;Specify file for ssh console output. /dev/null if not specified
* **RXDBGRMT**&mdash;Specify file for debugging output on remote system. Currently gets output of printenv (useful to see DISPLAY)
* **RXDBGSSH**&mdash;Specify additional switches for ssh command. For instance use "-vvv" to get full ssh debug output

## Configuring the X Server for direct TCP connections

In order to use RXDISPLAY and direct TCP connections, there are two additional extra steps:
* The X Server to which the app is going to display must have tcp enabled in the Display Manager
    * **lightdm:** `sudo nano /etc/lightdm/lightdm.conf`. Search for the line "#xserver-allow-tcp=false" and add a new line after that: `xserver-allow-tcp=true`
    * **xdm:** `sudo nano /etc/X11/xdm/Xservers`. Find the line that enables the console Xserver (by default the line starts with ":0"), and change "-nolisten tcp" to "-listen tcp"

After you edit the Display Manager configuration, restart the system for the changes to take effect. (You could restart lightdm or xdm, but doing so will probably kill your logged-in session, so you might as well reboot)

* The X server must have access control enabled, Use the xhost command to enable access control as desired:
```
     xhost +                  # Enables Xserver access from any host
     xhost +hostname          # Enables Xserver access from the specified hostname
```
These must be done on every system reboot. There are mechanisms to make this permanent and automatic. See `man xhost` for complete details.

## Problem-solving hints

* It's critical to test and ensure you can do password-less ssh to the remote X11 client system as your first step.
* Does the X11 app you're trying to run actually exist on the remote client system?
* Have you connected to the host with the X11 app previously? The host must exist in the rxapp-starting host ~/.ssh/known_hosts. If it's not there, simply ssh to the target system once with a password-less login, and ssh should add it to the known_hosts file
* Try using RXDBGOUT to get the ssh output, perhaps in conjuction with RXDBGSSH
```
   RXDBGOUT=afile rxapp remotehost /bin/xeyes
   RXDBGOUT=afile RXDBGSSH="-vvv" rxapp remotehost /bin/xeyes
```
After using one of the above commands, review the contents of *afile* on the rxapp-starting host.
* Ensure that the remote client is getting the correct DISPLAY variable
```
   RXDBGRMT=afile rxapp remotehost /bin/xeyes
```
And then review the file 'afile' on the remote host. Check that the value for DISPLAY is either "localhost:nn.0" (the default if RXDISPLAY is not set) or the same value you set for RXDISPLAY. 
* If using RXDISPLAY make sure that you have correctly made the appropriate changes from the section *Configuring the X Server for direct TCP connections*

## Note for *putty* users

The best practice is to  use a single tool to produce your SSH keys, and the best tool to use is `ssh-keygen`. Once you have generated your key(s), you can use puttygen to convert them to the format required by putty. This will ensure that your keys have the best fidelity with Linux SSH, yet also work with putty.

That said, if you have been using putty and have an ssh key established in putty and spread around to multiple host systems, you can convert it to a key for use with Linux SSH as follows:

* On **Windows**
    * run *puttygen*
    * Use the *Load button* and select your key (filename.ppk)
    * Enter the password for the key if prompted for it
    * Select *Export OpenSSH Key (New Format)* in the *Conversions* menu
    * Navigate to your ~/.ssh directory in the *Save As* dialog box if needed
    * Give the created private key a name and save it
    * Set the file protection on the newly-created private key to 600: `chmod 600 keyfile`
    * This newly-created private key can then be used when connecting with the `ssh` command to your remote system.

* On **Linux**
    * *puttygen* is a command-line tool
    * `puttygen mykey.ppk -o mysshkey -O private-openssh-new` (substitute for *mykey* and *mysshkey* as appropriate)
        * Set the file protection on the newly-created private key to 600: `chmod 600 keyfile`
    * The generated *mysshkey* private key can now be used with the Linux `ssh` command,`keychain`, and/or `ssh-agent`

**NOTE:** Some Linux distros come with older versions of putty and puttygen that do not work with putty PPK file version 3. If Linux puttygen complains "PuTTY key format too new", you can use puttygen on Windows (and I assume MacOS...can't test this) to create a PPK file version 2 by loading the PPK 3 key, setting PPK version 2 in the *Parameters for saving key files* menu item in the *Key* menu, and save the version 2 file with a new filename. This version 2 file can then be used with the older Linux puttygen.

**If you generate keys using `ssh-keygen`**, as detailed above, and want to use it with `putty`, here's how:

* On **Windows**
    * run `puttygen`
    * Select *Import* in the *Conversions* menu
    * Select the private key (without .pub) you generated with `ssh-keygen` and click Open
    * puttygen will load the key and populate the various fields in the puttygen window
    * Change the *Key comment* field so that it matches the comment field you used in `ssh-keygen`. This isn't mandatory, but in computing, naming consistency is goodness.
    * Select *Save private key* to save the private key in *putty's* PPK format

* On **Linux**
    * `puttygen myprivatesshkey -O private -o myprivatesshkey.ppk`
    * The use myprivatesshkey.ppk in *putty* (SSH configuration, then Auth, then *Private key for authentication)

Once you have *rxapp* working, you don't really need to use *putty* any more especially if you haven't done any putty customizations. A real Linux *xterm*, *lxterminal*, or any one of a number of other Linux X11 terminal apps all provide a better experience than *putty*.
