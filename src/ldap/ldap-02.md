# LDAP 02 | Einrichtung des Servers

**WICHTIG:** Alle Kommandos werden als root-User ausgeführt!

## Optionale Vorbereitung

Das war im Video bereits passiert.

```bash
ufw allow ssh
ufw enable
```

## Einrichtung des Hostnames

```bash
hostname="ldap.bm-uni.de"
hostnameShort=$(echo $hostname | cut -d "." -f1)
hostnamectl set-hostname $hostnameShort
sed -i "s/^127.0.1.1.*/127.0.1.1 $hostname $hostnameShort/g" /etc/hosts
bash
cat /etc/hosts
```

## Setzen der Variablen

```bash
basedn="dc=bm-uni,dc=de"
admindn="cn=admin,$basedn"
adminpwd="hpctech1!"
echo -e "\$basedn:\t$basedn\n\$admindn:\t$admindn\n\$adminpwd:\t$adminpwd"
```

## Installation
```bash
apt install slapd
```

## Umzug auf statische Konfiguration
```bash
systemctl stop slapd
sed -i "s/SLAPD_CONF=.*/SLAPD_CONF=\/etc\/ldap\/slapd.conf/g" /etc/default/slapd
cat /etc/default/slapd
rm /var/lib/ldap/*.mdb
```

## Initiale Konfiguration
```bash
adminpwdHash=$(slappasswd -h {SSHA} -s $adminpwd)

cat <<EOF >/etc/ldap/slapd.conf
# Schemata und Objektklassen
include         /etc/ldap/schema/core.schema
include         /etc/ldap/schema/cosine.schema
include         /etc/ldap/schema/inetorgperson.schema
include         /etc/ldap/schema/nis.schema
# Loglevel - 256 ist ein guter Mittelwert
# NICHT auf 0 setzen, da sonst gar nichts geloggt wird!
# https://www.openldap.org/doc/admin24/slapdconfig.html
loglevel        256
pidfile         /var/run/slapd/slapd.pid
argsfile        /var/run/slapd/slapd.args
modulepath      /usr/lib/ldap
moduleload      back_mdb
# Maximal 1000 Werte bei einer Suche zurück geben
sizelimit 1000
# Anzahl CPUs, die für das Indexing verwendet werden
tool-threads 2
#######################################################################
# Datenbank Nummer 1
database        mdb
# Der Basis-DN
suffix          "$basedn"
# Root-User
rootdn          "$admindn"
rootpw          "$adminpwdHash"
# Ablageort der Datenbank
directory       "/var/lib/ldap"
# Indices
index           objectClass eq
# Letzte Modifikation der Datenbank schreiben
lastmod         on
# Access Control Lists
access to attrs=userPassword,shadowLastChange
    by dn="$admindn" write
    by anonymous auth
    by self write
    by * none
access to *
    by dn="$admindn" write
    by * read
access to dn.base=""
    by * read
EOF
less /etc/ldap/slapd.conf
systemctl start slapd
systemctl status slapd
```

## Erste Daten
```bash
ldapsearch -x -D "$admindn" -b $basedn -w $adminpwd

cat <<EOF >/tmp/groups-and-users.ldif
# Base erstellen
dn: $basedn
objectClass: dcObject
objectClass: organization
o: fiktive Baron Münchhausen Universität
dc: bm-uni

# Gruppen erstellen
dn: ou=groups,$basedn
ou: groups
objectClass: top
objectClass: organizationalUnit

# User erstellen
dn: ou=users,$basedn
ou: users
objectClass: top
objectClass: organizationalUnit
EOF
cat /tmp/groups-and-users.ldif

ldapadd -x -D "$admindn" -w $adminpwd -f /tmp/groups-and-users.ldif
ldapsearch -x -D "$admindn" -b $basedn -H "ldap://ldap.bm-uni.de" -w $adminpwd
echo "ZmlrdGl2ZSBCYXJvbiBNw7xuY2hoYXVzZW4gVW5pdmVyc2l0w6R0\n" | base64 -d

cat <<EOF >/tmp/posixGruppe-und-studis.ldif
# posixGruppe anlegen
dn: cn=posixGruppe,ou=groups,$basedn
cn: posixGruppe
objectClass: top
objectClass: posixGroup
gidNumber: 10000
EOF

erwinPassword=$(slappasswd -h {SSHA} -s "3rw1n3rst13")
noraPassword=$(slappasswd -h {SSHA} -s "Zi%iY8hua4coe0ain8sho!p7")
siegfriedPassword=$(slappasswd -h {SSHA} -s "daStehIchNunIchArmerToR!")
cat <<EOF >>/tmp/posixGruppe-und-studis.ldif
# Erwin Erstie anlegen
dn: uid=e.erstie,ou=users,$basedn
objectClass: top
objectClass: inetorgperson
objectClass: posixAccount
objectClass: shadowAccount
cn: Erwin Erstie
sn: Erstie
uid: e.erstie
uidNumber: 10000
gidNumber: 10000
homeDirectory: /home/e.erstie
loginShell: /bin/bash
userPassword: $erwinPassword
shadowLastChange: 0
shadowMax: 0
shadowWarning: 0

# Nora Nieda anlegen
dn: uid=n.nieda,ou=users,$basedn
objectClass: top
objectClass: inetorgperson
objectClass: posixAccount
objectClass: shadowAccount
cn: Nora Nieda
sn: Nieda
uid: n.nieda
uidNumber: 10001
gidNumber: 10000
homeDirectory: /home/n.nieda
loginShell: /bin/zsh
userPassword: $noraPassword
shadowLastChange: 0
shadowMax: 0
shadowWarning: 0

# Siegfried von Säusel anlegen
dn: uid=s.saeusel,ou=users,$basedn
objectClass: top
objectClass: inetorgperson
objectClass: posixAccount
objectClass: shadowAccount
cn: Siegfried von Säusel
sn: Siegfried
uid: s.saeusel
uidNumber: 10002
gidNumber: 10000
homeDirectory: /home/s.saeusel
loginShell: /bin/bash
userPassword: $siegfriedPassword
shadowLastChange: 0
shadowMax: 0
shadowWarning: 0
EOF

ldapadd -x -H "ldap://ldap.bm-uni.de" -D "$admindn" -w $adminpwd -f /tmp/posixGruppe-und-studis.ldif
```

## Suchen

```bash
ldapsearch -x -H "ldap://ldap.bm-uni.de" -b $basedn -D "uid=e.erstie,ou=users,$basedn" -W
ldapsearch -x -H "ldap://ldap.bm-uni.de" -b $basedn -D "$admindn" -w $adminpwd "(uid=e.erstie)" uidNumber
ldapsearch -x -LLL -H "ldap://ldap.bm-uni.de" -b $basedn -D "$admindn" -w $adminpwd "(uid=e.erstie)" uidNumber
ldapsearch -x -LLL -H "ldap://ldap.bm-uni.de" -b $basedn -D "$admindn" -w $adminpwd "(cn=*Erstie*)" uid cn
ldapsearch -x -LLL -H "ldap://ldap.bm-uni.de" -b $basedn -D "$admindn" -w $adminpwd "(gidNumber=10000)" uid loginShell
ldapsearch -x -LLL -H "ldap://ldap.bm-uni.de" -b $basedn -D "$admindn" -w $adminpwd "(&(gidNumber=10000)(loginShell=/bin/zsh))" uid cn
```

## LDAPS

**Hinweis:** Keys müsst Ihr selber besorgen!

```bash
ll /etc/ldap/*.pem
chown openldap /etc/ldap/ldapkey.pem
chmod 600 /etc/ldap/ldapkey.pem
ll /etc/ldap/*.pem

sed -i '/^tool-threads*/a TLSCertificateKeyFile \/etc\/ldap\/ldapkey.pem' /etc/ldap/slapd.conf
sed -i '/^tool-threads*/a TLSCertificateFile \/etc\/ldap\/ldapcert.pem' /etc/ldap/slapd.conf
sed -i '/^tool-threads*/a # SSL Certs' /etc/ldap/slapd.conf

cat /etc/ldap/slapd.conf

sed -i 's/SLAPD_SERVICES=.*/SLAPD_SERVICES="ldapi:\/\/\/ ldaps:\/\/\/"/g' /etc/default/slapd
systemctl restart slapd
ss -tlpn

ldapsearch -x -LLL -H "ldap://ldap.bm-uni.de" -b $basedn -D "$admindn" -w $adminpwd "(cn=*Erstie*)" uid cn
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b $basedn -D "$admindn" -w $adminpwd "(cn=*Erstie*)" uid cn
```

## Firewall einstellen

```bash
ufw allow ldaps
ufw deny ldap
ufw status
```
