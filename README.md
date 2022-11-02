# Postfix-Dovecot-Virtual-Users-Mail-Server
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
# disable "new mail" notifications for local unix users
biff = no
# Name of this mail server, used in the SMTP HELO for outgoing mail. Make
# sure this resolves to the same IP as your reverse DNS hostname.
myhostname = mail.example.com
# Domains for which postfix will deliver local mail. Does not apply to
# virtual domains, which are configured below. Make sure to specify the FQDN
# of your sever, as well as localhost.
# Note: NEVER specify any virtual domains here!!! Those come later.
#mydestination = $myhostname, localhost.$mydomain, localhost, mail.$mydomain
mydestination = $myhostname, localhost.$mydomain, localhost

# SENDING MAIL
# Domain appended to mail sent locally from this machine - such as mail sent
# via the `sendmail` command.
myorigin = $myhostname
# mynetwork_style = host #unused
sender_bcc_maps = texthash:/etc/postfix/sender_bcc
recipient_bcc_maps = texthash:/etc/postfix/recipient_bcc

# prevent spammers from searching for valid users
disable_vrfy_command = yes

# require properly formatted email addresses - prevents a lot of spam
strict_rfc821_envelopes = yes

# don't give any helpful info when a mailbox doesn't exist
show_user_unknown_table_name = no

# limit maximum e-mail size to 50MB. mailbox size must be at least as big as
# the message size for the mail to be accepted, but has no meaning after
# that since we are using Dovecot for delivery.
message_size_limit = 51200000
mailbox_size_limit = 51200000

# require addresses of the form "user@domain.tld"
allow_percent_hack = no
swap_bangpath = no

# allow plus-aliasing: "user+tag@domain.tld" delivers to "user" mailbox
recipient_delimiter = +

# path to the SSL certificate for the mail server
smtpd_tls_cert_file = /etc/letsencrypt/live/mail.example.com/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/mail.example.com/privkey.pem

# These three lines define how postfix will connect to other mail servers.
# DANE is a stronger form of opportunistic TLS. You can read about it here:
# http://www.postfix.org/TLS_README.html#client_tls_dane
smtp_tls_security_level = dane
smtp_dns_support_level = dnssec
smtp_host_lookup=dns
# DANE requires a DNSSEC capable resolver. If your DNS resolver doesn't
# support DNSSEC, remove the above two lines and uncomment the below:
#smtp_tls_security_level = may
# allow other mail servers to connect using TLS, but don't require it
#smtpd_tls_security_level = may

# tickets and compression have known vulnerabilities
tls_ssl_options = no_ticket, no_compression

# Implement foreward secrecy as per http://www.postfix.org/FORWARD_SECRECY_README.html#quick-start
smtpd_tls_dh1024_param_file = ${config_directory}/dh2048.pem
smtpd_tls_dh512_param_file = ${config_directory}/dh512.pem

# cache incoming and outgoing TLS sessions
#smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_tlscache
#smtp_tls_session_cache_database  = btree:${data_directory}/smtp_tlscache

# enable SMTPD auth. Dovecot will place an `auth` socket in postfix's
# runtime directory that we will use for authentication.
smtpd_sasl_auth_enable = yes
smtpd_sasl_path = /var/run/dovecot/auth-client
smtpd_sasl_type = dovecot
smtpd_sasl_authenticated_header = yes


# only allow authentication over TLS
smtpd_tls_auth_only = yes

#don't allow plaintext auth methods on unencrypted connections
smtpd_sasl_security_options = noanonymous, noplaintext
# but plaintext auth is fine when using TLS
smtpd_sasl_tls_security_options = noanonymous

# add a message header when email was recieved over TLS
smtpd_tls_received_header = yes

# require that connecting mail servers identify themselves - this greatly
# reduces spam
smtpd_helo_required = yes

# The following block specifies some security restrictions for incoming
# mail. The gist of it is, authenticated users and connections from
# localhost can do anything they want. Random people connecting over the
# internet are treated with more suspicion: they must have a reverse DNS
# entry and present a valid, FQDN HELO hostname. In addition, they can only
# send mail to valid mailboxes on the server, and the sender's domain must
# actually exist.
smtpd_client_restrictions =
  permit_mynetworks,
  permit_sasl_authenticated,
  reject_unknown_reverse_client_hostname,
# you might want to consider:
#  reject_unknown_client_hostname,
# here. This will reject all incoming connections without a reverse DNS
# entry that resolves back to the client's IP address. This is a very
# restrictive check and may reject legitimate mail.
  reject_unauth_pipelining
smtpd_helo_restrictions =
  permit_mynetworks,
  permit_sasl_authenticated,
  reject_invalid_helo_hostname,
  reject_non_fqdn_helo_hostname,
# you might want to consider:
#  reject_unknown_helo_hostname,
# here. This will reject all incoming mail without a HELO hostname that
# properly resolves in DNS. This is a somewhat restrictive check and may
# reject legitimate mail.
  reject_unauth_pipelining
smtpd_sender_restrictions =
  permit_mynetworks,
  permit_sasl_authenticated,
  reject_non_fqdn_sender,
  reject_unknown_sender_domain,
  reject_unauth_pipelining,
  #  reject_unknown_reverse_client_hostname,
  #  reject_unknown_client_hostname,
  check_sender_access      texthash:/etc/postfix/rejected_tlds
smtpd_reject_unlisted_sender = yes  
 
smtpd_relay_restrictions = 
  permit_mynetworks, 
  permit_sasl_authenticated, 
# !!! THIS SETTING PREVENTS YOU FROM BEING AN OPEN RELAY !!!
  reject_unauth_destination
# !!!      DO NOT REMOVE IT UNDER ANY CIRCUMSTANCES      !!!
smtpd_recipient_restrictions =
  check_client_access texthash:/etc/postfix/client_checks,
  check_sender_access      texthash:/etc/postfix/sender_access,
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

# ALIAS DATABASE
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
# debugger_command =
#	PATH=/bin:/usr/bin:/usr/local/bin; export PATH; (echo cont;
#	echo where) | gdb $daemon_directory/$process_name $process_id 2>&1
#	>$config_directory/$process_name.$process_id.log & sleep 5
# debugger_command =
#	PATH=/bin:/usr/bin:/sbin:/usr/sbin; export PATH; screen
#	-dmS $process_name gdb $daemon_directory/$process_name
#	$process_id & sleep 1

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
  
#non_smtpd_milters=unix:/var/run/rspamd/rspamd.sock
#smtpd_milters=unix:/var/run/rspamd/rspamd.sock
#non_smtpd_milters=inet:127.0.0.1:11332
#smtpd_milters=inet:127.0.0.1:11332
#milter_protocol = 6
#milter_mail_macros=i {mail_addr} {client_addr} {client_name} {auth_authen}
#milter_default_action = accept
smtp_header_checks = regexp:/etc/postfix/smtp_header_checks
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
       swl.spamhaus.org*-4,
#       list.dnswl.org=127.[0..255].[0..255].0*-2,
#       list.dnswl.org=127.[0..255].[0..255].1*-4,
#       list.dnswl.org=127.[0..255].[0..255].[2..3]*-6
```
5. 

