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
pacman -S postfix postfix-mysql
systemctl start postfix
systemctl enable postfix
```
Install postfix-policyd-spf-perl from Aur

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

## Comment out this part until after completing rspamd installation and configuration ##  
non_smtpd_milters=unix:/var/run/rspamd/rspamd.sock 
smtpd_milters=unix:/var/run/rspamd/rspamd.sock
milter_protocol = 6
milter_mail_macros=i {mail_addr} {client_addr} {client_name} {auth_authen}
milter_default_action = accept
##

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
Install and configure postfixadmin with web server of your choice. Add DNS A record of your postfixadmin site. then launch YOURPOSTFIXADMINURL.setup.php in a browser. Your postfixadmin config file /etc/webapps/postfixadmin/config.local.php shoul include
```
<?php
$CONF['configured'] = true;
$CONF['setup_password'] = 'GENERATED DURING SETUP';
$CONF['database_type'] = 'mysqli';
$CONF['database_host'] = 'localhost';
$CONF['database_user'] = 'postfix_user';
$CONF['database_password'] = 'YOURPASSWORD';
$CONF['database_name'] = 'postfix_db';
$CONF['admin_email'] = 'A VALID EMAIL ADDRESS';
$CONF['domain_in_mailbox'] = 'YES';

// Default Domain Values
// Specify your default values below. Quota in MB.$CONF['aliases'] = '100';
$CONF['mailboxes'] = '100';
$CONF['maxquota'] = '100';

// Quota
// When you want to enforce quota for your mailbox users set this to 'YES'.
$CONF['quota'] = 'YES';
// If you want to enforce domain-level quotas set this to 'YES'.
$CONF['domain_quota'] = 'YES';
// You can either use '1024000' or '1048576'
$CONF['quota_multiplier'] = '1024000';
$CONF['used_quotas'] = 'YES';
```
Database postfix_db will generated by postfixadmin setup process

5. Install and configure dovecot with siev and quota plugins
```
pacman -S dovecot pigeonhole
mkdir /etc/dovecot
cp /usr/share/doc/dovecot/example-config/dovecot.conf /etc/dovecot/dovecot.conf
cp -r /usr/share/doc/dovecot/example-config/conf.d /etc/dovecot
openssl dhparam -out /etc/dovecot/dh.pem 4096 # This takes a long time
```
Replace content of /etc/dovecot/conf.d/10-ssl.conf with
```
ssl_dh = </etc/dovecot/dh.pem
ssl_cert = </etc/letsencrypt/live/mail.example.com/fullchain.pem #Assuming you have letsencrypt certificate in place
ssl_key = </etc/letsencrypt/live/mailexample.com/privkey.pem #Assuming you have letsencrypt certificate in place
ssl_min_protocol = TLSv1
```
Replace content of /etc/dovecot/dovecot.conf with
```
protocols = imap pop3 lmtp sieve
dict {
  quotadict = mysql:/etc/dovecot/dovecot-dict-sql.conf.ext
}
!include conf.d/*.conf
!include_try local.conf
!include_try /usr/share/dovecot/protocols.d/*.protocol
```
Create the following files
/etc/dovecot/dovecot-dict-sql.conf.ext
```
connect = host=localhost dbname=postfix_db user=postfix_user password=127posx
map {
pattern = priv/quota/storage
table = quota2
username_field = username
value_field = bytes
}
map {
pattern = priv/quota/messages
table = quota2
username_field = username
value_field = messages
}
```
/etc/dovecot/dovecot-sql.conf
```
driver = mysql
connect = host=localhost dbname=postfix_db user=postfix_user password=127posx
default_pass_scheme = MD5-CRYPT
user_query = SELECT '/home/vmail/%d/%u' as home, 'maildir:/home/vmail/%d/%u' as mail, 5000 AS uid, 5000 AS gid, concat('*:bytes=', quota) AS quota_rule FROM mailbox WHERE username = '%u' AND active = '1'
password_query = SELECT username as user, password, '/home/vmail/%d/%u' as userdb_home, 'maildir:/home/vmail/%d/%u' as userdb_mail, 5000 as  userdb_uid, 5000 as userdb_gid, concat('*:bytes=', quota) AS userdb_quota_rule FROM mailbox WHERE username = '%u' AND active = '1'
iterate_query = SELECT username AS user FROM mailbox
```
Replace content of the following files. Creat new files if not exist
/etc/dovecot/conf.d/10-auth.conf
```
auth_mechanisms = plain login
passdb {
    driver = sql
    args = /etc/dovecot/dovecot-sql.conf
}
userdb {
    driver = sql
    args = /etc/dovecot/dovecot-sql.conf
}

service auth {
    unix_listener auth-client {
        group = postfix
        mode = 0660
        user = postfix
    }
    user = root
}
```
/etc/dovecot/conf.d/10-dovecot-postfix.conf
```
mail_home = /home/vmail/%d/%u
mail_location = maildir:~
```
/etc/dovecot/conf.d/10-master.conf
```
service lmtp {
  unix_listener lmtp {
    #mode = 0666
  }
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = postfix
    group = postfix
  }
}

service dict {
	unix_listener dict {
		group = vmail
		mode = 0660
		user = vmail
	}
	user = root
}

service stats {
    unix_listener stats-reader {
        user = vmail
        group = vmail
        mode = 0660
    }

    unix_listener stats-writer {
        user = vmail
        group = vmail
        mode = 0660
    }
}
```
/etc/dovecot/conf.d/15-mailboxes.conf
```
namespace inbox {
  inbox = yes
  mailbox Trash {
    auto = no
    autoexpunge = 30d
    special_use = \Trash
  }
  mailbox Drafts {
    auto = no
    special_use = \Drafts
  }
  mailbox Sent {
    auto = subscribe # autocreate and autosubscribe the Sent mailbox
    special_use = \Sent
  }
  mailbox "Sent Messages" {
    auto = no
    special_use = \Sent
  }
  mailbox Junk {
    auto = create # autocreate Spam, but don't autosubscribe
    autoexpunge = 30d
    special_use = \Junk
  }
}
mailbox_list_index = yes
```
/etc/dovecot/conf.d/20-managesieve.conf
```
service managesieve-login {
}

service managesieve {
}

protocol sieve {
    managesieve_max_line_length = 65536
    managesieve_implementation_string = dovecot
}

plugin {
    sieve = file:~/sieve
    active=~/.dovecot.sieve
}
```
/etc/dovecot/conf.d/20-protocols.conf
```
mail_plugins=quota
protocol pop3 {
mail_plugins = quota
pop3_client_workarounds = outlook-no-nuls oe-ns-eoh
pop3_uidl_format = %08Xu%08Xv
}

protocol lmtp {
mail_plugins = quota sieve
postmaster_address = postmaster@wisecomnet.com
}	

protocol lda {
mail_plugins = quota
postmaster_address = postmaster@wisecomnet.com
}

protocol imap {
mail_plugins = $mail_plugins imap_quota imap_sieve
mail_plugin_dir = /usr/lib/dovecot/modules
}
```
/etc/dovecot/conf.d/90-quota.conf
```
plugin {
quota = dict:User quota::proxy::quotadict
#quota_set = dict:proxy::quota
#quota2 = dict:user::proxy::quotadict
#quota_exceeded_message = Quota exceeded
#quota_rule = *:storage=1GB:messages=10000
quota_rule2 = Trash:storage=+10%%
quota_rule3 = SPAM:ignore
#quota2 = maildir
quota_warning = storage=100%% quota-warning +100 %u
quota_warning2 = storage=95%% quota-warning +95 %u
quota_warning3 = storage=80%% quota-warning +80 %u
quota_warning4 = -storage=100%% quota-warning -100 %u # user is no longer over quota
  quota_status_success = DUNNO
  quota_status_nouser = DUNNO
  quota_status_overquota = "452 4.2.2 Mailbox is full and cannot receive any more emails"
}

service quota-status {
  executable = /usr/lib/dovecot/quota-status -p postfix
  unix_listener /var/spool/postfix/private/quota-status {
    	group = postfix
		mode = 0600
		user = postfix
  }
}

service quota-warning {
	executable = script /usr/local/bin/quota-warning.sh
	# use some unprivileged user for executing the quota warnings
	user = vmail
	unix_listener quota-warning {
		group = vmail
		mode = 0660
		user = vmail
	}
}	
```
/etc/dovecot/conf.d/91-sieve.conf
```
plugin {
    sieve_before = /etc/dovecot/sieve-before
    sieve_after = /home/vmail/%d/%u/sieve
    sieve_plugins = sieve_imapsieve sieve_extprograms
    # From elsewhere to Junk folder
    imapsieve_mailbox1_name = Junk
    imapsieve_mailbox1_causes = COPY
    imapsieve_mailbox1_before = file:/etc/dovecot/sieve/learn-spam.sieve

    # From Junk folder to elsewhere
    imapsieve_mailbox2_name = *
    imapsieve_mailbox2_from = Junk
    imapsieve_mailbox2_causes = COPY
    imapsieve_mailbox2_before = file:/etc/dovecot/sieve/learn-ham.sieve

    sieve_pipe_bin_dir = /etc/dovecot/sieve
    sieve_global_extensions = +vnd.dovecot.pipe
    }
```
/usr/local/bin/quota-warning.sh
```
#!/bin/sh
BOUNDARY="$1"
USER="$2"
MSG=""
if [ "$BOUNDARY" = "+100" ]; then
    MSG="Your mailbox is now overfull (>100%). In order for your account to continue functioning properly, you need to remove some emails NOW."
elif [ "$BOUNDARY" = "+95" ]; then
    MSG="Your mailbox is now over 95% full. Please remove some emails ASAP."
elif [ "$BOUNDARY" = "+80" ]; then
    MSG="Your mailbox is now over 80% full. Please consider removing some emails to save space."
elif [ "$BOUNDARY" = "-100" ]; then
    MSG="Your mailbox is now back to normal (<100%)."
fi

cat << EOF | /usr/lib/dovecot/dovecot-lda -d $USER -o "plugin/quota=maildir:User quota:noenforcing"
From: postmaster@wisecomnet.com
To: ${USER}
Bcc: postmaster@wisecomnet.com
Subject: Email Account Quota Warning

Dear ${USER},

$MSG

Best regards,
Your Mail System
EOF
```
Make the file executable
```
chomd +x /usr/local/bin/quota-warning.sh
```
Then
```
systemctl start dovecot
systemctl enable dovecot
```
6. Install and configure rspamd amd redis
```
pacman -S rspamd redis
systemctl start rspamd; systemctl enable rspamd
systemctl start redis; systemctl enable redis
mkdir /etc/rspamd/local.d
```
Create the following files
/etc/rspamd/local.d/arc.conf
```
path = "/var/lib/rspamd/dkim/$selector.key";
selector_map = "/etc/rspamd/dkim_selectors.map";
```
/etc/rspamd/local.d/classifier-bayes.conf
```
servers = "127.0.0.1";
backend = "redis";
autolearn = true;
```
/etc/rspamd/local.d/dkim_signing.conf
```
path = "/var/lib/rspamd/dkim/$selector.key";
selector_map = "/etc/rspamd/dkim_selectors.map";
allow_username_mismatch = true;
```
/etc/rspamd/local.d/redis.conf
```
servers = "127.0.0.1";
```
/etc/rspamd/local.d/worker-controller.inc
```
password "PASSWORD" #Password is generated by "rspamadm pw"
```
/etc/rspamd/local.d/worker-proxy.inc
```
bind_socket = "/var/run/rspamd/rspamd.sock mode=0660 owner=postfix";
upstream "local" {
  self_scan = yes; # Enable self-scan
}
```
/etc/rspamd/dkim_selectors.map
```
DOMAINNAME SELECTOR
```
/etc/dovecot/sieve/learn-ham.sieve
```
require ["vnd.dovecot.pipe", "copy", "imapsieve"];
pipe :copy "rspamd-learn-ham.sh";
```
/etc/dovecot/sieve/learn-spam.sieve
```
require ["vnd.dovecot.pipe", "copy", "imapsieve"];
pipe :copy "rspamd-learn-spam.sh";
```
/etc/dovecot/sieve/rspamd-learn-ham.sh
```
#!/bin/sh
exec /usr/bin/rspamc learn_ham
```
/etc/dovecot/sieve/rspamd-learn-spam.sh
```
#!/bin/sh
exec /usr/bin/rspamc learn_spam
```
/etc/dovecot/sieve-befor/spam-to-junk.sieve
```
require "fileinto";
if header :contains "X-Spam" "Yes" {
 fileinto "Junk";
 stop;
}
```
Compile the .sieve files
```
sievec /etc/dovecot/sieve
sievec /etc/dovecot/sieve-before
```
7. Fire it up

