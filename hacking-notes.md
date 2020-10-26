#- << LINUX >> -#
	##-Shell Stabilisation-##
	In order to ensure that we dont accidentally lose our shell by using CTRL+C to end a process, we stabilise it.
	Stabilising also gives us the ability to run sudo, and use things like nano (as we need a pty for that).
		
		###-PYTHON###
			'which python' - find which version of python is on the system.
			'python -c 'import pty;pty.spawn("/bin/bash")''
			
			< BACKGROUND NC SESSION WITH CTRL+Z >
			'stty raw -echo' - means CTRL+C doesnt close our shell, and also gives us autocomplete.
			'fg' - foregrounds the session, getting our remote shell back.
			< HIT ENTER A FEW TIMES >
			'reset'
			'export TERM=xterm-256color' - sets the terminal to deal with things - if 256 colour doesnt work then just use 'xterm'.
			
	
		###-'/usr/bin/script -qc /bin/bash /dev/null'###
		is another alternative to spawning a pty with Python.
		We then need to background the session and use our 'stty raw -echo' etc.
		
		Should you wish to have the number of columns and rows displaying to you match your terminal, run 'stty size' on your local machine.
		The first number is the number of rows, the second the number of columns. Now, on the remote shell, run:
		'stty rows <row-num> columns <column-num>'.
		
		
	##-Enumeration-##
		Probably the first command you want to run is 'id'. This will tell you your username, and what groups you belong to.
		This is useful information to know for later enumeration.
		
		###-Find Command-###
			Once we know our username, user id, and group names and ids, we can begin enumeration with the find command. 
			
			####-Files we own:####
				'find / -user <our username> 2>/dev/null'
			
			####-Files belonging to our group:####
				'find / -group <group-name> 2>/dev/null'
			
			####-Files that we can write to:####
				'find / -type f -writable 2>/dev/null'
				We can replace '-writable' with '-readable' also, as well as changing '-type f' to '-type d' to search for directories. 
			
			####-Files of a particular extension:####
				'find / -type f -name *.<ext> 2>/dev/null'
			
			We can run the find command with '-exec ls -la {} \;' to run a command upon every result found (the result is filled into the {}).
				
			It is worth noting that we can pipe (|) into 'grep -v '<str1>\|<str2>...' to filter out results we do not want.
			
			A useful thing to do is redirect the results to a file (/tmp is typically a word writeable directory), and then go through.


		###-Useful Files and Directories to Check-###
			####-DIRECTORIES:####
				/home/* - all users home directories
				/var/backups - typical back up folder
				/tmp - temporary storage for system processes
				/var/tmp - temporary storage for system processes
				/opt - additional packages/software
				/var/mail - mail directory
				
			####-FILES:####
				/etc/passwd - cat and pipe into 'grep /bin' to find names of all users on the box that have shells. 
				If writeable we can duplicate the root users line and just generate a password hash with:
				'openssl passwd -1 -salt <salt> <password>', and place it in the position of the password. 
				
				/etc/sudoers - we may be able to read sudo permissions of users without using sudo -l, hoping for a NOPASSWD entry.
				If this file is writeable, we can append this line to give ourselves sudo access to all commands to privesc with:
				'echo "<username> ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers'
				
				/home/*/.ssh/id_rsa - this is a users ssh private key and if we can access it, we can copy to our system and ssh in as that user.
				
				/home/*/.ssh/authorized_keys - if this file is writeable, we can add our own public key to it.
				
				If we can make a directory of .ssh and then create an authorized_key file for the user.
			

		###-Suid Files-###
			'find / -perm -4000 2>/dev/null'
			Check on https://gtfobins.github.io/ for anything useful. 
			Sometimes the results will contain something cutsom which you can go on to enumerate, reverse engineer, and exploit.
			
		###-Sudo Permissions-###
			'sudo -l' - will list which files you have
			Check on https://gtfobins.github.io/ for anything useful. 
			Sometimes the results will contain something cutsom which you can go on to enumerate, reverse engineer, and exploit.
			
		###-Capabilities-###
			'getcap -r / 2>/dev/null' - lists all capabilities that applications have.
			Check on https://gtfobins.github.io/ for anything useful. 
			
		###-PATH Variable exploitation-###

		###-Python Import Exploitation-###
		
		###-Cron Jobs-###
		
		###-Running Proccesses-###
		
		###-Docker-###
		
		###-LXD Group Privesc-###
		
		###-KERNEL EXPLOITS-###
	
		###-SUDO EXPLOITS-###
	
		
		
	##-Transferring Files On and Off Linux Machines-##
		###-Python-###
		
		###-SCP-###
		
		###-Netcat-###
		
		
#- << WINDOWS >> -#
	##-Enumeration-##
	
	##-Impacket-##
	
	##-Evil WinRM-##
	
	##-Transferring Files On and Off Windows Machines-##
		###-Powershell-###
		
#- << REVERSE SHELLS >> -#
	##-BASH-##
	
	##-NETCAT-##
	
	##-PHP-##
	
	##-PYTHON-##
	
	##-POWERSHELL-##
	
	##-NODE JS-##
	

#- << NETWORK ENUMERATION >> -#
	##-FTP-##
	
	##-SSH-##
	
	##-SMB-##
	
	##-NFS-##
	
	##-Email-##
	

#- << WEB ENUMERATION + EXPLOITATION >> -#
	##-Directory and File Enumeration-##
	
	##-Subdomain Enumeration-##
	
	##-XSS-##
	
	##-SQL Injection-##
		###-Basic Enumeration-###
		
		###-Database Enumeration-###
			####-Database Names-####
			
			####-Table Names-####
			
			####-Column Names-####
			
		###-Dump Data-###
		
		###-SQLMAP-###
	
	##-LFI + RFI-##
	
	##-File Upload Vulnerabilities-##
		###-Double Extension Bypass-###
		
		###-Change Extension in Burp Suite-###
		
		###-Intercept Request of Client Side Validation Script-###
		
		###-MIME Types-###
		
		###-Magic Bytes-###
		
		###-Finding Where The File Was Uploaded To-###
		
	##-WORDPRESS-##
		###-WPSCAN-###
		
		###-Shell Upload-###
			####-Theme-####
			
			####-Plugins-####
			
			####-Editing Current Pages-####
			
		###-URE Exploit-###
		
	##-Brute Forcing With HYDRA-##
	
	
#- << STEGANOGRAPHY >> -#
	##-Binwalk-##
	
	##-ExifTool-##
	
	##-StegHide-##
	
	##-StegCrack-##
	
	##-Audio Wave Forms-##
	
	
#- << BINARY EXPLOITATION >> -#


#- << SEARCHING FOR EXPLOITS >> -#
	##-SearchSploit-##
	
	##-Exploit DB-##