# Project 10 - Honeypot

Time spent: 20 hours spent in total

> Objective: Setup a honeypot and provide a working demonstration of its features.

### Required: Overview & Setup

- [X] A basic writeup (250-500 words) on the `README.md` desribing the overall approach, resources/tools used, findings
- [X] A specific, reproducible honeypot setup, ideally automated. There are several possibilities for this:
	- A Vagrantfile or Dockerfile which provisions the honeypot as a VM or container
	- A bash script that installs and configures the honeypot for a specific OS
	- Alternatively, **detailed** notes added to the `README.md` regarding the setup, requirements, features, etc.

### Setup
* ON YOUR HOST MACHINE
Start by cloning the MHN source and launching via Vagrant:
```
$ git clone https://github.com/threatstream/mhn.git
$ cd mhn
$ vagrant up
```
This should create two VMs:
honeypot - the honeypot VM, aka the target
server - the VM running the admin console for monitoring data collected by the honeypot

Verify the two VMs are up and running:
```
$ vagrant global-status
id       name     provider   state    directory
--------------------------------------------------------------------------------------------------------------------------
924087e  server   virtualbox running  /Users/user/week9/mhn
e91f356  honeypot virtualbox running  /Users/user/week9/mhn
Confirm the IP addresses of each. The values below are statically assigned in the Vagrant file, so your output should match the below:
 ```

Confirm the IP addresses of each. The values are statically assigned in the Vagrant file, so your output should match the below:
```
$ vagrant ssh server -c "hostname -I | cut -d' ' -f2" 2>/dev/null
10.254.254.100
$ vagrant ssh honeypot -c "hostname -I | cut -d' ' -f2" 2>/dev/null
10.254.254.101
```

Open shell on server:
```
$ vagrant ssh server
```

* ON YOUR HOST MACHINE
After executing the vagrant ssh server command above, the command prompt will change to vagrant@mhn-server:~, indicating you're now inside the server VM. Start by opening a superuser shell and installing git:
```
$ sudo su -
$ apt-get install -y git
```

At this point we need to clone the mhn repo again inside the server vm. Due to an outstanding issue we will need to use a patched fork instead of the primary repo:
```
$ cd /opt/
$ git clone https://github.com/0x7fff9/mhn.git
# Cloning into 'mhn'...
# ...
$ cd mhn
# pwd should now be /opt/mhn
$ git checkout 0x7fff9-encoding_patch
Branch 0x7fff9-encoding_patch set up to track remote branch 0x7fff9-encoding_patch from origin.
Switched to a new branch '0x7fff9-encoding_patch'
```

Now, from /opt/mhn, launch the pre-requisite install scripts:
```
$ cd scripts
# pwd should now be /opt/mhn/scripts
$ ./install_hpfeeds.sh ; ./install_mnemosyne.sh ; ./install_honeymap.sh
```

This will take ~10 mins. After this, check the supervisor process to make it all components were installed
```
$ supervisorctl status
geoloc                           RUNNING    pid 29334, uptime 0:02:22
honeymap                         RUNNING    pid 29335, uptime 0:02:22
hpfeeds-broker                   RUNNING    pid 10253, uptime 0:07:33
mnemosyne                        RUNNING    pid 28222, uptime 0:06:09
```

Important: Since this is a private network deployment, you need to change a mnemosyne config option to support this. In the server vm, edit /opt/mnemosyne/mnemosyne.cfg and change ignore_rfc1918 to False:
```
[normalizer]
ignore_rfc1918 = False
```

Now restart mnemosyne and run the final server install script:
```
$ supervisorctl restart mnemosyne
# pwd should still be /opt/mhn/scripts
$ ./install_mhnserver.sh
```

The script will prompt you for values (choose whatever you like for Superuser password, otherwise blank indicates default):
```
Do you wish to run in Debug mode?: y/n n
Superuser email: YOUR-EMAIL@YOUR-SITE.com
Superuser password: 
Superuser password: (again): 
Server base url ["http://1.2.3.5"]: http://10.254.254.100
Honeymap url ["http://1.2.3.5:3000"]: http://10.254.254.100:3000
Mail server address ["localhost"]: 
Mail server port [25]: 
Use TLS for email?: y/n y
Use SSL for email?: y/n y
Mail server username [""]:  
Mail server password [""]: 
Mail default sender [""]: 
Path for log file ["/var/log/mhn/mhn.log"]: 
```

The script will complete in about ~10 mins. When it completes, exit the superuser shell and the server VM by running the exit command twice to return to your host machine:
```
root@mhn-server:/opt/mhn/scripts# exit
# logout
vagrant@mhn-server:~$ exit
# logout
# Connection to 127.0.0.1 closed.
```

* ON YOUR HOST MACHINE
Access the MHN admin console via a browser at http://10.254.254.100 and login with the email / password you specified in the previous step.
Click around to familiarize yourself with the interface.
Go to the Deploy page and in the Select Script input, select Ubuntu - Dionaea and copy the Deploy Command snippet to your clipboard. Dionaea is a low-interaction honeypot designed to trap malware (though we'll just look at using it to detect port scanning).

Back in the terminal, from your host machine, open a shell into the honeypot VM:
```
$ vagrant ssh honeypot
```

The prompt should change again.

* INSIDE THE HONEYPOT VM
Open a superuser shell and execute the deploy command you copied from the browser page:
```
$ sudo su -
# command copied from "Deploy Command"
$ wget "http://10.254.254.100/api/script/?text=true&script_id=2" -O deploy.sh && sudo bash deploy.sh http://10.254.254.100 DuGSk2KT
```

The script will run for a few minutes and will eventually prompt you twice with File to Patch: to which you should supply the following values, respectively:
	/etc/dionaea/dionaea.conf
	/usr/lib/dionaea/python/dionaea/ihandlers.py
```
The text leading up to this was:
--------------------------
|--- /etc/dionaea/dionaea.conf
|+++ /etc/dionaea/dionaea.conf.new
--------------------------
File to patch: /etc/dionaea/dionaea.conf

...

The text leading up to this was:
--------------------------
|
|--- /usr/lib/dionaea/python/dionaea/ihandlers.py
|+++ /usr/lib/dionaea/python/dionaea/ihandlers.py.new
--------------------------
File to patch: /usr/lib/dionaea/python/dionaea/ihandlers.py
```

When it completes, exit the superuser shell and the honeypot VM by running the exit command twice to return to your host machine:
```
root@mhn-honeypot:~# exit
logout
vagrant@mhn-honeypot:~$ exit
logout
Connection to 127.0.0.1 closed.
```

* ON YOUR HOST MACHINE
Now, in the browser, go to the Sensors page and you should see mhn-honeypot-dionaea listed.
Check out the Attacks page next; you should see no data...yet.

* ATTACK
Let's try a simple attack against the honeypot VM using nmap to see how it is detected.
You could stage the attack from your host machine or your Kali container or VM, which may already have nmap installed (in which case, skip the setup step).

* SETUP
For simplicity's sake, we'll attack from the server VM (if you're running Windows, this may be the easiest way to get nmap installed). Starting from the host, open a Vagrant shell into the server VM and install nmap via apt:
```
$ vagrant ssh server
vagrant@mhn-server:~$ sudo apt-get install -y nmap
Attack with nmap
```
* ATTACK WITH NMAP
Run a scan against the honeypot VM using nmap:
```
vagrant@mhn-server:~$ nmap -sV -P0 10.254.254.101

Starting Nmap 5.21 ( http://nmap.org ) at 2017-04-08 18:55 UTC
```
Leave nmap running and go back to the Attacks page in the console. You should see evidence of the nmap


### Required: Demonstration

- [X] A basic writeup of the attack (what offensive tools were used, what specifically was detected by the honeypot)
- [X] An example of the data captured by the honeypot (example: IDS logs including IP, request paths, alerts triggered)
- [X] A screen-cap of the attack being conducted

Unfortunately, I was unable to configure this assignment correctly, due to issues with the Virtual Machine setup. After trying several different methods, I have decided to demonstrate my knowledge of what this specific honeypot is supposed to do as an alternative.  
<img src='error.png' title='error' width='' alt='error' />
<img src='honeypot.gif' title='honeypot' width='' alt='honeypot' />
    
### Optional: Features
- Honeypot
	- [ ] HTTPS enabled (self-signed SSL cert)
	- [ ] A web application with both authenticated and unauthenticated footprint
	- [ ] Database back-end
	- [ ] Custom exploits (example: intentionally added SQLI vulnerabilities)
	- [ ] Custom traps (example: modified version of known vulnerability to prevent full exploitation)
	- [ ] Custom IDS alert (example: email sent when footprinting detected)
	- [ ] Custom incident response (example: IDS alert triggers added firewall rule to block an IP)
- Demonstration
	- [ ] Additional attack demos/writeups
	- [ ] Captured malicious payload
	- [ ] Enhanced logging of exploit post-exploit activity (example: attacker-initiated commands captured and logged)
