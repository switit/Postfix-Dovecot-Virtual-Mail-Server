# Postfix-Dovecot-Virtual-Mail-Server
Steps to create a mail server based on postfix, dovecot, mariadb and others addons.

### Prerequisites

1. Steps are based on a server running Arch Linux but shoud work with other distros
2. A domain name to be used as the mail server registered with DNS pointing to mail server IP address. DNS records must include A and MX records
3. Letsencrypt certificate for the mail domain in 2.
4. hostname is set to <i>mail.example.com</i> and /etc/hosts adjusted to reflect the hostname.

### Steps

1. Log in to the server as root or change to root with sudo su
2. Create system group and user for mail server
```
    groupadd -g 5000 vmail
    useradd -u 5000 -g vmail -s /usr/bin/nologin -d /home/vmail -m vmail
```
3. Install and configure mariadb
```
pacman -S mariadb
mariadb-install-db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
systemctl start mariadb
systemctl enable mariadb
mysql_secure_installation
mysql -u root -p
# In mariadb promp:
CREATE DATABASE postfix_db;
GRANT ALL ON postfix_db.* TO 'postfix_user'@'localhost' IDENTIFIED BY 'YOURPASSWORD';
FLUSH PRIVILEGES;
quit;
```
4. Install postfix
```
pacman -S postfix
systemctl start postfix
systemctl enable postfix
```
5. 

