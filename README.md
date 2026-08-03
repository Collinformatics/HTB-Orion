# Hack The Box Labs: Orion

This lab exploits Craft CMS version 5.6.16, the CSV-2025-32432 has a Base Score of 10.0.

https://nvd.nist.gov/vuln/detail/CVE-2025-32432

https://github.com/c0gnit00/CVE-2025-32432



# Recon:

Lets start with an nmap scan:

	nmap 10.129.77.211 -sV -sC 

	Nmap scan report for orion.htb (10.129.77.211)
	Host is up (0.11s latency).
	Not shown: 998 closed tcp ports (reset)
	PORT   STATE SERVICE VERSION
	22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
	| ssh-hostkey: 
	|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
	|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
	80/tcp open  http    nginx 1.18.0 (Ubuntu)
	|_http-server-header: nginx/1.18.0 (Ubuntu)
	|_http-title: Orion Telecom
	Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel


Since weve got a website lets add the ip to the hosts file:

	echo '10.129.77.211 orion.htb' | sudo tee -a /etc/hosts

After clicking through the website we see that its very light, theres a form at the bottom that fails to send a request due to a missing API, and theres no links to other webpages.

This page doesnt seem to provide us with any attack surfaces, so enumerate the website and see what we can find:

- NOTE: Be careful when fuzzing, if you overload the server you'll get a "No space left on device" error. If this happens you'll need to reset the machine.

		ffuf -ic -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://orion.htb/FUZZ -e .php,.sh,.jpg,.jpeg,.png,.html,.txt,.bak,.js -mc all -fc 404,403


- From from this we've found a few other pages, lets start by going to http://orion.htb/admin

Notice that at the bottom there is a version for a Content Management System (CMS), Craft CMS 5.6.16.



# Exploit:

If we search for Craft CMS 5.6.16 on the web, we can find CVE-2025-32432

Note:

- The exploit involves writing to a log file, some CVEs use a path that will not work for this machine.

- The path /var/lib/php/sessions/sess_<sessionID> was found to work in this instance.

A exploit script can be found at: https://github.com/c0gnit00/CVE-2025-32432








# Privilege Escalation:






============================================================================



