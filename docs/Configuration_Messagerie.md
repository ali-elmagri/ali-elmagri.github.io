# Compte Rendu : Configuration Serveur Messagerie

Ce document détaille la configuration complète d'un serveur de messagerie pour le domaine constructions-dijonnaises.com  :
- **Authentification** : LDAP://10.50.255.254
- **Services** :  SpamAssassin, ClamAV, Dovecot, Postfix
- **Auto-configuration** : Thunderbird 


### Installer des Paquets

apt -y install postfix dovecot-core dovecot-imapd dovecot-lmtpd dovecot-managesieved \
  dovecot-sieve spamassassin spamc clamav-daemon clamav clamsmtp opendkim opendkim-tools \
  rsyslog mailutils apache2 postfix-ldap dovecot-ldap


### Création de l'Utilisateur Virtuel

Créer l'user propriétaire des messageries virtuelles :

useradd -d /var/mail -U -u 5000 vmail
mkdir -p /var/mail/vmail/constructions-dijonnaises.com
chown -R vmail:vmail /var/mail/vmail


## Configuration Postfix 

### Fichier /etc/mailname

On écrit "constructions-dijonnaises.com" dans /etc/mailname

### Fichier `/etc/postfix/main.cf`

Configuration complète du fichier principal de Postfix :

myhostname = constructions-dijonnaises.com
mydomain = constructions-dijonnaises.com
myorigin = $mydomain

mydestination = localhost
relay_domains = $mydomain

virtual_mailbox_domains = constructions-dijonnaises.com
virtual_mailbox_base = /var/mail/vmail
virtual_uid_maps = static:5000
virtual_gid_maps = static:5000

virtual_mailbox_maps = ldap:/etc/postfix/ldap_virtual_mailbox_maps.cf
virtual_alias_maps = ldap:/etc/postfix/ldap_virtual_aliases.cf

smtpd_tls_cert_file = /etc/letsencrypt/live/constructions-dijonnaises.com/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/constructions-dijonnaises.com/privkey.pem
smtpd_tls_security_level = may
smtp_tls_security_level = may
smtpd_use_tls = yes
smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtp_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1

smtpd_sasl_auth_enable = yes
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_authenticated_header = yes
smtpd_sasl_security_options = noanonymous, noplaintext
smtpd_sasl_tls_security_options = noanonymous

smtpd_helo_restrictions = 
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_invalid_helo_hostname

smtpd_sender_restrictions = 
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_sender_login_mismatch,
    reject_unknown_sender_domain

smtpd_recipient_restrictions = 
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination,
    reject_unverified_recipient

milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:localhost:12301
non_smtpd_milters = inet:localhost:12301

### Fichier `/etc/postfix/master.cf`


smtp      inet  n       -       y       -       -       smtpd

smtps     inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING

submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING

pickup    unix  n       -       y       60      1       pickup

cleanup   unix  n       -       y       -       0       cleanup

qmgr      unix  n       -       n       300     1       qmgr

tlsmgr    unix  -       -       y       1000?   1       tlsmgr

smtp      unix  -       -       y       -       -       smtp

smtp-amavis unix -    -       y       -       2       smtp
  -o smtp_data_done_timeout=1200
  -o smtp_send_xforward_command=yes

127.0.0.1:10025 inet n  -       y       -       -       smtpd
  -o content_filter=
  -o local_recipient_maps=
  -o relay_recipient_maps=
  -o smtpd_restriction_classes=
  -o smtpd_delay_reject=no
  -o smtpd_client_restrictions=permit_innetwork_submit_type,reject
  -o smtpd_helo_restrictions=
  -o smtpd_sender_restrictions=
  -o smtpd_recipient_restrictions=permit_innetwork_submit_type,reject
  -o smtpd_relay_restrictions=permit_this_host,reject
  -o smtpd_known_hostname_classes=
  -o smtpd_etrn_restrictions=
  -o smtpd_data_restrictions=reject_unauth_pipelining
  -o smtpd_end_of_data_restrictions=
  -o mynetworks_style=host
  -o mynetworks=127.0.0.0/8
  -o relay_domains_maps=
  -o relay_recipient_maps=
  -o local_recipient_maps=

local     unix  -       n       n       -       -       local

virtual   unix  -       n       n       -       -       virtual

dovecot   unix  -       n       n       -       -       lmtp
  -o lmtp_deliver_xforward_command=yes
  -o lmtp_destination_concurrency_limit=1
  -o lmtp_destination_recipient_limit=1

bounce    unix  -       -       y       -       0       bounce

defer     unix  -       -       y       -       0       bounce

trace     unix  -       -       y       -       0       bounce

retry     unix  -       -       y       -       0       bounce

verify    unix  -       -       y       -       1       verify

flush     unix  n       -       y       1000?   0       flush

proxymap  unix  -       -       n       -       -       proxymap

anvil     unix  -       -       y       1       1       anvil

scache    unix  -       -       y       100     1       scache


Créer et configurer les fichiers LDAP en remplaçant le domaine et le mot de passe.

#### Fichier : `/etc/postfix/ldap_virtual_mailbox_maps.cf`

server_host = ldap://10.50.255.254
version = 3
bind = yes
bind_dn = cn=mail-ro,dc=constructions-dijonnaises,dc=com
bind_pw = tprezo
search_base = ou=Utilisateurs,dc=constructions-dijonnaises,dc=com
scope = sub
query_filter = (&(mail=%s)(objectClass=inetOrgPerson))
result_attribute = mail

#### Fichier : `/etc/postfix/ldap_virtual_aliases.cf`

server_host = ldap://10.50.255.254
version = 3
bind = yes
bind_dn = cn=mail-ro,dc=constructions-dijonnaises,dc=com
bind_pw = tprezo
search_base = ou=Utilisateurs,dc=constructions-dijonnaises,dc=com
scope = sub
query_filter = (&(mail=%s)(objectClass=alias))
result_attribute = mail


## Configuration Dovecot 

### Fichier `/etc/dovecot/conf.d/10-mail.conf`


mail_driver = maildir
mail_path = /var/mail/vmail/%d/%n

mail_uid = 5000
mail_gid = 5000
mail_privileged_group = vmail

mail_access_groups = vmail

maildir_very_dirty_syncs = no
mmap_disable = no
dotlock_use_excl = yes

fsync_disable = no
maildir_stat_limit = 100

mail_plugins = quota sieve


## Fichier `/etc/dovecot/conf.d/10-master.conf`


service imap-login {
    inet_listener imaps {
        port = 993
        ssl = yes
    }
}

service lmtp { 
    unix_listener /var/spool/postfix/private/dovecot-lmtp {
        mode = 0600
        user = postfix 
        group = postfix 
    } 
}

service auth { 
    unix_listener /var/spool/postfix/private/auth {
        mode = 0666
        user = postfix 
        group = postfix
    }
}

### Fichier `/etc/dovecot/conf.d/10-ssl.conf`


openssl dhparam 2048 > /etc/dovecot/dh.pem
chmod 600 /etc/dovecot/dh.pem


Configuration complete SSL/TLS :

ssl = required
ssl_cert = /etc/letsencrypt/live/constructions-dijonnaises.com/fullchain.pem
ssl_key = /etc/letsencrypt/live/constructions-dijonnaises.com/privkey.pem

ssl_dh = /etc/dovecot/dh.pem

ssl_protocols = TLSv1.2 TLSv1.3
ssl_ciphers = HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4
ssl_prefer_server_ciphers = yes

ssl_client_cert = no

ssl_require_crl = no

ssl_options = no_compression


### Configuration de l'Authentification LDAP

**Fichier `/etc/dovecot/conf.d/10-auth.conf`**  :

auth_mechanisms = plain login cram-md5

!include auth-system.conf.ext
!include auth-ldap.conf.ext


## Fichier `/etc/dovecot/conf.d/auth-ldap.conf.ext` :

ldap_hosts = 10.50.255.254
ldap_uris = ldap://10.50.255.254/
ldap_version = 3
ldap_timeout = 5

ldap_auth_dn = cn=mail-ro,dc=constructions-dijonnaises,dc=com
ldap_auth_bind_userdn = yes
ldap_auth_dn_password = tprezo

ldap_base = ou=Utilisateurs,dc=constructions-dijonnaises,dc=com

ldap_deref = never

ldap_scope = subtree

ldap_tls = no
ldap_tls_ca_cert_file = /etc/ssl/certs/ca-certificates.crt
ldap_tls_ca_cert_dir = /etc/ssl/certs
ldap_tls_cipher_suite = HIGH:!aNULL
ldap_tls_require_cert = demand



## Ficher `/etc/dovecot/dovecot-ldap.conf.ext`:

hosts = 10.50.255.254
port = 389
dn = cn=mail-ro,dc=constructions-dijonnaises,dc=com
dnpass = tprezo
base = ou=Utilisateurs,dc=constructions-dijonnaises,dc=com
scope = subtree
timeout = 5

filter = (&(objectClass=posixAccount)(|(uid=%u)(mail=%u)))
pass_filter = (&(objectClass=posixAccount)(|(uid=%u)(mail=%u)))
user_filter = (&(objectClass=posixAccount)(|(uid=%u)(mail=%u)))
user_attrs = =uid=5000,=gid=5000,=home=/var/mail/vmail/%d/%n

pass_attrs = mail=user,userPassword=password

iterate_attrs = uid=user
iterate_filter = (&(objectClass=posixAccount)(uid=*))


### Gestion des Quotas 

## Fichier `/etc/dovecot/conf.d/20-imap.conf` :

protocol imap {
  imap_max_line_length = 65536

  imap_idle_notify_interval = 2 mins

  mail_plugins = $mail_plugins quota imap_quota

  imap_logout_format = in=%i out=%o deleted=%{deleted} expunged=%{expunged} trashed=%{trashed} hdr_count=%{fetch_hdr_count} hdr_bytes=%{fetch_hdr_bytes} body_count=%{fetch_body_count} body_bytes=%{fetch_body_bytes}
}

imap_capability = IMAP4rev1 SASL-IR LOGIN-REFERRALS ID ENABLE IDLE NAMESPACE LITERAL+ STARTTLS

**Fichier `/etc/dovecot/conf.d/90-plugin.conf`** - Configuration des quotas :

plugin {
  quota = maildir:storage=5G:ignore=Junk:ignore=Trash
  
  #quota = LDAP:/etc/dovecot/dovecot-ldap-quota.conf.ext

  quota_rule = *:storage=5G:messages=unlimited
  
  quota_grace = 20%

  quota_status_success = OK
  quota_status_nouser = NOUSER
  quota_status_overquota = OVERQUOTA

  sieve = /var/mail/vmail/%d/%n/.dovecot.sieve
  sieve_dir = /var/mail/vmail/%d/%n/sieve
  sieve_global_dir = /var/lib/dovecot/sieve/global/
  sieve_before = /var/lib/dovecot/sieve/before.sieve
  sieve_after = /var/lib/dovecot/sieve/after.sieve

  autocreate = Drafts
  autocreate2 = Sent
  autocreate3 = Trash
  autocreate4 = Junk
  autosubscribe = Drafts
  autosubscribe2 = Sent
  autosubscribe3 = Trash
  autosubscribe4 = Junk
}

## Thunderbird

### Fichier `/etc/apache2/sites-available/000-default.conf`


ServerName constructions-dijonnaises.com


### Fichier `/etc/apache2/sites-available/thunderbird-autoconfig.conf`


<VirtualHost *:80>
    ServerName autoconfig.constructions-dijonnaises.com
    DocumentRoot /var/www/html/autoconfig/

    <Directory /var/www/html/autoconfig/>
        Options -Indexes +FollowSymLinks +MultiViews -Includes -ExecCGI
        AllowOverride None
        Order allow,deny
        Allow from all
    </Directory>
</VirtualHost>


Recharger Apache :


a2ensite thunderbird-autoconfig
systemctl reload apache2


### Fichier `/var/www/html/autoconfig/mail/config-v1.1.xml`


<?xml version="1.0" encoding="UTF-8"?>
<clientConfig version="1.1">
  <emailProvider id="constructions-dijonnaises.com">
    <domain>constructions-dijonnaises.com</domain>
    <displayName>Constructions Dijonnaises Mail</displayName>
    <displayShortName>CD Mail</displayShortName>

    <incomingServer type="imap">
      <hostname>constructions-dijonnaises.com</hostname>
      <port>993</port>
      <socketType>SSL</socketType>
      <authentication>password-cleartext</authentication>
      <username>%EMAILADDRESS%</username>
    </incomingServer>

    <outgoingServer type="smtp">
      <hostname>constructions-dijonnaises.com</hostname>
      <port>465</port>
      <socketType>SSL</socketType>
      <authentication>password-cleartext</authentication>
      <username>%EMAILADDRESS%</username>
    </outgoingServer>
  </emailProvider>
</clientConfig>


## Redémarrage des Services et Tests

### Commandes de Redémarrage


systemctl restart apache2
systemctl restart opendkim
systemctl restart postfix
systemctl restart dovecot
systemctl restart spamd
systemctl restart clamsmtp
systemctl restart clamav-daemon

### Tests d'Authentification et de Flux


doveadm auth test -x service=imap <adresse_ip>


Tester l'envoi de mail :

echo "Test du mail reussi" | mailx -s "test du postfix" <adresse_ldap>

Vérifier la présence du fichier dans le répertoire user :


/var/mail/vmail/constructions-dijonnaises.com/<user>/new



**Identifiants LDAP à utiliser :**
- Serveur : `ldap://10.50.255.254`
- Mot de passe : `tprezo`
