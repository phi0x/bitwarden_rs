# bitwarden_rs
A reference solution for bitwarden_rs

referenced: https://www.linode.com/docs/guides/how-to-self-host-the-bitwarden-rs-password-manager/ tutorial. Credits go to OP, forked tutorial and tweaked a bit.

1. Uninstall any potential docker setup files.
sudo apt-get remove docker docker-engine docker.io containerd runc

2. Install package prerequisites for compatibility with the upstream Docker repository.
apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common

3. Add the official Docker GPG repository key.
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

4. Add the Docker upstream repository.
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

5. update apt-get
sudo apt-get update

6. Install required docker packages
sudo apt-get install docker-ce docker-ce-cli containerd.io

7. Start/enable docker daemon
sudo systemctl enable --now docker

8. confirm docker works
sudo docker ps

9. Pull the bitwarden_rs image.
sudo docker pull bitwardenrs/server:latest

10. Select the desired file system path to store application data. In this guide, the path /srv/bitwarden is used. Create the directory if necessary, and enforce strict permissions for the root user only.
sudo mkdir /srv/bitwarden
sudo chmod go-rwx /srv/bitwarden

11. Create the Docker container for bitwarden_rs with signups disabled and setup admin token along with setting U2F token domain. 
note: Can generate an admin token with: openssl rand -base64 48

docker run -d --name bitwarden -v /srv/bitwarden:/data -e WEBSOCKET_ENABLED=true -e SIGNUPS_ALLOWED=false -e ADMIN_TOKEN="YOUR_TOKEN_HERE" -e DOMAIN=https://YOUR_DOMAIN_HERE -p 127.0.0.1:8080:80 -p 127.0.0.1:3012:3012 --restart on-failure bitwardenrs/server:latest

12. Setup Caddy the reverse proxy that'll serve HTTPS, first pull latest alpine image.
sudo docker pull caddy/caddy:alpine

13. Create Caddy config
nano /etc/Caddyfile

config:

example.com {
  encode gzip

  # The negotiation endpoint is also proxied to Rocket
  reverse_proxy /notifications/hub/negotiate 0.0.0.0:8080

  # Notifications redirected to the websockets server
  reverse_proxy /notifications/hub 0.0.0.0:3012

  # Send all other traffic to the regular bitwarden_rs endpoint
  reverse_proxy 0.0.0.0:8080
}

Note: The site name you choose in this file must match the desired URL that bitwarden_rs is served under. When navigating to the web interface later in this guide, ensure that you type the same hostname chosen in this configuration file (in this example, example.com).

14. Create Caddy folder

sudo mkdir /etc/caddy
sudo chmod go-rwx /etc/caddy

15. Create Caddy docker
sudo docker run -d --name caddy -v /etc/Caddyfile:/etc/caddy/Caddyfile -v /etc/caddy:/root/.local/share/caddy --net host --restart on-failure caddy/caddy:alpine

16. view Caddy logs
sudo docker logs caddy

note: look for any issues such as:
    2020/02/23 05:46:19 [INFO] Unable to deactivate the authorization: <url>
    2020/02/23 05:46:19 [ERROR][example.com] failed to obtain certificate: acme: Error -> One or more domains had a problem:

If any issues, you can stop Caddy/start Caddy with:
sudo docker stop caddy
sudo docker start caddy

17. Test the website
https://YOUR_DOMAIN_HERE

Should see login screen.

18. Configure admin settings
Go to https://YOUR_DOMAIN_HERE/admin
Login with the admin token from step 11.

    18.1 Click on 'Settings', then 'General Settings'
        18.2 Setup the 'Domain URL': https://YOUR_DOMAIN_HERE
        18.3 Invitation organization name: example.com

    18.4 Click on 'SMTP Email Settings' (note I use gmail smtp)
        18.5 'Enabled' check.
        18.6 'Host': smtp.gmail.com
        18.7 'Enable Secure SMTP' check.
        18.8 'Port': 587
        18.9 'From Address': YOUR_EMAIL_HERE@gmail.com
        18.10 'From Name': WHATEVER_NAME_YOU_WANT_HERE
        18.11 'Username': YOUR_EMAIL_HERE@gmail.com
        18.12 'Password': APP_PASSWORD_HERE Note: generated App password for gmail. https://myaccount.google.com/apppasswords

    18.13 Click on 'Email 2FA Settings'
        18.14 'Enabled' check.

    18.15 click 'save'.
    18.16 Test SMTP via the 'SMTP Email Settings', 'Test SMTP': enter your email you'd like to test sending to and see if it works.

19. Setup backups, install sqlite3
sudo apt-get install sqlite3

20. Create a directory for backups.
sudo mkdir /srv/backup
sudo chmod go-rwx /srv/backup

21. Create backup service
nano /etc/systemd/system/bitwarden-backup.service

config: 
[Unit]
Description=backup the bitwarden sqlite database

[Service]
Type=oneshot
WorkingDirectory=/srv/backup
#Take backup
ExecStart=/usr/bin/env sh -c 'sqlite3 /srv/bitwarden/db.sqlite3 ".backup backup-$(date -Is | tr : _).sq3"'
#backup file retention 30 days.
ExecStart=/usr/bin/find . -type f -mtime +30 -name 'backup*' -delete
#Sync local backups offsite - remove this line below if you do not want to do offsite backups via SSH/SCP.
ExecStart=/usr/bin/env sh -c 'sshpass -p "YOUR_SERVER_PASS_HERE" scp -P 22022 -r /srv/backup/ USERNAME_HERE@YOUR_SERVER_HERE:BACKUP_PATH_LOCATION_HERE'

22. start backup service
sudo systemctl start bitwarden-backup.service

23. Verify that a backup file is present:
sudo ls -l /srv/backup/

Should see a file like example: -rw-r--r-- 1 root root 139264 Feb 24 18:16 backup-2020-02-24T18_16_50-07_00.sq3

24. To schedule regular backups using this backup service unit, create the following systemd timer unit.
nano /etc/systemd/system/bitwarden-backup.timer

config:
[Unit]
Description=schedule bitwarden backups

[Timer]
OnCalendar=04:00
Persistent=true

[Install]
WantedBy=multi-user.target

25. Start and enable this timer unit.
sudo systemctl enable bitwarden-backup.timer
sudo systemctl start bitwarden-backup.timer

Should see output similar to:
  bitwarden-backup.timer - schedule bitwarden backups
  Loaded: loaded (/etc/systemd/system/bitwarden-backup.timer; enabled; vendor preset: enabled)
  Active: active (waiting) since Mon 2020-02-24 22:09:44 MST; 7s ago
  Trigger: Tue 2020-02-25 04:00:00 MST; 5h 50min left

26. Setup SSH port to 22022
    26.1 edit ssh file
    nano /etc/sshd/sshd_config

    locate the '#Port 22' and update it to 'Port 22022'
    26.2 restart sshd
    service sshd restart
    26.3 reconnect to your server via port 22022 to test that it works.

27. Setup IP tables
    27.1 Install IPtables packages
    apt-get install iptables iptables-persistent -y
    27.2 Run iptables commands
    iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
    iptables -A INPUT -p udp -m multiport --dports 53 -j ACCEPT
    iptables -A INPUT -p tcp -m multiport --dports 22022,80,443,8080 -j ACCEPT
    iptables -P INPUT DROP
    iptables-save > /etc/iptables/rules.v4


28. reboot server to see if everything comes up OK still by itself.

29. validate iptables are loaded
iptables -L -v -n

Look for your rules you inserted in step 27.2.
example INPUT chain:
Chain INPUT (policy DROP 1181 packets, 65075 bytes)
 pkts bytes target     prot opt in     out     source               destination
 135K   68M ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
   72  4921 ACCEPT     udp  --  *      *       0.0.0.0/0            0.0.0.0/0            multiport dports 53
10394  617K ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            multiport dports 22022,80,443,8080
