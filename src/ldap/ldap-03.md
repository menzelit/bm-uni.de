# LDAP 03 | Einrichtung von ACLs

**WICHTIG:** Alle Kommandos werden als root-User ausgeführt!

# Nutzernamen POSIX-compliant machen

```bash
cat <<EOF >/tmp/modify-users.ldif
#uid ändern
dn: uid=s.saeusel,ou=users,$basedn
changetype: modrdn
newrdn: uid=ssaeusel
deleteoldrdn: 1

#home
dn: uid=ssaeusel,ou=users,$basedn
changetype: modify
replace: homeDirectory
homeDirectory: /home/ssauesel

#mail
dn: uid=ssaeusel,ou=users,$basedn
changetype: modify
add: mail
mail: ssauesel@stud.${domain}

#password
dn: uid=ssaeusel,ou=users,$basedn
changetype: modify
delete: userPassword
EOF

cat <<EOF >>/tmp/modify-users.ldif
#uid ändern
dn: uid=n.nieda,ou=users,$basedn
changetype: modrdn
newrdn: uid=nnieda
deleteoldrdn: 1

#mail
dn: uid=nnieda,ou=users,$basedn
changetype: modify
add: mail
mail: nnieda@stud.${domain}

#home
dn: uid=nnieda,ou=users,$basedn
changetype: modify
replace: homeDirectory
homeDirectory: /home/nnieda

#uid ändern
dn: uid=e.erstie,ou=users,$basedn
changetype: modrdn
newrdn: uid=eerstie
deleteoldrdn: 1

#home
dn: uid=eerstie,ou=users,$basedn
changetype: modify
replace: homeDirectory
homeDirectory: /home/eerstie

#mail
dn: uid=eerstie,ou=users,$basedn
changetype: modify
add: mail
mail: eerstie@stud.${domain}
EOF

ldapmodify -x -H "ldaps://ldap.bm-uni.de" -D "$admindn" -w $adminpwd -f /tmp/modify-users.ldif
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b ou=users,$basedn -D "$admindn" -w $adminpwd uid homeDirectory mail userPassword
```

## Weitere organizational units vorbereiten

```bash
cat <<EOF >/tmp/further-units-and-users.ldif
# binduser
dn: ou=binduser,$basedn
ou: binduser
objectClass: top
objectClass: organizationalUnit

# projects
dn: ou=projects,$basedn
ou: projects
objectClass: top
objectClass: organizationalUnit

EOF
```

## Weitere Nutzer anlegen und eintragen
```bash
adamPassword="adam"
ingoPassword="ingo"
wiebkePassword="wiebke"
cat <<EOF >>/tmp/further-units-and-users.ldif
# Adam Assistent
dn: uid=aassistent,ou=users,$basedn
objectClass: top
objectClass: inetorgperson
objectClass: posixAccount
objectClass: organizationalPerson
objectClass: shadowAccount
cn: Adam Assistent
sn: Assistent
givenName: Adam
uid: aassistent
mail: aassistent@${domain}
telephoneNumber: +49 30 23125 - 105
uidNumber: 10003
gidNumber: 10000
homeDirectory: /home/aassistent
loginShell: /bin/bash
userPassword: $(slappasswd -h {SSHA} -s $adamPassword)
shadowLastChange: 0
shadowMax: 0
shadowWarning: 0

# Ingo Ingenieur
dn: uid=iingenieur,ou=users,$basedn
objectClass: top
objectClass: inetorgperson
objectClass: posixAccount
objectClass: organizationalPerson
objectClass: shadowAccount
cn: Prof. Ingo Ingenieur
sn: Ingenieur
givenName: Ingo
uid: iingenieur
mail: iingenieur@${domain}
telephoneNumber: +49 30 23125 - 102
uidNumber: 10005
gidNumber: 10000
homeDirectory: /home/iingenieur
loginShell: /bin/bash
userPassword: $(slappasswd -h {SSHA} -s $ingoPassword)
shadowLastChange: 0
shadowMax: 0
shadowWarning: 0

# Wiebke Wimi
dn: uid=wwimi,ou=users,$basedn
objectClass: top
objectClass: inetorgperson
objectClass: posixAccount
objectClass: organizationalPerson
objectClass: shadowAccount
cn: Wiebke Wimi
sn: Wimi
givenName: Wiebke
uid: wwimi
mail: wwimi@${domain}
telephoneNumber: +49 30 23125 - 212
uidNumber: 10010
gidNumber: 10000
homeDirectory: /home/wwimi
loginShell: /bin/bash
userPassword: $(slappasswd -h {SSHA} -s $wiebkePassword)
shadowLastChange: 0
shadowMax: 0
shadowWarning: 0
EOF

ldapadd -x -H "ldaps://ldap.bm-uni.de" -D "$admindn" -w $adminpwd -f /tmp/further-units-and-users.ldif
ldapsearch -x -H "ldaps://ldap.bm-uni.de" -b $basedn -D "uid=wwimi,ou=users,$basedn" -w $wiebkePassword "(uid=wwimi)"
```

## RFC2307bis einrichten
```bash
wget -P /etc/ldap/schema https://raw.githubusercontent.com/jtyr/rfc2307bis/master/rfc2307bis.schema
sed -i "s/nis.schema/rfc2307bis.schema/" /etc/ldap/slapd.conf
head /etc/ldap/slapd.conf
systemctl restart slapd
systemctl status slapd
```

## Weitere Gruppen anlegen
```bash
cat <<EOF >/tmp/groups.ldif
dn: cn=professors,ou=groups,$basedn
cn: professors
objectClass: top
objectClass: groupOfNames
objectClass: posixGroup
gidNumber: 10001
member: uid=iingenieur,ou=users,$basedn

dn: cn=research-assistants,ou=groups,$basedn
cn: research-assistants
objectClass: top
objectClass: groupOfNames
objectClass: posixGroup
gidNumber: 10002
member: uid=wwimi,ou=users,$basedn

dn: cn=administration,ou=groups,$basedn
cn: administration
objectClass: top
objectClass: groupOfNames
objectClass: posixGroup
gidNumber: 10003
member: uid=aassistent,ou=users,$basedn

dn: cn=students,ou=groups,$basedn
cn: students
objectClass: top
objectClass: groupOfNames
objectClass: posixGroup
gidNumber: 10004
member: uid=eerstie,ou=users,$basedn
member: uid=nnieda,ou=users,$basedn
member: uid=ssaeusel,ou=users,$basedn
EOF

ldapadd -x -H "ldaps://ldap.bm-uni.de" -D "$admindn" -w $adminpwd -f /tmp/groups.ldif
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b ou=groups,$basedn cn gidNumber member
```

## ldapcompare
```bash
ldapcompare -H "ldaps://ldap.bm-uni.de" -D "$admindn" -w $adminpwd "cn=posixGruppe,ou=groups,$basedn" gidNumber:100000
ldapcompare -H "ldaps://ldap.bm-uni.de" -D "$admindn" -w $adminpwd "cn=posixGruppe,ou=groups,$basedn" gidNumber:10000
```

## bisherige ACLs auslagern
```bash
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b ou=users,$basedn -D "uid=wwimi,ou=users,$basedn" -w $wiebkePassword userPassword

tail -n 11 /etc/ldap/slapd.conf

cat <<EOF >/etc/ldap/acl.conf
# Access Control Lists
access to attrs=userPassword,shadowLastChange
    by dn="cn=admin,$basedn" write
    by anonymous auth
    by self write
    by * none
access to *
    by dn="cn=admin,$basedn" write
    by * read
access to dn.base=""
    by * read
EOF
head -n -10 /etc/ldap/slapd.conf > /etc/ldap/slapd.tmp.conf && mv /etc/ldap/slapd.tmp.conf /etc/ldap/slapd.conf
cat <<EOF >>/etc/ldap/slapd.conf
include         /etc/ldap/acl.conf
EOF
systemctl restart slapd
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b ou=users,$basedn -D "uid=wwimi,ou=users,$basedn" -w $wiebkePassword userPassword
```

## dirty test: ACLs vertauschen
Einmal hin...
```bash
cat <<EOF >/etc/ldap/acl.conf
# Access Control Lists
access to *
    by dn="cn=admin,$basedn" write
    by * read
access to attrs=userPassword,shadowLastChange
    by dn="cn=admin,$basedn" write
    by anonymous auth
    by self write
    by * none
access to dn.base=""
    by * read
EOF
systemctl restart slapd
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b ou=users,$basedn -D "uid=wwimi,ou=users,$basedn" -w $wiebkePassword userPassword
```

... einmal her:
```bash
cat <<EOF >/etc/ldap/acl.conf
# Access Control Lists
access to attrs=userPassword,shadowLastChange
    by dn="cn=admin,$basedn" write
    by anonymous auth
    by self write
    by * none
access to *
    by dn="cn=admin,$basedn" write
    by * read
access to dn.base=""
    by * read
EOF
systemctl restart slapd
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b ou=users,$basedn -D "uid=wwimi,ou=users,$basedn" -w $wiebkePassword userPassword
```

## unsere ACLs aufbauen und testen
```bash

cat <<EOF >/etc/ldap/acl.conf
access to dn.base="$basedn" by * read

access to attrs=userPassword,shadowLastChange
        by anonymous auth
        by self write
        by group.exact="cn=administration,ou=groups,$basedn" write
        by * none

access to dn.subtree="ou=binduser,$basedn"
        by group.exact="cn=administration,ou=groups,$basedn" write
        by * none

access to dn.subtree="ou=groups,$basedn"
        by group.exact="cn=administration,ou=groups,$basedn" write
        by group.exact="cn=professors,ou=groups,$basedn" write
        by dn.one="ou=binduser,$basedn" read
        by * none

access to dn.subtree="ou=users,$basedn"
        by group.exact="cn=administration,ou=groups,$basedn" write
        by group.exact="cn=professors,ou=groups,$basedn" write
        by dn.one="ou=binduser,$basedn" read
        by self read
        by * none

access to * by * none
EOF
cat /etc/ldap/acl.conf
systemctl restart slapd
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b $basedn
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b $basedn -D "uid=wwimi,ou=users,$basedn" -w $wiebkePassword
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b $basedn -D "uid=aassistent,ou=users,$basedn" -w $adamPassword dn
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b $basedn -D "uid=iingenieur,ou=users,$basedn" -w $ingoPassword dn
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b "ou=users,$basedn" -D "uid=iingenieur,ou=users,$basedn" -w $ingoPassword dn userPassword
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b "ou=users,$basedn" -D "uid=aassistent,ou=users,$basedn" -w $adamPassword dn userPassword
```

## Testbinder einfügen
```bash
testbinderPassword="testbinder"
cat <<EOF >/tmp/binder.ldif
# Testbinder
dn: cn=testbinder,ou=binduser,$basedn
objectClass: organizationalRole
objectClass: simpleSecurityObject
cn: testbinder
userPassword: $(slappasswd -h {SSHA} -s $testbinderPassword)
description: "Demo-Binder, angelegt am $(date)"
EOF

ldapadd -x -H "ldaps://ldap.bm-uni.de" -D "uid=aassistent,ou=users,$basedn" -w $adamPassword -f /tmp/binder.ldif
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b "ou=binduser,$basedn" -D "uid=aassistent,ou=users,$basedn" -w $adamPassword
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b $basedn -D "cn=testbinder,ou=binduser,$basedn" -w $testbinderPassword dn userPassword
```

## Wiebe wird Admin
```bash
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b $basedn -D "uid=wwimi,ou=users,$basedn" -w $wiebkePassword dn userPassword

cat <<EOF >/tmp/wiebke-admin.ldif
# Wiebke soll Admin werden
dn: cn=administration,ou=groups,$basedn
changetype: modify
add: member
member: uid=wwimi,ou=users,$basedn

EOF
cat /tmp/wiebke-admin.ldif

ldapmodify -x -H "ldaps://ldap.bm-uni.de" -D "uid=iingenieur,ou=users,$basedn" -w $ingoPassword -f /tmp/wiebke-admin.ldif
ldapsearch -x -LLL -H "ldaps://ldap.bm-uni.de" -b $basedn -D "uid=wwimi,ou=users,$basedn" -w $wiebkePassword dn userPassword
```
