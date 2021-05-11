# Rsync Setup

So I'm stringing together steps between two different sites.  To capture on my lessons learned about my lessons learned (metaaaaa), I'm putting this in a markdown file, in lieu of a google drive doc. My OCD will be at ease :)


## Scope
Intended to show how to configure using the `plotman archive` feature on a Synology NAS and a Unbuntu chia plotter using `plotman v 0.2`.  `plotman` leverages the common `rsync` functionality.  The document will walk through configuring the NAS, the plotting rig, and how to validate functionality.

## Sources
Two main sources:

### Synology Walkthrough
https://silica.io/using-ssh-key-authentification-on-a-synology-nas-for-remote-rsync-backups/

### Rsync Setup Wiki
From the `Plotman` Wiki: https://github.com/ericaltendorf/plotman/wiki/Archiving#set-up-rsync-daemon-rsyncd

The main flow of this walkthrough is going to hop between the two main sources.  I'll try to use anchors and links where I can.  To avoid confusion - I'm not going to completely rewrite/identically copy a section's text from the page, but rather just have notes on the section and give additional context on how to collect the information the walkthrough is referencing.

# Setup Synology for Rsync

## Enable Synology NAS

This section pertains to "Public Key Authentication is disabled by default..." of [Synology Walkthrough](#synology-walkthrough)):

As a prerequisite - you'll want to have `ssh` enabled on your Synology NAS.  That can be turned on via the `Control Panel` --> `Terminal & SNMP` and making sure the `Enable SSH service` is checked.

1. Log into Synology using your admin credentials.  You're going to be creating/adding users and editing permissions, and enabling services.
2. Turn on `rsync` by navigating to the `Control Panel` clicking on the `rsync` tab and clicking `Enable rsync service`.  Be sure to validate that `Enable rsync account` is checked. Be sure to click `Apply`.  For the purposes of this walkthrough, we are going to assume that the rsync account on the NAS is called `rsync_account`
   ### Restart `rsync` Service
   Note: you'll have to restart thte 'rsync' service a couple of times during this process.  The easiest way to do that is to repeat this step of 
3. Click `edit rsync Account` and create an account if you haven't alreaady.
4. This *should* create a user in the `Users` section of hte `Control Panel`.  Some configs to validate about that account:
   1. `User Groups` tab: Account is a member of the `administrators` group (needed in order to ssh into the NAS)
   2. `Permissions`: has `R/W` permissions on the volume your plots belong.
   3. `Applications`: You might want to do a "least permissions" on this, but I haven't fully tested as of yet.  I've got `DSM`, `File Station`, `Universal Search` and `rsync` enabled.  You'll definitely at least need `rsync` enabled.


You can validate that the `rsync_account` has `R/W` permissions on your plotting folder, by going to the `Control Panel` --> `Shared Folder` --> finding your shared folder, selecting it, and clicking `Edit`.  Under the `Permissions` tab your account should have `Read/Write` checked.


# NAS SSH Configuration
So this is a bit sparse in the details from the wiki.

I used the [Synology Walkthrough](#synology-walkthrough) as a starting point.  Given that my steps differ slightly, I'll recreate them below

## NAS Configuration
1. Log into NAS via ssh from the command line of your local computer: `ssh rsync_account@NAS_IP_Addr`
2. Edit the SSH Service config file: `sudo vi /etc/ssh/sshd_config`.
   1. Uncomment the line `PubkeyAuthentication yes`
   2. Uncomment the line `AuthorizedKeysFile .ssh/authorized_keys`
3. Restart the `ssh` service via the Synology NAS `Control Panel` --> `Terminal & SNMP`. Uncheck the `Enable SSH Service` box and click `Apply`. Then click the checkbox again so it is enabled, and click `Apply`. Note: you will have lost your ssh connection.
4. You'll also want to make sure the home directory is setup for the `rsync` account.  Synology is weird, and puts home directories within `/var/services/homes/`, so make sure there is a `rsync_account` directory (i.e. `/var/services/homes/rsync_account`).  If not create one.
   1. While you're at it, create the `.ssh` directory and `authorized_keys` file.  
### Creating SSH Directories
   1. __`.ssh` Directory__: validating that you're in your home directory (`cd ~`), run `mkdir .ssh`
   2. __`authorized_keys` File__: `cd` into that `.ssh` directory now, and create the authorized keys file by running `touch authorized_keys`
# Ubuntu Plotter Configuration
1. SSH into your plotter.  This should be the account you are using to plot with - for this example I'll reference `chiaplotter` moving forward.
   1. Optional: Ubuntu leverages the `useradd` command (f needed).  There's tons of walkthroughs on how to run that command with a brief google search. So I won't belabor that.  Just one important note, after the account is created:
   2. Optional: Add the `chiaplotter` user to the `administrators` group with `sudo usermod -a -G groupName userName`
2. Validate you have the proper ssh directories.  Make sure you're in your home directory (`cd ~`), and run `ls -alR ../` 
   1. this should be the same thing as running `ls -alR /home`.  If you have a lot of users, that could get quite hairy quickly, in which case just run the command `ls -alR ~`. This will recursively look through your home directory.  You want to make sure you have a `.ssh` directory within your `chiaplotter` home directory, and an `authorized_keys` file within the `.ssh` directory. See the [Creating SSH Directories](#creating-ssh-directories) section above for detailed instructions.
3. As your `chiaplotter` user, run `ssh-keygen -t rsa -b 4096`.  `plotman` is not configured for authentication as of this release (`plotman 0.2`). That means you'll need to create an SSH key without a password.
   1. For each prompt, hit enter.  You'll be hitting enter 3 times. First time: accepting default file location; 2nd and 3rd Times: Not entering a password.  You should recieve output that says:
        ``` 
        Your identification has been saved in /home/chiaplotter/.ssh/id_rsa.
        Your public key has been saved in /home/chiaplotter/.ssh/id.rsa.pub.
        ```
   2. Make sure to run as `sudo` if you get error messages (that should fix most errors, if not, google is your friend)
4. Copy the public Key to the NAS with `ssh-copy-id rsync_account@NAS_IP_Address`. You'll be prompted for the password of the rsync_account, which should be the linux password set on the account.  Enter it and press `Enter`.
   1. ITS AN IMPORTANT DISTINCTION that you are taking the `chiaplotter` ssh key on your plotting rig, and copying it to the `rsync_account` on the NAS.
5. SSHing to the NAS (`ssh rsync_account@NAS_IP_Address`).  If prompted for a password, enter the linux password for the `rsync_account`. Change the file permissions:
   1. `chmod 0711 ~`
   2. `chmod 0711 ~/.ssh`
   3. `chmod 0600 ~/.ssh/authorized_keys`
6. Logout of the system by entering `logout`.  You should be returned to your plotting rig, shown by your command line changing to `chiaplotter@plottingRigHostname` (or whatever you called your device).
7. Validate connectivity by `ssh`ing again, and you should be able to login without a password. For example, from your `chiaplotter` account on your farming rig, run `ssh rsync_account@NAS_IP_Address`.  You should log in (your `username@hostname` will change at the beginning of the command line.  Additionally, you can run `ifconfig` and see that your ip has changed).

# NAS: Edit `/etc/rsyncd.conf`
This section pertains to the [Plotman Wiki - Section Setting Up rsyncd](https://github.com/ericaltendorf/plotman/wiki/Archiving#set-up-rsync-daemon-rsyncd).  Since the steps are good, I'm just providing additional context.
- "Remote Machine" is intended to mean the Synology NAS
- If you completed the steps above in the [Enable Synology NAS](#enable-synology-nas) section, you have completed the "install rsync" step
- Step 2, of creating the `/etc/rsyncd.conf` (on the NAS):
  - Should be done on the synology NAS via `ssh`
  - Be sure to create the file in the *absolute* path (`vi /etc/rsyncd.conf`), and not the *relative* path (`vi etc/rsyncd.conf`)
  - You can find the `path` to populate by running `ssh rsync@NAS_IP_Address df -aBK`. This is the same as line 52 of the `archive.py` [plotman source code](https://github.com/ericaltendorf/plotman/blob/5788e7fa2df2186ccb94db2d388766f9d52116a4/src/plotman/archive.py#L52).
    - The column of interest will be the `Volume` column.  Make note of the row that aligns to the Volume that your chia plots are mounted on (i.e. /`volume1`). This will be your `path` value.
  - I also got a few errors when I added some of the lines verbatim that were in there.

```
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
lock file = /var/run/rsync.lock
use chroot = no
port = 12000

# thing you put inside the brackets is considered your rsyncd_module in the plotman.yaml
[chia]
# The path variable is considered what rsyncd_path is in plotman.yaml
path = /volume1/chiaDir
comment = Chia
# UID is the username, GID is the group ID
uid = [username]
gid = [group]
read only = false
list = true
# Your plotting Device's IP Address
hosts allow = [IP_ADDR_OF_PLOTTER]
```
    - [username] - should be the same username you setup for `rsync_account`
    - [group] - I actually had a *different* group name than the wiki claiming it to be identical.  In order to find my proper gid I `ssh rsync_account@NAS_IP_Addr` and then ran `ls -al ../ | grep rsync_account | cut -d ' ' -f 3,4`. That should should show you the user owner and group owner of your home directory.  You want the group owner.
- Steps 3 and 4 - see the [Restart `rsync` Serivice](#restart-rsync-service) section to restart the service.  No need to permanently enable `rsync`, as you've already done that if you've turned on `rsync` in the `Control Panel` in the previous section.

## Validate Functionality & Collect Some Info
Now that you have the sshkey setup, you should be able to run the command `ssh rsync_account@NAS_IP df -aBK | grep /volume1`.  You can remove the `grep` command if you're unsure of hte mount point  of your drive.  But this is the command that will be run by `plotman` and the value that you put in `/volume1` is going to be used to filter down to find the specific drive, and the available drive space.

Additionally, on your plotting rig, as your `plottingaccount`, do the following steps to test functionality of `rsync`:
1. Change directories into the location of your `dst` location in plotman (i.e. `cd /mnt/ssd/plots_staging`)
2. Create a test file to copy over (i.e. `echo "This is a super simple test file, and it's small, but I sure hope it works" > testfile.txt`)
3. Run the following `rsync` command

`rsync --bwlimit=80000 --remove-source-files -P /mnt/ssd/plots_staging/testfile.txt rsync://rsync_account@NAS_IP_Address:12000/rsyncd_module`

Be sure to change the values for `/mnt/ssd/plots_staging/testfile.txt`, `rsync_account`, `NAS_IP_Address` and `rsyncd_module` to be specific to your configuration before running.  Some tips for these values:
- `rsyncd_module` - see below in the [Edit `plotman.yaml`](#edit-plotmanyaml) section to see where from the `/etc/rsyncd.conf` you can retrieve the module name.
- `/mnt/ssd/plots_staging/testfile.txt` - you want this to be the `dst` path you have set in your `plotman.yaml` file, and the `testfile.txt` to be the name of the file you jsut created.  Assuming you followed the steps above in this section, that should be the file name.

If you get any errors, they likely won't be super specific.  This is where the `rsyncd.conf` comes into play, specifically the avlue set for  `log file` - which in our default config was: `log file = /var/log/rsyncd.log`.  If you can't figure it out, `ssh` into the log NAS, and look at the log file (i.e. `tail -n 20 /var/log/rsyncd.log`).  The logs will have timestamps, in all likelihood it will be the logs all correlating with the last timestamp or two.  It's important you do this validation step! As if this command doesn't run as is, the `plotman` archiving feature will not work!


# Plotting Rig: Edit `plotman.yaml`
On your plotting rig, edit the `plotman.yaml` file (typically located: `~/.config/plotman/plotman.yaml`)

You will want to uncomment the `archive` section and set the values as such. You will need to personalize these values, but for the sake of simplicity, these align with the `/etc/rsyncd.conf` we just set above.  * It's also important to note that you will want to ensure the very first `archive:` line is uncommented, otherwise you'll get a plotman error.

```
        archive:
          rsyncd_module: chia
          rsyncd_path: /volume1
          rsyncd_bwlimit: 80000  # Bandwidth limit in KB/s
          rsyncd_host: NAS_IP_Address
          rsyncd_user: rsync_account
```

Notes about the information above:
- `rsyncd_module` is the same as the value inside the brackets of `/etc/rsyncd.conf` aka `[chia]`.  Do not include brackets in this.
- `rsync_path` - is actually the value you entered after in the `grep` [Validate Functionality & Collect Some Info](#validate-functionality--collect-some-info) section.
- `rsync_bwlimit` - that's the default in plotman, change if you'd like
- `rsyncd_host` - IP Address of your NAS
- `rsync_user` - the `rsync_account` that was created for this exercise, and used as the target in the `ssh-copy-id` command earlier.

Optional - you may also have to change your `dst` drive location in the `plotman.yaml`.  If it is currently set to your NAS which you mounted locally, you are going to DOS your network by causing `rsync` to think it needs to copy every file in the farming NAS to the farming NAS.  The purpose of `rsync` is to copy the file locally to your SSD, which is not only faster to write, but also frees up resources for your `plotman` to kick off another instance, while `rsync` can run in the background out of band to copy the file over.  This however, is especially personalized, and dependent upon your plotting rig setup.  Also, make sure whatever value you do set your `dst` location to, the only thing that goes in this directory are plot files.  No other references to other files or directories.


# Workaround/ Fix for Rsync Error

Some users may find that if you setup and enable `rsync`\ `archive`, afer setup, you get an error message, stating that no directories can be found.  You may need to edit the `archive.py` file of your plotman instance and remove an extra `/` - `vi venv/lib/python3.8/site-packages/plotman/archive.py`.

In order for me to get it to work, I had to change:

`df_cmd = ('ssh %s@%s df -aBK | grep " %s/"' %`

into:
` df_cmd = ('ssh %s@%s df -aBK | grep " %s"' %`

It removes the slash at the very end of the line, after the very last `%s`.




# Helpful Resources
Just a repo of links I used as reference
https://silica.io/using-ssh-key-authentification-on-a-synology-nas-for-remote-rsync-backups/
https://community.synology.com/enu/forum/1/post/136213
