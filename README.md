# knoxsso
KnoxSSO with FreeIPA

Step 1: Install FreeIPA Server and HDP 3.1.4 on Ambari 2.7.4.0

Follow below for the FreeIPA:

OS Image: ami-centos-7.4-1.11.5-00-1543261671 (ami-012f8fcc06515f35c)

Prepare OS base scripts

```bash
echo "***************** unzip and wget"
yum install -y wget
yum install -y mlocate
updatedb

cp -p /etc/security/limits.conf /etc/security/limits.conf.ORIG

sed -i '$ a *    soft    nofile 65536' /etc/security/limits.conf
sed -i '$ a *    hard    nofile 65536' /etc/security/limits.conf
sed -i '$ a *    soft    nproc 65536' /etc/security/limits.conf
sed -i '$ a *    hard    nproc 65536' /etc/security/limits.conf

echo ******************" Installing ntp"
yum install -y ntp
systemctl enable ntpd
systemctl start ntpd

service ntpd start


echo "***************** Check if NTP service is active:"
systemctl status ntpd

service ntpd status

echo "***** STOP IP Tables (firewalls)"
yum install -y iptables-services
systemctl start iptables
systemctl stop iptables
systemctl status iptables


echo "***************** Disable SELinux running the following:"
setenforce 0

echo "*****************  Make a backup of the config file located in /etc/selinux"
cd /etc/selinux
cp -p config config.ORIG

sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# Change the behaviour in the creation of new files
umask 0022
echo umask 0022 >> /etc/profile

echo "************** restart"
shutdown -r now
```

https://www.golinuxcloud.com/configure-setup-freeipa-server-client-linux/

On FreeIPA Server:
```bash
yum install ipa-server  -y

ipa-server-install
```

Capture the below information

```bash
###INFO###
#The IPA Master Server will be configured with:
#Hostname:       ip-172-31-21-109.us-west-1.compute.internal
#IP address(es): 172.31.21.109
#Domain name:    us-west-1.compute.internal
#Realm name:     US-WEST-1.COMPUTE.INTERNAL

```

On FreeIPA Client (Ambari / HDP nodes)

```bash
yum install ipa-client -y
ipa-client-install
```

Update below on your local /etc/hosts

```bash
13.57.215.149 ip-172-31-21-109.us-west-1.compute.internal
```

Create users pchalla, ldapbind by opening http://13.57.215.149/ipa/ui

and make sure to give admin access to ldapbind user

run the below to check the connectivity

```bash
yum install openldap*

ldapsearch -x -h ldap://ip-172-31-21-109.us-west-1.compute.internal -p 389 -b cn=users,cn=accounts,dc=us-west-1,dc=compute,dc=internal uid=pchalla
```

Setup LDAP on Ambari

```bash
[root@ip-172-31-26-231 centos]# ambari-server setup-ldap
Using python  /usr/bin/python
Enter Ambari Admin login: admin
Enter Ambari Admin password:

Fetching LDAP configuration from DB.
Primary LDAP Host (ip-172-31-21-109.us-west-1.compute.internal):
Primary LDAP Port (636): 389
Secondary LDAP Host <Optional>:
Secondary LDAP Port <Optional>:
Use SSL [true/false] (false):
User object class (posixAccount):
User ID attribute (uid):
Group object class (posixGroup):
Group name attribute (cn):
Group member attribute (member):
Distinguished name attribute (dn):
Search Base (cn=accounts,dc=us-west-1,dc=compute,dc=internal): cn=users,cn=accounts,dc=us-west-1,dc=compute,dc=internal
Referral method [follow/ignore] (follow):
Bind anonymously [true/false] (false):
Bind DN (uid=ldapbind,cn=users,cn=accounts,dc=us-west-1,dc=compute,dc=internal):
Enter Bind DN Password:
Confirm Bind DN Password:
Handling behavior for username collisions [convert/skip] for LDAP sync (skip):
```

Install Knox and update Advanced Topology with below entries

ref: https://docs.cloudera.com/HDPDocuments/HDP3/HDP-3.1.4/configuring-knox-sso/content/sso_set_up_knox_sso.html

```bash
<param>
    <name>main.ldapRealm.userDnTemplate</name>
    <value>uid={0},cn=users,cn=accounts,dc=us-west-1,dc=compute,dc=internal</value>
</param>
<param>
    <name>main.ldapRealm.contextFactory.url</name>
    <value>ldap://ip-172-31-21-109.us-west-1.compute.internal:389</value>
</param>

<role>AMBARI</role>
   <url>http://ip-172-31-26-231.us-west-1.compute.internal:8080</url>
</service>

<service>
   <role>AMBARIUI</role>
   <url>http://ip-172-31-26-231.us-west-1.compute.internal:8080</url>
</service>

<service>
<role>ZEPPELINUI</role>
 <url>http://ip-172-31-26-231.us-west-1.compute.internal:9995</url>
 </service>

 <service>
 <role>ZEPPELINWS</role>
  <url>ws://ip-172-31-26-231.us-west-1.compute.internal:9995/ws</url>
  </service>
```

Update Advanced knoxsso-topology

```bash
 <param>
    <name>main.ldapRealm.userDnTemplate</name>
    <value>uid={0},cn=users,cn=accounts,dc=us-west-1,dc=compute,dc=internal</value>
</param>
<param>
    <name>main.ldapRealm.contextFactory.url</name>
    <value>ldap://ip-172-31-21-109.us-west-1.compute.internal:389</value>
</param>

<param>
   <name>knoxsso.redirect.whitelist.regex</name>
    <value>.*</value>
</param>
```

Create Certificate for KnoxSSO

```bash
[root@ip-172-31-26-231 bin]# ./knoxcli.sh export-cert --type PEM
Certificate gateway-identity has been successfully exported to: /usr/hdp/3.1.4.0-315/knox/data/security/keystores/gateway-identity.pem
[root@ip-172-31-26-231 bin]#
[root@ip-172-31-26-231 bin]# cat /usr/hdp/3.1.4.0-315/knox/data/security/keystores/gateway-identity.pem
-----BEGIN CERTIFICATE-----
MIICfzCCAeigAwIBAgIIegFKXByWqk8wDQYJKoZIhvcNAQEFBQAwgYExCzAJBgNVBAYTAlVTMQ0w
CwYDVQQIEwRUZXN0MQ0wCwYDVQQHEwRUZXN0MQ8wDQYDVQQKEwZIYWRvb3AxDTALBgNVBAsTBFRl
c3QxNDAyBgNVBAMTK2lwLTE3Mi0zMS0yNi0yMzEudXMtd2VzdC0xLmNvbXB1dGUuaW50ZXJuYWww
HhcNMTkxMDI0MTYzMzIyWhcNMjAxMDIzMTYzMzIyWjCBgTELMAkGA1UEBhMCVVMxDTALBgNVBAgT
BFRlc3QxDTALBgNVBAcTBFRlc3QxDzANBgNVBAoTBkhhZG9vcDENMAsGA1UECxMEVGVzdDE0MDIG
A1UEAxMraXAtMTcyLTMxLTI2LTIzMS51cy13ZXN0LTEuY29tcHV0ZS5pbnRlcm5hbDCBnzANBgkq
hkiG9w0BAQEFAAOBjQAwgYkCgYEAumzntQKpAreRdzJx5bZWf6aht7DL8Om8QK8zbY5uZ+DV+LVS
1P6fvjd9MIK0P4ELd5ylt3TBrIGBczBjR4evgYpbIrZ1liC3YafT+TpaiX+vH9WwB8CX8lom/tPu
Jrwp8x5pPS6UoguVwZf3R8fz2FBbjaM2k1WY3dwurA0zrisCAwEAATANBgkqhkiG9w0BAQUFAAOB
gQAslCYprF6wd1anAaKRRfBdNbvIfpYnBX/gGfJ70MbWV6HkgiWBaeNWMs2g4c2CQlT6lxQANXYW
J9wdwcNzI/T8SJb8vMzmrDMowjzkwAFy6lJioRpFpKX1ZBgQBppw1AT3Yb4nTs4DBzya4b++T7ry
7iOJZegzVvfw176ONNvnSA==
-----END CERTIFICATE-----
```

run the sso setup on Ambari

```bash
[root@ip-172-31-26-231 bin]# ambari-server setup-sso
Using python  /usr/bin/python
Setting up SSO authentication properties...
Enter Ambari Admin login: admin
Enter Ambari Admin password:

SSO is currently not configured
Do you want to configure SSO authentication [y/n] (y)?
Provider URL (https://knox.example.com:8443/gateway/knoxsso/api/v1/websso): https://ip-172-31-26-231.us-west-1.compute.internal:8443/gateway/knoxsso/api/v1/websso
Public Certificate PEM (empty line to finish input):
MIICfzCCAeigAwIBAgIIegFKXByWqk8wDQYJKoZIhvcNAQEFBQAwgYExCzAJBgNVBAYTAlVTMQ0w
CwYDVQQIEwRUZXN0MQ0wCwYDVQQHEwRUZXN0MQ8wDQYDVQQKEwZIYWRvb3AxDTALBgNVBAsTBFRl
c3QxNDAyBgNVBAMTK2lwLTE3Mi0zMS0yNi0yMzEudXMtd2VzdC0xLmNvbXB1dGUuaW50ZXJuYWww
HhcNMTkxMDI0MTYzMzIyWhcNMjAxMDIzMTYzMzIyWjCBgTELMAkGA1UEBhMCVVMxDTALBgNVBAgT
BFRlc3QxDTALBgNVBAcTBFRlc3QxDzANBgNVBAoTBkhhZG9vcDENMAsGA1UECxMEVGVzdDE0MDIG
A1UEAxMraXAtMTcyLTMxLTI2LTIzMS51cy13ZXN0LTEuY29tcHV0ZS5pbnRlcm5hbDCBnzANBgkq
hkiG9w0BAQEFAAOBjQAwgYkCgYEAumzntQKpAreRdzJx5bZWf6aht7DL8Om8QK8zbY5uZ+DV+LVS
1P6fvjd9MIK0P4ELd5ylt3TBrIGBczBjR4evgYpbIrZ1liC3YafT+TpaiX+vH9WwB8CX8lom/tPu
Jrwp8x5pPS6UoguVwZf3R8fz2FBbjaM2k1WY3dwurA0zrisCAwEAATANBgkqhkiG9w0BAQUFAAOB
gQAslCYprF6wd1anAaKRRfBdNbvIfpYnBX/gGfJ70MbWV6HkgiWBaeNWMs2g4c2CQlT6lxQANXYW
J9wdwcNzI/T8SJb8vMzmrDMowjzkwAFy6lJioRpFpKX1ZBgQBppw1AT3Yb4nTs4DBzya4b++T7ry
7iOJZegzVvfw176ONNvnSA==

Use SSO for Ambari [y/n] (n)? y
Manage SSO configurations for eligible services [y/n] (n)? y
 Use SSO for all services [y/n] (n)? y
JWT Cookie name (hadoop-jwt):
JWT audiences list (comma-separated), empty for any ():

```

Open Ambari with knox url - it should redirect to knoxsso page

https://ip-172-31-26-231.us-west-1.compute.internal:8443/gateway/default/ambari/

