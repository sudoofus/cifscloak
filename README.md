# Cifscloak
### Mount cifs shares using encrypted credentials

Cifscloak is a simple python script for encrypting and storing cifs credentials.  
In short, it is a wrap around 'mount -t cifs' with user credentials encrypted and stored in a sqlite database.  

### Tested so far  
Ubuntu 20.04, Redhat, Centos, Oracle Linux, python3.8.

### Quick start:  

1/ Install

Latest:
`sudo pip3 install git+https://github.com/sudoofus/cifscloak.git`  
`sudo python3 -m pip install cifscloak`  

Script installs to:  
/usr/local/bin/cifscloak  
Included an easier console script name at v1.0.29  
/usr/local/bin/cifsfs  
Now you can type 'cifsfs' command instead of 'cifscloak'  

2/ Create an encrypted cifstab and add cifs mounts.  
cifscloak addmount --name <give_name_to_mount> --sharename <share_name> --mountpoint <mount_point> --i <cifs_server_address> --options <cifs_mount_options> --user cifsusername

`sudo cifscloak addmount -n films -s myfilms -m /mnt/films -i myfileserver -o "ro" -u cifsuser`  
`Password:`  

`sudo cifscloak addmount -n games -s mygames -m /mnt/games -i myfileserver -u cifsuser`  
`Password:`

3/ Mount one or more cifs shares.  
cifscloak mount --names <name1> <name2>  
Or mount all shares.  
cifscloak mount -a  

`sudo cifscloak mount -n films games`

4/ Unmount one or more cifs shares.  
cifscloak mount -u --names <name1> <name2>  
Or unmount all cifs shares named in cifstab  
cifscloak mount -u -a  

`sudo cifscloak -u -n films games`

5/ List cifs share aliases stored in the cifstab.  

`sudo cifscloak listmounts`

  {  
      "films": {  
          "name": "films",  
          "host": "myfileserver",  
          "share": "myfilms",  
          "mountpoint": "/mnt/films",  
          "options": "ro"  
      },  
      "games": {  
          "name": "games",  
          "host": "myfileserver",  
          "share": "mygames",  
          "mountpoint": "/mnt/games",  
          "options": ""  
      }  
  }  

6/ Remove one or more cifs shares from the cifstab.  
cifscloak removemounts --names <name1> <name2>

`sudo cifscloak removemounts -n films games`

7/ Create systemd file.  

`sudo cifscloak systemdfile -a`

#Generated by cifscloak  
[Unit]  
After=multi-user.target  
Description=cifscloak  
  
[Service]  
Type=oneshot  
RemainAfterExit=yes  
ExecStart=/usr/local/bin/cifscloak mount -a -r 6  
ExecStop=/usr/local/bin/cifscloak mount -u -a  
  
[Install]  
WantedBy=multi-user.target  

`sudo cifscloak systemdfile -a > /etc/systemd/system/cifscloak.service`  
`systemctl enable cifscloak`  
`systemctl start cifscloak`  

### Uninstall
`python3 -m pip uninstall cifscloak`  
`rm /root/.cifstab`  

### Synopsis
This utility should be run through sudo or directly as root.

When executed as the root user the following directory and files are created the first time that the script is executed:  
> 0755 /root/.cifstab/  
> 0400 /root/.cifstab/.keyfile  
> 0644 /root/.cifstab/.cifstab.db  

cryptography.fernet is used to generate the .keyfile and take care of encryption.  
sqlite3 is used to store encrypted cifs information into /root/.cifstab/.cifstab.db

Of course if you have the .keyfile and cifstab.db it is really easy to decrypt and display the passwords.  
Be sure that the cifscloak script is not writable by anyone except root otherwise it would be trivial for a user to modify the script to have it write out the passwords somewhere  next time the script is executed.

For example:  
> 550 /usr/bin/cifscloak

The .cifstab directory is created in the home (~) directory of the user running the script, the directory can also be overridden with environment variable 'CIFSCLOAK_HOME' e.g.  
> export CIFSCLOAK_HOME='/home/sudoofus'  

This will have the effect of creating the following .cifstab directory  
> /home/sudoofus/.cifstab/  

Alternatively the .cifstab directory is created in the home directory of whichever user is running the script.  

### Example
`sudo cifscloak addmount -n films -s myfilms -m /mnt/films -i myfileserver -o "ro" -u cifsuser`  
`Password:`  

`sqlite> .schema cifstab`

```CREATE TABLE cifstab(
        name,
        address,
        sharename,
        mountpoint,
        options,
        user,
        password,
        PRIMARY KEY (name)
        );
```

Everything apart from the name rests encypted.  
`sqlite> select * from cifstab;`

```
films|gAAAAABl0KUkN57RMohIrenfwERNZYzEWqDZSGC50o-vwp1rcNkZMDk_5aD1XSpRJQplC3JmGBLJn9DqKwOHqAIVqUZscMAdpA==|gAAAAABl0KUlINxD3l2xtTCclaHz7YqWTh-GeyyYRhmWNeKVI6C3CoAQb4nEd_bKGSAb2tfevFuZnv6snkcTLYuf3I4GT30LSw==|gAAAAABl0KUlXF42gwBXOq3lAe7McKPrp9eWgY4ecCFP9Qmh0rIef42_vckACXvLtu-Fwi5ApO1Kb9waLTrSUNzLdXGgoBZoXQ==|gAAAAABl0KUlLg927C7PlSteAfpG0q2BNRpcFshsbWfSWxqH5JB9tIK2g0Mduh-ckStYOP0OiCV2xByvKufzboTrUjy61rb1wg==|gAAAAABl0KUlWBGAzK4NjGdOOUjKGZA3kE09AG0Pjsadi-5URVsQMBh5IRFMU-zjTuTlg5vrAjNLoMQIbt-QmOw-BCihM3taWQ==|gAAAAABl0KUlkuWEBToAaeig1vgFSFi5dIjQnlT5e4JkwVES8hoMtQvf_6UK3MRUm0vdEhXuRmoPvRqE0SYyPXtqiN1W2z07MRPHdHO27TbbcAvnE8SNQFTq9uSRR452cUQMje6CQ643
```

### Mount cifs shares at boot time through systemd
Cifscloak can generate a simple systemd file that seems to work fine for me on Ubuntu and Centos 8.  
Initially I did not write in any retry mechanism because it just felt sloppy but after systemd gave me a bit of a ride ( through my lack of understanding ) and after reading the documentation ( which suggested that causing the boot to wait is bad ), I instead wrote in a retry. Retrying 6 times seems to get the mounts sorted during boot, -r 6 is added to the systemd file by default. 

* Mountpoint directories are automatically created with default permissions.

## Help
### cifscloak -h
usage: cifscloak [-h] {addmount,mount,removemounts,listmounts} ...  
  
cifscloak - command line utility for mounting cifs shares using encrypted passwords  
  
positional arguments:  
  {addmount,mount,removemounts,listmounts}  
                        Subcommands  
    addmount            Add a cifs mount to cifstab, addmount -h for help  
    mount               Mount cifs shares, mount -h for help  
    removemounts        Remove cifs mounts from cifstab. removemount -h for help  
    listmounts          Display cifstab shares  
  
optional arguments:  
  -h, --help            show this help message and exit  
  
### cifscloak addmount -h

usage: cifscloak addmount [-h] -n NAME -s SHARENAME -i IPADDRESS -m MOUNTPOINT -u USER [-o OPTIONS]  
  
optional arguments:  
  -h, --help            show this help message and exit  
  -n NAME, --name NAME  Connection name e.g identifying server name  
  -s SHARENAME, --sharename SHARENAME  
                        Share name  
  -i IPADDRESS, --ipaddress IPADDRESS  
                        Server address or ipaddress  
  -m MOUNTPOINT, --mountpoint MOUNTPOINT  
                        Mount point  
  -u USER, --user USER  User name  
  -p PASSWORD, --password PASSWORD  
                        Allows a password to be specified on the command line, otherwise getpass is used and password entry is hidden  
  -o OPTIONS, --options OPTIONS  
                        Quoted csv options e.g. "domain=mydomain,ro"   
 
## cifscloak removemounts -h
  
usage: cifscloak removemounts [-h] -n NAMES [NAMES ...]  
  
optional arguments:  
  -h, --help            show this help message and exit  
  -n NAMES [NAMES ...], --names NAMES [NAMES ...]  
                        Remove cifs mounts e.g. -a films music  
  
## cifscloak mount -h

usage: cifscloak mount [-h] [-u] [-r RETRIES] [-w WAITSECS] (-n NAMES [NAMES ...] | -a)  
  
optional arguments:  
  -h, --help            show this help message and exit  
  -u                    Unmount the named cifs shares, e.g -a films music  
  -r RETRIES, --retries RETRIES  
                        Optional ( default: 3 ) - Retry count, useful when systemd is in play  
  -w WAITSECS, --waitsecs WAITSECS  
                        Optional ( default: 5 seconds ) - Wait time in seconds between retries  
  -n NAMES [NAMES ...], --names NAMES [NAMES ...]  
                        Mount reference names, e.g -n films music. --names and --all are mutually exclusive  
  -a, --all             Mount everything in the cifstab.  
  
## cifscloak systemdfile -h

usage: cifscloak systemdfile [-h] (-n NAMES [NAMES ...] | -a)  
  
optional arguments:  
  -h, --help            show this help message and exit  
  -n NAMES [NAMES ...], --names NAMES [NAMES ...]  
                        Add named shares to the systemd unit file  
  -a, --all             Add all cifstab shares to the systemd unit file  
  
### Shout out

Thank you to mgazzin ( https://github.com/mgazzin ) for suggesting CIFSCLOAK_HOME, implemented in v1.0.27  

Thank you to thesnipiid ( https://github.com/thesnipiid ) for improving reliability by suggesting quotes around the password field, implemented in v1.0.27  