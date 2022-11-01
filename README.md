# Postfix-Dovecot-Virtual-Mail-Server
Steps to create a mail server based on postfix, dovecot, mariadb and others addons.
1. Change to root with sudo su
2. Creat system group and user for mail server
<pre><code>
groupadd -g 5000 vmail
useradd -u 5000 -g vmail -s /usr/bin/nologin -d /home/vmail -m vmail
</code></pre>
3. Install and configure mariadb
<pre><code>
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
</code></pre>
4. Install postfix
<pre><code>
pacman -S postfix
systemctl start postfix
systemctl enable postfix
</code></pre>
5. 

