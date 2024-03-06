# Οδηγίες εγκατάστασης και παραμετροποίσης Shibboleth SP v3.x on Debian-Ubuntu Linux

<img width="120px" src="https://grnet.gr/wp-content/uploads/2023/01/01_EDYTE_LOGO_COLOR-1.png" />

## Περιεχόμενα

1. [Προαπαιτούμενα](#απαιτήσεις-συστήματος)
2. [Λογισμικό που θα εγκατασταθεί](#λογισμικό-που-θα-εγκατασταθεί)
3. [Άλλες απαιτήσεις](#άλλες-απαιτήσεις)
4. [Οδηγίες εγκατάστασης](#οδηγίες-εγκατάστασης)
   1. [Εγκατάσταση των προαπαιτούμενων λογισμικών](#install-software-requirements)
   2. [Παραμετροποίηση environment](#προετοιμασία-περιβάλλοντος)
   3. [Εγκατάσταση Shibboleth Service Provider](#εγκατάσταση-shibboleth-service-provider)
5. [Οδηγίες παραμετροποίησης](#οδηγίες-παραμετροποίησης)
   1. [Παράδειγμα παραμετροποίησης SSL σε Apache2](#παραμετροποίηση-ssl-και-apache2)
   2. [Παραμετροποίηση Shibboleth SP](#παραμετροποίηση-shibboleth-sp)
   3. [Ενεργοποίηση υποστήριξης των επιθυμητών Attributes στο Shibboleth SP](#ενεργοποίηση-υποστήριξης-των-επιθυμητών-attributes-στο-shibboleth-sp)
   4. [Κατανάλωση metadata grnet και edugain](#κατανάλωση-metadata-grnet-και-edugain)
7. [Άυξηση startup timeout](#αυξηση-startup-timeout)
8. [OPTIONAL - Maintain 'shibd' working](#optional---maintain-shibd-working)
9. [Utility](#utility)
10. [Authors](#authors)
11. [Thanks](#thanks)


## Απαιτήσεις συστήματος

 * CPU: 2 Core
 * RAM: 4 GB
 * HDD: 20 GB
 * OS: Debian 10

## Λογισμικό που θα εγκατασταθεί

 * ca-certificates
 * ntp
 * vim
 * libapache2-mod-php, php, libapache2-mod-shib, apache2 (>= 2.4)
 * openssl

## Άλλες απαιτήσεις

 * SSL Credentials: HTTPS Certificate & Key

## Notes

Σε αυτόν τον οδηγό χρησιμοποιείται το `example.org` για domain name και `sp.example.org` για FQDN (Full Qualified Domain Name) για να δώσουμε κάποιες τιμές στα παραδείγματά μας

Παρακαλούμε πολύ, θυμηθείτε να **αντικαταστήσετε** τα `example.org` domain name  και `sp.example.org` με τις τιμές αυτών του δικού σας σέρβερ.

## Οδηγίες Εγκατάστασης

### Install software requirements

1. Become ROOT:
   * `sudo su -`
   
2. Ενημερώσεις πακέτων:
   * `apt update && apt-get upgrade -y --no-install-recommends`
  
3. Εγκατάσταση απαιτούμενων πακέτων: 
   * `apt install ca-certificates vim openssl`

### Προετοιμασία περιβάλλοντος

1. Επεξεργαστείτε το  `/etc/hosts`:
   * `vim /etc/hosts`
  
     ```bash
     127.0.1.1 sp.example.org sp
     ```
   (*Αντικαταστήστε το `sp.example.org` με το δικό σας SP Full Qualified Domain Name*)
   
   (*Αντικαταστήστε `sp` με το hostname του SP*)

### Εγκατάσταση Shibboleth Service Provider

1. Become ROOT: 
   * `sudo su -`

2. Εγκατάσταση Shibboleth SP:
   * `apt install apache2 libapache2-mod-shib ntp --no-install-recommends`

   Από δω και στο εξής η τοποθεσία του SP στον server  θα είναι στο /etc/shibboleth


## Οδηγίες παραμετροποίησης

### Παραμετροποίηση SSL και Apache2

> Η παραμετροποίηση αυτή είναι ένα παράδειγμα, προσαρμόστε ανάλογα των απαιτήσεών σας

1. Become ROOT:
   * `sudo su -`

2. Δημιουργία του DocumentRoot:
   ```bash
   mkdir /var/www/html/myservice
   
   echo '<h1>It Works!</h1>' > /var/www/html/myservice/index.html
   ```
3. Εγκατάσταση certbot για τα SSL πιστοποιητικά
   * `sudo snap install --classic certbot`
   * `sudo ln -s /snap/bin/certbot /usr/bin/certbot`

   Εγκατάσταση πιστοποιητικού για apache
   * `sudo certbot --apache`

4. Δημιουργία Virtualhost 

    Επεξεργαστείτε το αρχείο

   ```/etc/apache2/sites-enabled/000-default-le-ssl.conf ```
   ```bash 
   <IfModule mod_ssl.c>
    <VirtualHost _default_:443>
    ServerAdmin info@kapoiosp.com
    ServerName localhost
    DocumentRoot /var/www/html/myservice
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/καποιοσπ/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/καποιοσπ/privkey.pem
    <FilesMatch "\.(cgi|shtml|phtml|php)$">
        SSLOptions +StdEnvVars
    </FilesMatch>
    <Directory /usr/lib/cgi-bin>
      SSLOptions +StdEnvVars
    </Directory>
    BrowserMatch "MSIE [2-6]" \
      nokeepalive ssl-unclean-shutdown \
      downgrade-1.0 force-response-1.0
    BrowserMatch "MSIE [17-9]" ssl-unclean-shutdown
    Alias /myservice/ /var/www/html/myservice/
    <Location /myservice/>
      AuthType shibboleth
      ShibRequestSetting requireSession 1
      Require valid-user
    </Location>
   </VirtualHost>
   </IfModule>
   ```

5. Ενεργοποίηση **proxy_http**, **SSL** και **headers** Apache2 modules:
   ```bash
   a2enmod ssl headers alias include negotiation
      
   systemctl restart apache2.service
   ```

### Παραμετροποίηση Shibboleth SP

1. Become ROOT: 
   * `sudo su -`

2. Αλλαγή του SP entityID και email επικοινωνίας τεχνικού υπεύθυνου:
   ```bash
   sed -i "s/sp.example.org/$(hostname -f)/" /etc/shibboleth/shibboleth2.xml
   
   sed -i "s/root@localhost/<TECH-CONTACT-EMAIL-ADDRESS-HERE>/" /etc/shibboleth/shibboleth2.xml
   
   sed -i 's/handlerSSL="false"/handlerSSL="true"/' /etc/shibboleth/shibboleth2.xml
   
   sed -i 's/cookieProps="http"/cookieProps="https"/' /etc/shibboleth/shibboleth2.xml
   
   sed -i 's/cookieProps="https">/cookieProps="https" redirectLimit="exact">/' /etc/shibboleth/shibboleth2.xml
   ```

3. Δημιουργία ζεύγους κλειδιών Signing και Encryption για τα SP metadata:
   * Ubuntu:
     ```bash
     cd /etc/shibboleth
   
     shib-keygen -u _shibd -g _shibd -h $(hostname -f) -y 30 -e https://$(hostname -f)/shibboleth -n sp-signing -f
   
     shib-keygen -u _shibd -g _shibd -h $(hostname -f) -y 30 -e https://$(hostname -f)/shibboleth -n sp-encrypt -f
   
     /usr/sbin/shibd -t
 
     systemctl restart shibd.service
   
     systemctl restart apache2.service
     ```

   * Debian
     ```bash
     cd /etc/shibboleth
   
     ./keygen.sh -u shibd -g shibd -h $(hostname -f) -y 30 -e https://$(hostname -f)/shibboleth -n sp-signing -f
   
     ./keygen.sh -u shibd -g shibd -h $(hostname -f) -y 30 -e https://$(hostname -f)/shibboleth -n sp-encrypt -f
   
     LD_LIBRARY_PATH=/opt/shibboleth/lib64 /usr/sbin/shibd -t
 
     systemctl restart shibd.service
   
     systemctl restart apache2.service
     ```

4. Ενεργοποίηση Shibboleth Apache2:
   ```bash
   a2enmod shib
   
   systemctl reload apache2.service
   ```

5. Πλέον μπορείτε να δείτε τα metadata του SP στο:
   * ht<span>tps://</span>sp.example.org/Shibboleth.sso/Metadata

     (*Αντικαταστήστε το `sp.example.org` με το FQDN σας*)


### Ενεργοποίηση υποστήριξης των επιθυμητών Attributes στο Shibboleth SP
> Το Attribute Map αρχείο χρησιμοποιείται από το Service Provider για την αναγνώριση και υποστήριξη των attributes που απελευθερώνονται από τους Identity Providers

Για να ενεργοποιήσετε τα επιθυμητά attributes θα χρειαστεί να τα βγάλετε από comments στο αρχείο `/etc/shibboleth/attribute-map.xml` και στην συνέχεια προχωρήστε σε restart του `shibd` service:
* `sudo systemctl restart shibd.service`

### Κατανάλωση metadata grnet και edugain
Επεξεργαστείτε το ```/etc/shibboleth/shibboleth2.xml```
και προσθέστε το παρακάτω
```bash
<MetadataProvider type="XML" uri="https://md.aai.grnet.gr/aggregates/grnet-metadata.xml"
backingFilePath="metadata-delos-grnet.xml" reloadInterval="7200">
  <MetadataFilter type="RequireValidUntil" maxValidityInterval="604800"/> 
  <MetadataFilter type="Signature" certificate="grnet-mdsigner.crt"/>
</MetadataProvider>
```
ώπου το mdsigner-delos-grnet.crt μπορείτε να το βρείτε εδώ
``` https://md.aai.grnet.gr/grnetaai_md_cert.html ```
αποθηκεύστε το τοπικά με 
```bash
cd /etc/shibboleth/
curl -o grnet-mdsigner.crt https://md.aai.grnet.gr/grnetaai_md_cert.pem
```
Αντίστοιχα για το eduGAIN:
```bash
<MetadataProvider type="XML" uri="https://md.aai.grnet.gr/feeds/edugain-idp-samlmd.xml"
  backingFilePath="metadata-default-edugain_all.xml" reloadInterval="7200">
  <MetadataFilter type="RequireValidUntil" maxValidityInterval="604800"/>
  <MetadataFilter type="Signature" certificate="grnet-mdsigner.crt"/>
</MetadataProvider>
```

Τα metadata του eduGAIN λογω του αρκετά μεγάλου αριθμού συμμετεχόντων έχουν αυξηθεί πολύ σε μέγεθος. 
Για τον λόγο αυτό προτείνεται η αύξηση του timeout στην εκκίνηση του SP


Εναλλακτικά, μπορεί να χρησιμοποιηθεί η **ΔΟΚΙΜΑΣΤΙΚΗ/STAGING** υπηρεσία mdq [https://github.com/iay/md-query] του GRNET. 
Για την ενεργοποίηση αυτού 
```bash
<MetadataProvider type="Dynamic" ignoreTransport="true" maxCacheDuration="86400" minCacheDuration="60">
    <Subst>http://mdq.grnet.gr:8080/entities/$entityID</Subst>
</MetadataProvider>
```


## Αύξηση του startup timeout

Shibboleth Documentation: https://wiki.shibboleth.net/confluence/display/SP3/LinuxSystemd

```bash
sudo mkdir /etc/systemd/system/shibd.service.d

echo -e '[Service]\nTimeoutStartSec=60min' | sudo tee /etc/systemd/system/shibd.service.d/timeout.conf

sudo systemctl daemon-reload

sudo systemctl restart shibd.service
```

## OPTIONAL - Maintain '```shibd```' working

1. Edit '`shibd`' init script:
   * `vim /etc/init.d/shibd`

     ```bash
     #...other lines...
     ### END INIT INFO

     # Import useful functions like 'status_of_proc' needed to 'status'
     . /lib/lsb/init-functions

     #...other lines...

     # Add 'status' operation
     status)
       status_of_proc -p "$PIDFILE" "$DAEMON" "$NAME" && exit 0 || exit $?
       ;;
     *)
       echo "Usage: $SCRIPTNAME {start|stop|restart|force-reload|status}" >&2
       exit 1
       ;;

     esac
     exit 0
     ```
2. Create a new watchdog for '`shibd`':
   * `vim /etc/cron.hourly/watch-shibd.sh`

     ```bash
     #! /bin/bash
     SERVICE=/etc/init.d/shibd
     STOPPED_MESSAGE="failed"

     if [[ "`$SERVICE status`" == *"$STOPPED_MESSAGE"* ]];
     then
       $SERVICE stop
       sleep 1
       $SERVICE start
     fi
     ```

3. Reload daemon:
   * `systemctl daemon-reload`

## Authors
* Halil Adem για το GRNET
 
## Thanks
 * GARR WIKI: https://github.com/ConsortiumGARR/idem-tutorials/blob/master/idem-fedops/HOWTO-Shibboleth/Service%20Provider/Debian/HOWTO%20Install%20and%20Configure%20a%20Shibboleth%20SP%20v3.x%20on%20Debian-Ubuntu%20Linux.md
 