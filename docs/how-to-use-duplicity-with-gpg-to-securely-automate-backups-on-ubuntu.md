How To Use Duplicity with GPG to Securely Automate Backups on Ubuntu | DigitalOcean
===================================================================================

By: Justin Ellingwood

### Introduction

---

Duplicity is a versatile local and remote backup program that can implement a number of transfer protocols and third-party storage solutions.

In this guide, we will discuss how to install duplicity on an Ubuntu 12.04 VPS. We will be installing from source and then configuring it to take advantage of GPG encryption.

To follow along, you will need access to two machines, one Ubuntu 12.04 VPS, which will be backed up, and a second Linux machine or VPS of any variety that can be accessed by SSH.

How To Install Duplicity from Source on Ubuntu
----------------------------------------------

---

We are using an Ubuntu 12.04 VPS for this guide. The duplicity package in the default repositories is outdated, and actually suffers from some problems with connecting to remote hosts due to a change in the backend.

We will avoid these problems by getting the source files and installing manually.

Log into the Ubuntu 12.04 VPS that you will be backing up, **as root**.

### Install the Prerequisite Packages

---

Although we are installing duplicity from source, we will get the prerequisites from the default Ubuntu repositories.

Update the source database and then install the needed packages with these two commands:

    apt-get update
    apt-get install ncftp python-paramiko python-pycryptopp lftp python-boto python-dev librsync-dev

This installs a number of different handlers for transferring the data to the remote computer. We won't be using most of these within this guide, but they are good options to have.

### Download and Install Duplicity from Source

---

The duplicity source files are housed at at [launchpad.net](http://launchpad.net). We will download them to the root user's home directory.

    cd /root
    wget http://code.launchpad.net/duplicity/0.6-series/0.6.22/+download/duplicity-0.6.22.tar.gz

Unpack the source and move into the package directory that is created:

    tar xzvf duplicity*
    cd duplicity*

Next, we will complete the actual installation with the following command:

    python setup.py install

Because this is a package installed from source, it will be placed in the `/usr/local/bin/` directory.

Create SSH and GPG Keys
-----------------------

---

Our configuration of duplicity will use two different kinds of keys to achieve a nice intersection between convenience and security.

We will use SSH keys to securely authenticate with the remote system without having to provide a password. We will also use GPG to encrypt the data before we transfer it to the backup location.

### Create SSH Keys

---

We will generate an RSA encrypted SSH key for our root user to allow password-less logins to the machine that will host the backups.

If you have not done so already, make sure you have a root password configured on the machine you will be transferring the data to. You can do this by logging into the machine as root (through SSH or the Console Access button on the droplets page if this is a VPS) and issuing this command:

    passwd

Back in the droplet with duplicity, we will generate a key pair with the following command:

    ssh-keygen -t rsa

Press **Enter** at the prompts to create a password-less SSH key with the default settings.

Transfer it to the system that will host your backups with this command:

    ssh-copy-id root@backupHost

Answer **yes** to accept the unverified host, and then enter the root password of the remote system to transfer your public key.

Test that you can now log in without a password from your duplicity droplet by issuing:

    ssh root@backupHost

You should be logged in without having to provide any further credentials.

While you are logged in through SSH, create the directory structure that will house our backup files:

    mkdir -p /remotebackup/duplicityDroplet

You can name the directory anything you'd like, but remember the value so that you can specify it later.

When you are finished, exit back out into your duplicity droplet:

    exit

### Create GPG Keys

---

We will be using GPG for extra security and encryption. The commands will store our keys in a hidden directory at `/root/.gnupg/`:

    gpg --gen-key

You will be asked a series of questions that will configure the parameters of the key pair.

    Please select what kind of key you want:
       (1) RSA and RSA (default)
       (2) DSA and Elgamal
       (3) DSA (sign only)
       (4) RSA (sign only)
    Your selection?
    RSA keys may be between 1024 and 4096 bits long.
    What keysize do you want? (2048)
    Requested keysize is 2048 bits
    Please specify how long the key should be valid.
             0 = key does not expire
          <n>  = key expires in n days
          <n>w = key expires in n weeks
          <n>m = key expires in n months
          <n>y = key expires in n years
    Key is valid for? (0)
    Key does not expire at all
    Is this correct? (y/N) y

Press **enter** to accept the default "RSA and RSA" keys. Press **enter** twice again to accept the default keysize and no expiration date.

Type **y** to confirm your parameters.

    You need a user ID to identify your key; the software constructs the user ID
    from the Real Name, Comment and Email Address in this form:
        "Heinrich Heine (Der Dichter) <heinrichh@duesseldorf.de>"

    Real name: Your Name
    Email address: your\_email@example.com
    Comment:
    You selected this USER-ID:
        "Your Name \<your\_email@example.com\>"

    Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o

Enter the name, email address, and optionally, a comment that will be associated with this key. Type **O** to confirm your information.

Next, you will be setting up a passphrase to use with GPG. Unlike with the SSH keys, where we defaulted to no passphrase allow duplicity to operate in the background, you should supply a passphrase for this step to allow secure encryption and decryption of your data.

    Enter passphrase:
    Repeat passphrase:

At this point, you will be asked to generate entropy. Entropy is basically a word that describes how much unpredictability is in a system. Your VPS needs entropy to create a key that is actually random.

    We need to generate a lot of random bytes. It is a good idea to perform
    some other action (type on the keyboard, move the mouse, utilize the
    disks) during the prime generation; this gives the random number
    generator a better chance to gain enough entropy.

    Not enough random bytes available.  Please do some other work to give
    the OS a chance to collect more entropy! (Need 280 more bytes)

If you need some help creating entropy, there is a guide on [using Haveged to generate entropy](https://www.digitalocean.com/community/articles/how-to-setup-additional-entropy-for-cloud-servers-using-haveged) here. In my experience, just installing some packages from apt is enough to generate the entropy needed. SSH in with a new terminal to do this.

When you've generated enough random pieces of information, your key will be created:

    gpg: /root/.gnupg/trustdb.gpg: trustdb created
    gpg: key 05AB3DF5 marked as ultimately trusted
    public and secret key created and signed.

    gpg: checking the trustdb
    gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
    gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
    pub   2048R/05AB3DF5 2013-09-19
          Key fingerprint = AF21 2669 07F7 ADDE 4ECF  2A33 A57F 6998 05AB 3DF5
    uid                  Your Name
    sub   2048R/32866E3B 2013-09-19

The part highlighted above is your public key ID. You will need this later to encrypt the data you will be transferring.

If you forget to write down your public key ID, you can get it again by querying the gpg keyring:

    gpg --list-keys

---

    /root/.gnupg/pubring.gpg
    ------------------------
    pub   2048R/05AB3DF5 2013-09-19
    uid                  Your Name <your_email@example.com>
    sub   2048R/32866E3B 2013-09-19

We now have all of the necessary components in place to securely backup using duplicity.

How To Use Duplicity
--------------------

---

### Run an Initial Test

---

We will run an initial test of our duplicity system by creating a folder of dummy files to back up. Run the following commands:

    cd ~
    mkdir test
    touch test/file{1..100}

This creates a directory called `test` in the root home directory. It then fills the directory with files numbered 1-100.

We will move the files to the remote server, first without the GPG key we generated. We will use "sftp", which is a secure protocol included with SSH that replicates the functionality of ftp.

    duplicity /root/test sftp://root@backupHost//remotebackup/duplicityDroplet

Notice the double slashes between the remote host and the file path. This is because we are specifying an absolute path. If it was a relative path from the default directory that sftp puts us in, we could use only one slash.

You will be asked to accept the remote host and then asked to create and confirm a key to use to encrypt the data. As you can see, GPG will still be used unless we specifically tell it not to. The only difference is that we are not using the keys we created, we could type in any password here.

    Import of duplicity.backends.dpbxbackend Failed: No module named dropbox
    The authenticity of host '162.243.2.14' can't be established.
    SSH-RSA key fingerprint is 1f:4b:ae:1c:43:91:aa:2b:04:5b:a4:8e:cd:ea:e6:60.
    Are you sure you want to continue connecting (yes/no)? yes
    Local and Remote metadata are synchronized, no sync needed.
    Last full backup date: none
    GnuPG passphrase:
    Retype passphrase to confirm:

The backup then runs and you will be presented with statistics when the process completes:

    No signatures found, switching to full backup.
    --------------[ Backup Statistics ]--------------
    StartTime 1379614581.49 (Thu Sep 19 18:16:21 2013)
    EndTime 1379614581.60 (Thu Sep 19 18:16:21 2013)
    ElapsedTime 0.11 (0.11 seconds)
    SourceFiles 101
    SourceFileSize 4096 (4.00 KB)
    NewFiles 101
    NewFileSize 4096 (4.00 KB)
    DeletedFiles 0
    ChangedFiles 0
    ChangedFileSize 0 (0 bytes)
    ChangedDeltaSize 0 (0 bytes)
    DeltaEntries 101
    RawDeltaSize 0 (0 bytes)
    TotalDestinationSizeChange 1022 (1022 bytes)
    Errors 0
    -------------------------------------------------

If we SSH into our remote system, we can see that the backups completed successfully:

    ssh root@backupHost
    cd /remotebackup/duplicityDroplet
    ls

    duplicity-full.20130919T181705Z.manifest.gpg
    duplicity-full.20130919T181705Z.vol1.difftar.gpg
    duplicity-full-signatures.20130919T181705Z.sigtar.gpg

These files contain the backup information. Since this was just a test, we can delete them by running:

    rm duplicity*

Exit back into the duplicity droplet:

    exit

We can now remove the test directory and all of its contents:

    rm -r /root/test

### Create Your First Backup

---

We will create our first real backup by using the following general syntax:

    duplicity --encrypt-key key\_from\_GPG --exclude files\_to\_exclude --include files\_to\_include path\_to\_back\_up sftp://root@backupHost//remotebackup/duplicityDroplet

We will back up our entire root directory, with the exception of `/proc`, `/sys`, and `/tmp`. We will use the GPG key we created. We do this by specifying the ID within the command, and preceding the command with the passphrase:

    PASSPHRASE="passphrase\_for\_GPG" duplicity --encrypt-key 05AB3DF5 --exclude /proc --exclude /sys --exclude /tmp / sftp://root@backupHost//remotebackup/duplicityDroplet

The command above will take some time. Because this is the first time we've run the backup, duplicity will create a full back up. Duplicity divides the chunks of data into volumes to simplify the file transfers.

    --------------[ Backup Statistics ]--------------
    StartTime 1379621305.09 (Thu Sep 19 20:08:25 2013)
    EndTime 1379621490.47 (Thu Sep 19 20:11:30 2013)
    ElapsedTime 185.38 (3 minutes 5.38 seconds)
    SourceFiles 33123
    SourceFileSize 813465245 (776 MB)
    NewFiles 33123
    NewFileSize 813464221 (776 MB)
    DeletedFiles 0
    ChangedFiles 0
    ChangedFileSize 0 (0 bytes)
    ChangedDeltaSize 0 (0 bytes)
    DeltaEntries 33123
    RawDeltaSize 802133584 (765 MB)
    TotalDestinationSizeChange 369163424 (352 MB)
    Errors 0
    -------------------------------------------------

On a fresh droplet, my configuration created 15 volumes which were transfered to the remote system.

Because we now have a full backup on the remote system, our next backup will automatically be an incremental backup. These are faster and require less data transfer. My first run took over three minutes, while my incremental backup took less than eight seconds.

    --------------[ Backup Statistics ]--------------
    StartTime 1379621776.23 (Thu Sep 19 20:16:16 2013)
    EndTime 1379621783.80 (Thu Sep 19 20:16:23 2013)
    ElapsedTime 7.57 (7.57 seconds)
    SourceFiles 33128
    SourceFileSize 820560987 (783 MB)
    NewFiles 11
    NewFileSize 12217723 (11.7 MB)
    DeletedFiles 3
    ChangedFiles 1
    ChangedFileSize 600 (600 bytes)
    ChangedDeltaSize 0 (0 bytes)
    DeltaEntries 15
    RawDeltaSize 12197851 (11.6 MB)
    TotalDestinationSizeChange 12201207 (11.6 MB)
    Errors 0
    -------------------------------------------------

To force another full backup, you can add the "full" command to the duplicity call prior to any options:

    PASSPHRASE="passphrase_for_GPG" duplicity full --encrypt-key 05AB3DF5 --exclude /proc --exclude /sys --exclude /tmp / sftp://root@backupHost//remotebackup/duplicityDroplet

### Restore a Backup

---

Duplicity makes restoring easy. You can restore by simply reversing the remote and local parameters.

We don't need the encrypt-key option since we are only decrypting data. We also don't need the exclude parameters because they aren't included in the backup in the first place.

For instance, if we wanted to restore the data we just backed up, in its entirety, we could use this command:

    PASSPHRASE="passphrase\_for\_GPG" duplicity sftp://root@backupHost//remotebackup/duplicityDroplet /

Perhaps a safer option is only restoring the files or directories that you need. You can do this by adding an option to the above command:

    PASSPHRASE="passphrase_for_GPG" duplicity --file-to-restore /path/to/file sftp://root@backupHost//remotebackup/duplicityDroplet /path/to/restore/file

Make sure you test your ability to restore correctly, so that you do not run into problems when you are in a dire situation.

Automate Backups
----------------

---

We can automate duplicity by creating a few cron jobs. Click here to [learn more about how to configure cron](https://www.digitalocean.com/community/articles/how-to-schedule-routine-tasks-with-cron-and-anacron-on-a-vps).

### Create a Passphrase File

---

We will create a protected file to store our GPG passphrase so that we do not have to put it directly in our automation script.

Go to the root user's home directory and create a new hidden file with your text editor:

    cd /root
    nano .passphrase

The only thing we will need to put in this file is the passphrase specification that you have been preceding your duplicity commands with:

    PASSPHRASE="passphrase\_for\_GPG"

Save and close the file.

Make it only readable by root by executing:

    chmod 700 /root/.passphrase

### Set Up Daily Incremental Backups

---

We will set up duplicity to create daily incremental backups. This will keep our backups up-to-date.

Scripts listed in `/etc/cron.daily` are run once a day, so this is the perfect place to create our backup script.

Navigate to that folder and create a file called `duplicity.inc`:

    cd /etc/cron.daily
    nano duplicity.inc

Copy the following bash script into the file. Replace the duplicity command with the command you would like to use to backup your system.

    #!/bin/sh

    test -x $(which duplicity) || exit 0
    . /root/.passphrase

    export PASSPHRASE
    $(which duplicity) --encrypt-key 05AB3DF5 --exclude /proc --exclude /sys --exclude /tmp / sftp://root@backupHost//remotebackup/duplicityDroplet

Save and close the file.

Make it executable by typing the following command:

    chmod 755 duplicity.inc

Test it by calling it:

    ./duplicity.inc

It should complete without any errors.

### Set Up Weekly Full Backups

---

Incremental backups build off of full backups. This means that they will get increasingly unwieldy as changes stack up. We will configure weekly full backups to refresh the base.

We will do this by creating a similar script within the `/etc/cron.weekly` directory.

Navigate to the directory and create a new file:

    cd /etc/cron.weekly
    nano duplicity.full

Copy the following bash script into the file. Notice that we included the "full" command to force duplicity to run a full backup.

    #!/bin/sh

    test -x $(which duplicity) || exit 0
    . /root/.passphrase

    export PASSPHRASE
    $(which duplicity) full --encrypt-key 05AB3DF5 --exclude /proc --exclude /sys --exclude /tmp / sftp://root@backupHost//remotebackup/duplicityDroplet

We are also going to add an additional duplicity command on the end to clean out old backup files. We will keep a total of three full backups and their associated incremental backups.

Add this to the end of the file

    $(which duplicity) remove-all-but-n-full 3 --force sftp://root@backupHost//remotebackup/duplicityDroplet

Save and close the file.

Make it executable with the following command:

    chmod 755 duplicity.full

Test it by calling:

    ./duplicity.full

It should do a full backup and then remove any files necessary.

Conclusion
----------

---

You should now have a fully operational, automated backup solution in place. Be sure to regularly validate your backups in order to not fall victim to a false sense of security.

There are many other backup tools available, but duplicity is a flexible, simple solution that will fulfill many users' needs.

By Justin Ellingwood
