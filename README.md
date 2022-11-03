# Postfix-Dovecot-Virtual-User-Mail-Server
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
4. Install and configure postfix
```
pacman -S postfix
systemctl start postfix
systemctl enable postfix
```
Replace /etc/postfix/main.cf content with
```
mydomain = example.com
biff = no
myhostname = mail.example.com
mydestination = $myhostname, localhost.$mydomain, localhost
myorigin = $myhostname
disable_vrfy_command = yes
matted email addresses - prevents a lot of spam
strict_rfc821_envelopes = yes
show_user_unknown_table_name = no
message_size_limit = 51200000
mailbox_size_limit = 51200000
allow_percent_hack = no
swap_bangpath = no
recipient_delimiter = +
smtpd_tls_cert_file = /etc/letsencrypt/live/mail.example.com/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/mail.example.com/privkey.pem
smtp_tls_security_level = dane
smtp_dns_support_level = dnssec
smtp_host_lookup=dns
tls_ssl_options = no_ticket, no_compression
smtpd_tls_dh1024_param_file = ${config_directory}/dh2048.pem
smtpd_tls_dh512_param_file = ${config_directory}/dh512.pem
smtpd_sasl_auth_enable = yes
smtpd_sasl_path = /var/run/dovecot/auth-client
smtpd_sasl_type = dovecot
smtpd_sasl_authenticated_header = yes
smtpd_tls_auth_only = yes
smtpd_sasl_tls_security_options = noanonymous
smtpd_tls_received_header = yes
smtpd_helo_required = yes

smtpd_client_restrictions =
  permit_mynetworks,
  permit_sasl_authenticated,
  reject_unknown_reverse_client_hostname,
  reject_unauth_pipelining
smtpd_helo_restrictions =
  permit_mynetworks,
  permit_sasl_authenticated,
  reject_invalid_helo_hostname,
  reject_non_fqdn_helo_hostname,
  reject_unauth_pipelining
smtpd_sender_restrictions =
  permit_mynetworks,
  permit_sasl_authenticated,
  reject_non_fqdn_sender,
  reject_unknown_sender_domain,
  reject_unauth_pipelining,
  #check_sender_access      texthash:/etc/postfix/rejected_tlds
smtpd_reject_unlisted_sender = yes  
smtpd_relay_restrictions = 
  permit_mynetworks, 
  permit_sasl_authenticated, 
# !!! THIS SETTING PREVENTS YOU FROM BEING AN OPEN RELAY !!!
  reject_unauth_destination
# !!!      DO NOT REMOVE IT UNDER ANY CIRCUMSTANCES      !!!
smtpd_recipient_restrictions =
  check_policy_service unix:private/policy-spf,
  check_policy_service unix:private/quota-status,
  permit_mynetworks,
  permit_sasl_authenticated,
  reject_invalid_hostname,
  reject_non_fqdn_sender,
  reject_non_fqdn_recipient,
  reject_unknown_sender_domain,
  reject_unknown_recipient_domain,
  reject_unauth_destination,
  reject_unauth_pipelining,
  reject_unverified_recipient
  reject_rhsbl_helo dbl.spamhaus.org,
  reject_rhsbl_reverse_client dbl.spamhaus.org,
  reject_rhsbl_sender dbl.spamhaus.org
  
smtpd_data_restrictions =
  permit_mynetworks,
  permit_sasl_authenticated,
  reject_multi_recipient_bounce,
  reject_unauth_pipelining

alias_maps = texthash:/etc/postfix/aliases
alias_database = $alias_maps

# DELIVERY TO MAILBOX
home_mailbox = Maildir/

# SHOW SOFTWARE VERSION OR NOT
smtpd_banner = $myhostname ESMTP $mail_name ($mail_version)

# DEBUGGING CONTROL
debug_peer_level = 2
#debug_peer_list = 127.0.0.1
#debug_peer_list = some.domain
debugger_command =
	 PATH=/bin:/usr/bin:/usr/local/bin:/usr/X11R6/bin
	 ddd $daemon_directory/$process_name $process_id & sleep 5

virtual_alias_maps = proxy:mysql:/etc/postfix/virtual_alias_maps.cf
virtual_mailbox_domains = proxy:mysql:/etc/postfix/virtual_domains_maps.cf
virtual_mailbox_maps = proxy:mysql:/etc/postfix/virtual_mailbox_maps.cf
virtual_mailbox_base = /home/vmail
virtual_mailbox_limit = 512000000
virtual_minimum_uid = 5000
virtual_transport = lmtp:unix:private/dovecot-lmtp
virtual_uid_maps = static:5000
virtual_gid_maps = static:5000
local_transport = virtual
local_recipient_maps = $virtual_mailbox_maps
transport_maps = texthash:/etc/postfix/transport
smtpd_use_tls = yes
smtpd_sasl_local_domain = $mydomain
broken_sasl_auth_clients = yes
smtpd_tls_loglevel = 1
  
non_smtpd_milters=unix:/var/run/rspamd/rspamd.sock
smtpd_milters=unix:/var/run/rspamd/rspamd.sock
milter_protocol = 6
milter_mail_macros=i {mail_addr} {client_addr} {client_name} {auth_authen}
milter_default_action = accept
smtpd_recipient_limit = 50
smtpd_recipient_overshoot_limit = 51
smtpd_hard_error_limit = 20
smtpd_client_recipient_rate_limit = 50
smtpd_client_connection_rate_limit = 10
smtpd_client_message_rate_limit = 25
default_extra_recipient_limit = 50
duplicate_filter_limit = 50
default_destination_recipient_limit = 50

delay_warning_time = 4h
policy-spf_time_limit = 3600s

postscreen_access_list = permit_mynetworks cidr:/etc/postfix/postscreen_access.cidr
postscreen_blacklist_action = drop
postscreen_greet_action = enforce
postscreen_dnsbl_threshold = 3
postscreen_dnsbl_action = drop
postscreen_dnsbl_sites =
        zen.spamhaus.org*3
        b.barracudacentral.org=127.0.0.[2..11]*2
        bl.spameatingmonkey.net*2
        bl.spamcop.net
        dnsbl.sorbs.net
       swl.spamhaus.org*-4
```
Replace /etc/postfix/master.cf content with
```
smtp      inet  n       -       n       -       1       postscreen
smtpd     pass  -       -       n       -       -       smtpd
dnsblog   unix  -       -       n       -       0       dnsblog
tlsproxy  unix  -       -       n       -       0       tlsproxy
submission inet n       -       n       -       -       smtpd
  -o smtpd_tls_dh1024_param_file=${config_directory}/dh1024.pem
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
smtps     inet  n       -       n       -       -       smtpd
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
pickup    unix  n       -       n       60      1       pickup
cleanup   unix  n       -       y       -       0       cleanup
header_cleanup unix n   -       -       -       0       cleanup
 -o header_checks=regexp:/etc/postfix/submission_header_cleanup.cf
qmgr      unix  n       -       n       300     1       qmgr
#qmgr     unix  n       -       n       300     1       oqmgr
tlsmgr    unix  -       -       n       1000?   1       tlsmgr
rewrite   unix  -       -       n       -       -       trivial-rewrite
bounce    unix  -       -       n       -       0       bounce
defer     unix  -       -       n       -       0       bounce
trace     unix  -       -       n       -       0       bounce
verify    unix  -       -       n       -       1       verify
flush     unix  n       -       n       1000?   0       flush
proxymap  unix  -       -       n       -       -       proxymap
proxywrite unix -       -       n       -       1       proxymap
smtp      unix  -       -       n       -       -       smtp
relay     unix  -       -       n       -       -       smtp
showq     unix  n       -       n       -       -       showq
error     unix  -       -       n       -       -       error
retry     unix  -       -       n       -       -       error
discard   unix  -       -       n       -       -       discard
local     unix  -       n       n       -       -       local
virtual   unix  -       n       n       -       -       virtual
lmtp      unix  -       -       -       -       -       lmtp
anvil     unix  -       -       n       -       1       anvil
scache    unix  -       -       n       -       1       scache
policy-spf  unix  -       n       n       -       0       spawn
     user=nobody argv=/usr/lib/postfix/postfix-policyd-spf-perl  
```
Creat the following 4 files

/etc/postfix/virtual_alias_maps.cf
```
user = postfix_user
password = YOURPASSWORD
hosts = localhost
dbname = postfix_db
query = SELECT goto FROM alias WHERE address='%s' AND active = '1'
```
/etc/postfix/virtual_domains_maps.cf
```
user = postfix_user
password = YOURPASSWORD
hosts = localhost
dbname = postfix_db
query = SELECT domain FROM domain WHERE domain='%s' AND backupmx = '0' AND active = '1'
```
/etc/postfix/virtual_mailbox_limits.cf
```
user = postfix_user
password = YOURPASSWORD
hosts = localhost
dbname = postfix_db
query = SELECT maildir FROM mailbox WHERE username='%s' AND active = '1'
```
/etc/postfix/virtual_mailbox_maps.cf
```
user = postfix_user
password = YOURPASSWORD
hosts = localhost
dbname = postfix_db
query = SELECT maildir FROM mailbox WHERE username='%s' AND active = '1'
```

5. Install and configure dovecot
```
pacman -S dovecot
```
mkdir /etc/dovecot
cp /usr/share/doc/dovecot/example-config/dovecot.conf /etc/dovecot/dovecot.conf
cp -r /usr/share/doc/dovecot/example-config/conf.d /etc/dovecot
openssl dhparam -out /etc/dovecot/dh.pem 4096 # This takes a long time
ssl_dh = </etc/dovecot/dh.pem
systemctl start dovecot
systemctl enable dovecot
```
6. 

