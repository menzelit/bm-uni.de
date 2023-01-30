# LDAP 06 | Ubuntu anbinden

# Erste Abfragen

```bash
basedn=""
admindn="cn=admin,$basedn"
adminpwd=""
domain=""
ldapsearch -x -D "$admindn" -b $basedn -w $adminpwd -H ldaps://ldap.${domain} displayName telephoneNumber
ldapsearch -x -D "$admindn" -b $basedn -w $adminpwd -H ldaps://ldap.${domain} "(objectClass=inetOrgPerson)" displayName telephoneNumber
ldapsearch -x -D "$admindn" -b $basedn -w $adminpwd -H ldaps://ldap.${domain} "(&(objectClass=inetOrgPerson)(telephoneNumber=*))" displayName telephoneNumber
ldapsearch -x -D "$admindn" -b $basedn -w $adminpwd -H ldaps://ldap.${domain} "(&(objectClass=inetOrgPerson)(telephoneNumber=*23125212))" displayName telephoneNumber
cat /etc/ldap/schema/core.schema | grep "2.5.4.20" -A5 -B1
```

# Auf dem Laptop

## Ersteinrichtung
```bash
sudo su
basedn=""
admindn="cn=admin,$basedn"
adminpwd=""
domain=""
apt install sddm-theme-maldives sssd -y

cat<< EOF >/etc/sddm.conf
[Autologin]
Relogin=false
Session=
User=

[General]
HaltCommand=/bin/systemctl poweroff
InputMethod=compose
Numlock=none
RebootCommand=/bin/systemctl reboot

[Theme]
Current=maldives
CursorTheme=
DisableAvatarsThreshold=7
EnableAvatars=true
FacesDir=/usr/share/sddm/faces
ThemeDir=/usr/share/sddm/themes

[Users]
DefaultPath=/bin:/usr/bin
HideShells=
HideUsers=
MaximumUid=60000
MinimumUid=1000
RememberLastSession=true
RememberLastUser=true
ReuseSession=false

[Wayland]
EnableHiDPI=false
SessionCommand=/usr/share/sddm/scripts/wayland-session
SessionDir=/usr/share/wayland-sessions
SessionLogFile=.local/share/sddm/wayland-session.log

[X11]
DisplayCommand=/usr/share/sddm/scripts/Xsetup
DisplayStopCommand=/usr/share/sddm/scripts/Xstop
EnableHiDPI=false
MinimumVT=1
ServerArguments=-nolisten tcp
ServerPath=/usr/bin/X
SessionCommand=/etc/sddm/Xsession
SessionDir=/usr/share/xsessions
SessionLogFile=.local/share/sddm/xorg-session.log
UserAuthFile=.Xauthority
XauthPath=/usr/bin/xauth
XephyrPath=/usr/bin/Xephyr
EOF

cat <<EOF >/etc/sssd/sssd.conf
[sssd]
config_file_version = 2
services = nss, pam
domains = bm-uni.de

[nss]
[pam]
offline_credentials_expiration = 7

[domain/bm-uni.de]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldaps://ldap.${domain}
ldap_schema = rfc2307bis
ldap_default_bind_dn = uid=pc-binder,ou=binduser,${basedn}
ldap_default_authtok = fmE-HiMyrXbJ
ldap_default_authtok_type = password
ldap_search_base = ${basedn}
ldap_user_search_base = ou=users,${basedn}
ldap_user_object_class = inetOrgPerson
ldap_user_name = uid
ldap_group_name = cn
ldap_user_gecos = gecos
ldap_group_search_base = ou=groups,${basedn}
ldap_group_object_class = posixgroup
cache_credentials = True
account_cache_expiration = 7
enumerate = false
EOF

chown root:root /etc/sssd/sssd.conf
chmod 600 /etc/sssd/sssd.conf
pam-auth-update --enable mkhomedir

systemctl start sssd
systemctl enable sssd
getent passwd
getent passwd eerstie
```

## Adminrechte

```bash
echo "%administration ALL=(ALL:ALL) ALL" > /etc/sudoers.d/administration
chmod 0440 /etc/sudoers.d/administration
```

## Zugangsbeschränkung

```bash
cat <<EOF >>/etc/sssd/sssd.conf
access_provider = simple
simple_allow_groups = professors,administration
EOF
systemctl restart sssd
```
