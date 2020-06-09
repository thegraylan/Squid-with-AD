# Squid-with-AD
Squid + kerberos + Active directory + RedHat This is a guide on deploying Squid on the RedHat 7 server, integrated into widows Active directory using a service account that authenticates using kerberos


Windows Server
Name: ServerUS01
Domain name: THEGRAYLAN.LOCAL
IP: 10.0.0.4
Squid Service account:  sa-squid
Admin account: sysadmin


Redhat server
Name: squid
IP: 10.0.0.5
DNS alias: proxy.thegraylan.local


Joining the redhat server to the domain
1. sudo yum update -y
2. sudo hostnamectl set-hostname squid.thegraylan.local
3 .sudo yum install -y sssd realmd krb5-workstation samba samba-common-tools oddjob oddjob-mkhomedir adcli

4. edit krb5.conf and add the following 
#sudo vim /etc/krb5.conf  
[libdefaults]
dns_lookup_realm = true
dns_lookup_kdc = true
rdns = false

5. Check the domain is available 
#realm discover thegraylan.local
6. Modify your sssd.conf to allow abbreviated usernames
#sudo realm join --verbose --user=sysadmin thegraylan.local
#sudo sed -i 's/use_fully_qualified_names .*/use_fully_qualified_names = False/g' /etc/sssd/sssd.conf
#sudo sed -i 's/fallback_homedir .*/fallback_homedir = \/home\/%u/g' /etc/sssd/sssd.conf
#sudo systemctl restart sssd 
7. Add a thegraylan_sudoers files to allow THEGRAYLAN domain users the ability to sudo
#echo "%Administrators@thegraylan.local ALL=(ALL:ALL) ALL" | sudo tee /etc/sudoers.d/thegraylan_sudoers

Create KEYTAB file for squid service account
Start the creation of a keytab file for squid-user. To do so, execute the command from the windows server:
#C:\Windows\system32\ktpass.exe /princ HTTP/squid.thegraylan.local@THEGRAYLAN.LOCAL /mapuser sa-squid@THEGRAYLAN>LOCAL /crypto AES256-SHA1  /ptype KRB5_NT_PRINCIPAL /pass PASSWORD /out squid.keytab


Install and configure Squid
1. #sudo yum install squid krb5-workstation
2. copy the squid.keytab file to /etc/squid/ folder on the squid server  and set squid to be the owner
#chown squid /etc/squid/squid.keytab
3. Edit the /etc/squid/squid.conf file
#sudo vim /etc/squid/squid.conf
auth_param negotiate program /usr/lib64/squid/negotiate_kerberos_auth -k /etc/squid/squid.keytab -s HTTP/squid.thegraylan.local@THEGRAYLAN.LOCAL
acl kerb-auth proxy_auth REQUIRED
http_access allow kerb-auth


4. edit the squid systemd start up file and add the following
#sudo vim vim /etc/sysconfig/squid
KRB5_KTNAME=/etc/squid/squid.keytab
export KRB5_KTNAME

4. Open the 3128 port in the firewall:
# sudo firewall-cmd --permanent --add-port=3128/tcp
# sudo  firewall-cmd --reload
5. Start the squid service:
# sudo systemctl start squid
6. Enable the squid service to start automatically when the system boots:
# sudo systemctl enable squid




