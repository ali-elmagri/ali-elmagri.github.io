# Compte-rendu :  Serveur DNS linnormandie.fr

Ce rapport documente la mise en place d'un serveur DNS pour le domaine `linnormandie.fr` sur une machine Debian.
Le serveur DNS sera le seul serveur du domaine. 
Le TTL est de 30 minutes. Le référent technique est laurent.martin@linnormandie.fr. 
Un enregistrement TXT pour le domaine est à ajouter avec comme contenu :
'v=spf1 a mx -all'

### Tableau de la configuration

| Usage| Nom de domaine	 | IPv4 | IPv6 |
|------|-----------------|------|------|
| DNS | dns.linnormandie.fr | 10.50.110.34 
| Web/Mail Principal | linnormandie.fr (@) | 10.204.52.1 | fe80::b46d:feaa:5cfe:36fd |
| Messagerie | smtp.linnormandie.fr | 10.204.52.2 |
| Support | support.linnormandie.fr | 10.204.52.3 

**Pré-requis**
- Modification de `/etc/hostname` et `/etc/hosts` pour dns.linnormandie.fr

**Modifier le nom de l'hôte dans le fichier /etc/hosts :**

127.0.0.1   localhost                                                  
127.0.1.1   dns.linnormandie.fr dns                                    
                                                                       
**Modifier le nom de l'hôte dans le fichier /etc/hostname :**

dns.linnormandie.fr     

**Installer des composants :**

apt -y install bind9 dnsutils 


### 3 Configuration des Zones DNS

#### A. Zone  (/etc/bind/db.linnormandie.fr)


;                                                                      
; BIND data file for local loopback interface                          
;                                                                      
$TTL    1800                                                           
@       IN      SOA     dns.linnormandie.fr. laurent.martin.linnormandie.fr. (2025112601 1800 600 700 1800)                                   
;                                                                      
@       IN      NS      dns.linnormandie.fr.                           
@       IN      A       10.204.52.1                                    
dns     IN      A       10.50.110.34                                   
smtp    IN      A       10.204.52.2                                    
imap    IN      A       10.204.52.2                                    
www     IN      A       10.204.52.1                                    
support IN      A       10.204.52.3                                    
mail    IN      A       10.204.52.1                                    
linnormandie.fr.        IN MX 10        smtp.linnormandie.fr.          
linnormandie.fr.        IN      AAAA    fe80::b46d:feaa:5cfe:36fd      
@ IN TXT "v=spf1 a mx -all"                                                               


#### B. Zones Inverses

**Réseau IPv4 (/etc/bind/db.linnormandie.fr.inv) :**

;                                                                      
; BIND reverse data file for local loopback interface                  
;                                                                      
$TTL    1800                                                           
@       IN      SOA     dns.linnormandie.fr.    laurent.martin.linnormandie.fr.        (2025112601 1800 600 700 1800)                         
;                                                                      
@       IN      NS      dns.linnormandie.fr.                           
1.52.204.10.in-addr.arpa.       IN PTR  linnormandie.fr.               
1.52.204.10.in-addr.arpa.       IN PTR  mail.linnormandie.fr.          
1.52.204.10.in-addr.arpa.       IN PTR  www.linnormandie.fr.           
2.52.204.10.in-addr.arpa.       IN PTR  imap.linnormandie.fr.          
2.52.204.10.in-addr.arpa.       IN PTR  smtp.linnormandie.fr.          
3.52.204.10.in-addr.arpa.       IN PTR  support.linnormandie.fr.       
@       IN      TXT     "v=spf1 a mx -all"         


**Réseau IPv6 (/etc/bind/db.linnormandie.fr.inv6) :**

;                                                                      
; BIND reverse data file for local loopback interface                  
;                                                                      
;                                                                      
$TTL    1800                                                           
@       IN      SOA     dns.linnormandie.fr. laurent.martin.linnormandi
e.fr. (2025112601 1800 86400 2419200 1800)                             
;                                                                      
@       IN      NS      dns.linnormandie.fr.                           
d.f.6.3.e.f.c.5.a.a.e.f.d.6.4.b.0.0.0.0.0.0.0.0.0.0.0.0.0.8.e.f.ip6.arpa.  IN PTR linnormandie.fr.                                       
@       IN      TXT     "v=spf1 a mx -all"    


#### A. Configuration des Zones (/etc/bind/named.conf.local)


zone "linnormandie.fr" {                                               
        type master;                                                   
        file "/etc/bind/db.linnormandie.fr";                           
        forwarders{};                                                  
};                                                                     
                                                                       
zone "52.204.10.in-addr.arpa" {                                        
        type master;                                                   
        file "/etc/bind/db.linnormandie.fr.inv";                       
        forwarders{};                                                  
};                                                                     
                                                                       
zone "0.0.0.0.0.0.0.0.0.0.0.0.0.8.e.f.ip6.arpa" {                      
        type master;                                                   
        file "/etc/bind/db.linnormandie.fr.inv6";                      
        forwarders{};                                                  
};                             


#### B. Options Globales (/etc/bind/named.conf.options)


options {                                                              
        directory "/var/cache/bind";                                   
        allow-query { any; };                                          
        forward only;                                                  
        forwarders {                                                   
                1.1.1.1;                                               
                8.8.8.8;                                               
        };                                                             
};             



Redémarrer bind le service :

systemctl restart bind9


Tests de résolution :

dig @127.0.0.1 www.linnormandie.fr
dig @127.0.0.1 antoinepernot.frr

