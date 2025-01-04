---
layout: post
title: "Setting up an e-mail server on Slackware Linux/RHEL"
categories: Linux
author: "Andre C"
meta: "servers"
---

## A basic/rudimentary e-mail server that you can expose to the public internet

While this guide was made for Slackware, it should work for RHEL as well.

This is assuming you already have DNS set up as well as Apache...

### Required Software
- Postfix
- Dovecot
- Spamassassin, Amavisd-new
- Clamav

## Set up your hosts file
/etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.1.20   yourhostname.yourdomainname.com      your hostname

# Install the required software above either by compiling from source, SlackBuilds, or if on RHEL use yum

# Configure Postfix



Open Postfix config file /etc/postfix/main.cf and find and edit the following lines:

~~~
## Uncomment and set your mail server FQDN ##
myhostname = server.unixmen.com

## Uncomment and Set domain name ##
mydomain = unixmen.com

## Uncomment ##
myorigin = $mydomain

## Set ipv4 ##
inet_interfaces = all

## Change to all ##
inet_protocols = all

## Comment ##

#mydestination = $myhostname, localhost.$mydomain, localhost,

## Uncomment ##\
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain

##b Uncomment and add IP range ##
mynetworks = 192.168.1.0/24, 127.0.0.0/8

## Uncomment ##
home_mailbox = Maildir/

~~~

Save and exit the file. Start Postfix service now:

    systemctl restart postfix


## Test (skip this if you want)

Create a test user and set a password

    useradd testpf
    passwd testpf

Test with telnet

~~~
telnet localhost smtp

Trying ::1...
Connected to localhost.
Escape character is '^]'.
220 server.unixmen.com ESMTP Postfix
ehlo localhost
250-server.unixmen.com
250-PIPELINING
250-SIZE 10240000
250-VRFY
250-ETRN
250-ENHANCEDSTATUSCODES
250-8BITMIME
250 DSN
mail from:<sk>
250 2.1.0 Ok
rcpt to:<sk>
250 2.1.5 Ok
data
354 End data with <CR><LF>.<CR><LF>
welcome to unixmen mail system
.
250 2.0.0 Ok: queued as 3E68E284C
quit
221 2.0.0 Bye
Connection closed by foreign host.
~~~

## End of Test section, proceed if all is well


# Install and configure Dovecot

Open the file /etc/dovecot/dovecot.conf file and edit

    protocols = imap pop3 lmtp
	
Open /etc/dovecot/conf.d/10-mail.conf

    mail_location = maildir:~/Maildir	
	
Open /etc/dovecot/conf.d/10-auth.conf

    disable_plaintext_auth = yes

    auth_mechanisms = plain login	

Open the file /etc/dovecot/conf.d/10-master.conf

~~~
mode = 0600
   user = postfix
    group = postfix
~~~	

Start dovecot

    systemctl start dovecot
	
## Configure SASL

Open /etc/postfix/main.cf

~~~	
smtpd_sasl_auth_enable = yes
broken_sasl_auth_clients = yes
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_security_options = noanonymous

smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination
~~~	

Open /etc/dovecot.conf

~~~	
auth default {
    mechanisms = plain login
    passdb pam {
    }
    userdb passwd {
    }
    user = root
    socket listen {
      client {
        path = /var/spool/postfix/private/auth
        mode = 0660
        user = postfix
        group = postfix
      }
    }
}
~~~	

## Generate SSL certificates (to later configure TLS)

Ensure crypto-utils is installed

    genkey --days 365 mail.example.com
	
The keypair should be here

    /etc/pki/tls/certs/mail.example.com.cert  # public cert
    /etc/pki/tls/private/mail.example.com.key  # private key


Open /etc/postfix/main.cf:

~~~	
smtpd_tls_security_level = may
smtpd_tls_key_file = /etc/pki/tls/private/mail.example.com.key
smtpd_tls_cert_file = /etc/pki/tls/certs/mail.example.com.cert
# smtpd_tls_CAfile = /etc/pki/tls/root.crt
smtpd_tls_loglevel = 1
smtpd_tls_session_cache_timeout = 3600s
smtpd_tls_session_cache_database = btree:/var/lib/postfix/smtpd_tls_cache
tls_random_source = dev:/dev/urandom
tls_random_exchange_name = /var/lib/postfix/prng_exch
smtpd_tls_auth_only = yes

content_filter=amavisfeed:[127.0.0.1]:10024
~~~	

Open /etc/dovecot.conf:

~~~	
protocols = imap imaps pop3 pop3s
#disable_plaintext_auth = no
#ssl_disable = no
ssl_cert_file = /etc/pki/tls/certs/mail.example.com.cert
ssl_key_file = /etc/pki/tls/private/mail.example.com.key
ssl_cipher_list = ALL:!LOW:!SSLv2 
~~~	

Restart postfix and dovecot

# Configuring Amavisd-new/ClamAV and Spamassassin

Check 

    groups clamav
	
If clamav isn't added run

    gpasswd -a clamav amavis
	
	
Open /etc/clamd.conf

    LocalSocket /var/run/clamav/clamd.sock
	

Open /etc/amavisd.conf

~~~	
$max_servers = 2;                   # num of pre-forked children (2..30 is common), -m
$daemon_user  = "amavis";           # (no default;  customary: vscan or amavis), -u
$daemon_group = "amavis";           # (no default;  customary: vscan or amavis), -g
...
$inet_socket_port = 10024;          # listen on this local TCP port(s)
...

~~~	

~~~	

$mydomain = 'example.com';                 
$MYHOME = '/var/amavis';                 
$helpers_home = "$MYHOME/var";             
$lock_file = "$MYHOME/var/amavisd.lock";   
$pid_file  = "$MYHOME/var/amavisd.pid";    
$myhostname = 'mail.example.com';         

~~~	

~~~	
$sa_tag_level_deflt  = 2.0;               
$sa_tag2_level_deflt = 6.2;               
$sa_kill_level_deflt = 6.9;                
$sa_dsn_cutoff_level = 10;                
# $sa_quarantine_cutoff_level = 25;        
$penpals_bonus_score = 8;                  
$penpals_threshold_high = $sa_kill_level_deflt;        
$sa_mail_body_size_limit = 400*1024;       
$sa_local_tests_only = 0;                  
~~~	

Okay, that was a lot, theres a bit more to edit

Look for the clamav section


~~~	
### http://www.clamav.net/
['ClamAV-clamd',
  \&ask_daemon, ["CONTSCAN {}\n", "/var/run/clamav/clamd.sock"],
  qr/\bOK$/, qr/\bFOUND$/,
  qr/^.*?: (?!Infected Archive)(.*) FOUND$/ ],
# # NOTE: run clamd under the same user as amavisd, or run it under its own
# #   uid such as clamav, add user clamav to the amavis group, and then add
# #   AllowSupplementaryGroups to clamd.conf;
# # NOTE: match socket name (LocalSocket) in clamav.conf to the socket name in
# #   this entry; when running chrooted one may prefer socket "$MYHOME/clamd".
~~~	

Now open /etc/master.cf and add this

~~~
amavisfeed unix    -       -       n        -      2     lmtp
    -o lmtp_data_done_timeout=1200
    -o lmtp_send_xforward_command=yes
    -o disable_dns_lookups=yes
    -o max_use=20
	
~~~

~~~
127.0.0.1:10025 inet n    -       n       -       -     smtpd
    -o content_filter=
    -o smtpd_delay_reject=no
    -o smtpd_client_restrictions=permit_mynetworks,reject
    -o smtpd_helo_restrictions=
    -o smtpd_sender_restrictions=
    -o smtpd_recipient_restrictions=permit_mynetworks,reject
    -o smtpd_data_restrictions=reject_unauth_pipelining
    -o smtpd_end_of_data_restrictions=
    -o smtpd_restriction_classes=
    -o mynetworks=127.0.0.0/8
    -o smtpd_error_sleep_time=0
    -o smtpd_soft_error_limit=1001
    -o smtpd_hard_error_limit=1000
    -o smtpd_client_connection_count_limit=0
    -o smtpd_client_connection_rate_limit=0
    -o receive_override_options=no_header_body_checks,no_unknown_recipient_checks,no_milters,no_address_mappings
    -o local_header_rewrite_clients=
    -o smtpd_milters=
    -o local_recipient_maps=
    -o relay_recipient_maps=
~~~

## Reload postfix/dovecot 

You should be good to go and have a working public facing e-mail server that can filter spam

## Congratulations, you are done! But wait....want a graphical frontend?

The basic is Squirrelmail. Roundcubemail is a step up, but I prefer <a href="https://www.rainloop.net/" target="_blank">Rainloop</a>