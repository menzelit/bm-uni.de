# LDAP 04 | LDAP Account Manager

**WICHTIG:** Alle Kommandos werden als root-User ausgeführt!

# Grundinstallation

Dateien gibt es über die [LAM-Webseite](https://ldap-account-manager.org/lamcms/LAMProReleases).

```bash
apt install /root/ldap-account-manager*.deb ldap-utils -y
```

# Webserver

## Apache-2-Config für den LAM

```bash
cat <<EOF >/etc/apache2/sites-available/lam.${domain}.conf
<VirtualHost *:80>
        ServerName lam.${domain}
        Redirect permanent / https://lam.${domain}
        ErrorLog \${APACHE_LOG_DIR}/error.log
        CustomLog \${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
<VirtualHost *:443>
        ServerName lam.${domain}
        DocumentRoot /usr/share/ldap-account-manager
        SSLEngine on
        SSLCertificateFile    /etc/ssl/private/lam.${domain}/fullchain.pem
        SSLCertificateKeyFile /etc/ssl/private/lam.${domain}/key.pem
        SSLProtocol all -SSLv2 -SSLv3
        SSLCipherSuite HIGH:!aNULL:!MD5
        SSLHonorCipherOrder on
        Header set Strict-Transport-Security "max-age=15768000; preload"
        Header always append X-Frame-Options "SAMEORIGIN"
        Header set Content-Security-Policy "default-src 'self' 'unsafe-inline'; script-src 'self' 'unsafe-inline'; img-src 'self'"
        Header always set Referrer-Policy "no-referrer"
        Header set X-XSS-Protection "1; mode=block"
        Header set X-Content-Type-Options "nosniff"
        ErrorLog /var/log/apache2/lam.${domain}_error.log
        CustomLog /var/log/apache2/lam.${domain}.log combined

        $(sed '/^Alias*/s/^/#/g' /etc/apache2/conf-available/ldap-account-manager.conf)

</VirtualHost>
EOF
```
## Module aktivieren und Firewalleinstellungen setzen

```bash
a2enmod ssl headers rewrite
a2ensite lam.${domain}.conf
a2dissite 000-default.conf
ufw allow 80
ufw allow 443
systemctl restart apache2
systemctl status apache2
```

## LDAP-Suchen ausführen
```bash
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b ou=users,$basedn -D "$admindn" -w $adminpwd uid
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b ou=users,$basedn -D "$admindn" -w $adminpwd memberOf
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b ou=users,$basedn -D "$admindn" -w $adminpwd "(memberOf=cn=administration,ou=groups,dc=bm-uni,dc=de)" memberOf
```

# Skript für das Versenden von Briefen

## Vorbereitung

```bash
mkdir -p /opt/bm
cd /opt/bm
apt install python3-pip -f
pip install fpdf
```

## Wrapper-Skript

Damit das auch bei Dir klappt, musst Du in Deiner Nextcloud einen public-Link anlegen. Dieser hat dann das Format `https://${nextcloudurl}/index.php/s/${key}` - diese beiden musst Du hier unten dann ersetzen.

```bash
key="HIERDEINKEY"
nextcloudurl="HIERDEINENEXTCLOUD"
cat <<EOF >/opt/bm/sendLetter
#!/bin/bash
/opt/bm/makeLetter \$1 \$2 \$3 \$4 \$5 \$6
echo "Brief erfolgreich erzeugt."
curl -k -T /tmp/letter.pdf -u "${key}:" -H 'X-Requested-With: XMLHttpRequest' https://${nextcloudurl}/public.php/webdav/letter.pdf
echo "Brief erfolgreich hochgeladen."
EOF
chmod +x /opt/bm/sendLetter
```

## PDF-Erzeugung

Wir nutzen hier das [Modul pyFDPF](https://pyfpdf.readthedocs.io/en/latest/Tutorial/index.html) und erzeugen uns auf Basis der übergebenen Argumente aus dem Wrapper eine kleine PDF, die wir nach `/tmp` wegspeichern, um sie dann im Wrapper per `curl` hochzuladen.

```bash
cat <<EOF >/opt/bm/makeLetter
#!/usr/bin/env python3
import sys
from fpdf import FPDF
pdf = FPDF()
pdf.add_page()
pdf.set_font('Arial', '', 12)
pdf.cell(50, 10, sys.argv[1] + " " + sys.argv[2],0,1)
pdf.cell(0, 3, sys.argv[3],0,1)
pdf.cell(0, 20, "Hallo "+sys.argv[1]+",",0,1)
pdf.cell(0, 3, "Dein Nutzername ist " +sys.argv[4] + " und Dein Passwort "+sys.argv[6]+".")
pdf.output('/tmp/letter.pdf', 'F')
EOF
chmod +x /opt/bm/makeLetter
```

**Hinweis**: Mit FPDF lassen sich erstaunlich komplexe PDF-Dateien erzeugen. Die [Doku der originalen PHP-Klasse](http://www.fpdf.org/) zeigen hier einiges auf. Selbstverständlich könnte man natürlich auch LaTeX-Dokumente erzeugen und dann in eine PDF-Datei übersetzen lassen.
