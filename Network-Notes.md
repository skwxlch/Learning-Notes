# - << NETWORK ENUMERATION >> - #
## FTP - DEFAULT PORT 21: ##
  ```bash
  ftp 0.0.0.0 21 #you will then be asked for the username and password.
  ```
  Test for 'anonymous' access - username and password 'anonymous'.
  This 'anonymous access can be disabled in the /etc/vsftpd.conf file, as well as other conigurations, such as the root folder for the ftp server.
  

## SSH - DEFAULT PORT 22: ##
  ```bash
  ssh user@0.0.0.0 -p22 # standard login will go to port 22, but if ssh is on a non standard port you use the -p to specify the port.
  ```
  If you have an ssh private key, you can find the username it belongs to in the corresponding id_rsa.pub file. Permissions of the id_rsa private key file must be 600. You can then use this to SSH into the machine as that user, with no password. 
  ```bash
  ssh -i id_rsa user@0.0.0.0 -p22
  ```
  Should the private key ask you for a passphrase, (or when you look at it you can see that it is encrypted from it's header), you may have to crack the passphrase. The quickest way to do this would be with ssh2john.py, which is bale to extract the passphrase in the form of a hash, which you can brute force with john. You then enter the passphrase when prompted to do so by the SSH application.
  

## EMAIL - DEFAULT PORTS 25,110 and 145: ##


## SMB - DEFAULT PORT 139 AND 445: ##
```bash 
smbclient -L //0.0.0.0/ #Lists all shares accessible by anyone. Use -U to specify a username.
smbclient //0.0.0.0/sharename #connects to the share so you can look at the files on it.
```
If SMB is open, you can use the (enum4linux.pl)[https://github.com/CiscoCXSecurity/enum4linux] script in order to enumerate users with it's 'RID cycling function'.

## NFS - DEFAULT PORT 2049: ##


## BRUTE FORCING NETWORK SERVICES WITH HYDRA: ##
