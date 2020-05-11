This is the guideline for installing new Replica for an existing IdM Server
Master
===================================================================================
0- The Replica and the master server must be running the same version of IdM
and RedHat

1- edit /etc/hosts: records for both Master and new Replica on both nodes:
$ vi /etc/hosts

2- Check if there is connectivity between these nodes (Master & new Replica)
3- Check if all the required ports on IdM Master is open and you can "telnet"
to these ports from new Replica. 
	Here is the list of ports and services:
	HTTP/HTTPS	80, 443	TCP
	LDAP/LDAPS	389, 636	TCP			(Maybe LDAPS
is not configured on the Server, so only 389 is the listening port on the
master)
	Kerberos	88, 464	TCP and UDP		(Maybe port 464 is not
configured on the server) 
	DNS	53	TCP and UDP
	NTP	123	UDP

###################################################################################################	
#		Maybe you should check firewalld and set the followings:
#
#		$ firewall-cmd --permanent
--add-service={ntp,http,https,ldap,ldaps,kerberos,kpasswd,dns} #
#		$ firewall-cmd --reload
#
###################################################################################################

4- Installing the required packages:
$ yum update
$ yum install ipa-client ipa-server bind bind-dyndb-ldap

5- Before executing Replica Installation command, it is recommended to join it
to the Domain as client using ipa-client:
$ ipa-client-install

# The wizard will ask you Domain name and ipa server full FQDN. Please note
that you may receive some error  regarding dns records, but if you have
already added the records to /etc/hosts, just ignore it.

# If you have already tried to join to the domain and you have received
errors, you may add --force-join to the command:
$ ipa-client-install --force-join

6- In IdM Master Server, add the new Replica Host as hostgroup member
adding the client to the "ipaservers" Host Group:
$ ipa hostgroup-add-member

# This can be done via IdM UI: Identity -> Groups -> Host Groups -> ipaservers
-> add ...

7- Now it's time to install new Replica

$ kinit admin
# Enter the password of IdM admin
$ ipa-replica-install --setup-ca --setup-dns --setup-kra --principal admin
--admin-password "admin_password"
# Based on your requirements you can add server role to the new replica during
installation by adding: --setup-ca, --setup-dns, --setup-kra
# If you are sure that all master server ports are open and you have not any
connectivity issues, you can add --skip-conncheck to the command to bypass
connection check.

8- If the installation is successfully done, you are able to run the command
and check the results:
$ ipa-replica-manage list
#displays the list of servers (master and replicas)

9- If you have encountered errors during the installation, you may check the
errors in /var/log/ipa-replica-install.log, resolve the issue, then:
uninstall and clean-up the installation using the following command:
$ ipa-server-install --uninstall
And start from ZERO!

===================================================================================
Installing Replica server roles after successful installation of Replica

$ sudo ipa-ca-install (Requires Directory password)
$ sudo ipa-dns-install (Requires Directory password)
$ sudo ipa-kra-install (Requires Directory password)

===================================================================================
Manage Replicas:

To list the servers and replicas:
$ ipa-replica-manage list

To list the replication agreements, add the replica's host name to
ipa-replica-manage list :
$ ipa-replica-manage list new_server.example.com

Creating and Removing Replication Agreements:
$ ipa-replica-manage connect server1.example.com server2.example.com
$ ipa-replica-manage disconnect server1.example.com server4.example.com

To remove all replication agreements and data related to a replica, use the
ipa-replica-manage del:
$ ipa-replica-manage del server2.example.com

To manually initiate a replication update, use the ipa-replica-manage
force-sync:
$ ipa-replica-manage force-sync --from server1.example.com

===================================================================================
Promoting Replica to CA Master Server:

1- Check which server handles certificate renewal currently
$ ipa config-show | grep "CA renewal master"

2- Make sure that CA ins installed on the Replica which you want to promote
it.

Changing the Server handles certificate renewal:
$ ipa config-mod --ca-renewal-master-server new_server.example.com

3- Check which server is generating CRLs:
you should run this command on all servers(replicas) to see which one is
enabled:
$ ipa-crlgen-manage status

4- On the current CRL generation master, disable the feature:
$ ipa-crlgen-manage disable

5- On the other CA host that you want to configure as the new CRL generation
master, enable the feature:
$ ipa-crlgen-manage enable

6- Make sure the /var/lib/ipa/pki-ca/publish/MasterCRL.bin file exists on the
new master CA server.

# Some management operations could be done via IdM UI. 
# You can also see the Topology graph and edit the Replication Agreements
(add/delete) by drawing a line between the servers.


###################################################################################################	
#
Known Issues
#
###################################################################################################

===================================================================================
Issue 1 - Add new user error:

Description:
You may encounter errors while creating new users on new Replica master, when
the old master is down.

Cause: 
Because the old master was the only node which was assigning ID. ID range is
not automatically retrieved when a replica dies and needs to be deleted, which
means the ID range previously assigned to the replica becomes unavailable. 

Resolution:
We should manually set ID range, ID range extension according to the below
explanations and commands:

ipa-replica-manage dnarange-set allows you to define the current ID range for
a specified server:
$ ipa-replica-manage dnarange-set masterA.example.com 1250-1499

ipa-replica-manage dnanextrange-set allows you to define the next ID range for
a specified server:
$ ipa-replica-manage dnanextrange-set masterB.example.com 1001-5000

Important:
Be careful not to create overlapping ID ranges. If any of the ID ranges you
assign to servers or replicas overlap, it could result in two different
servers assigning the same ID value to different entries. Do not set ID ranges
that include UID values of 1000 and lower; these values are reserved for
system use. Also, do not set an ID range that would include the 0 value; the
SSSD service does not handle the 0 ID value.

===================================================================================
Issue 2 - Error while installing CA Server Role on Replica.


Problem:

  [3/27]: creating ACIs for admin
  [4/27]: creating installation admin user
  [5/27]: configuring certificate server instance
ipaserver.install.dogtaginstance: CRITICAL Failed to configure CA instance:
Command '/usr/sbin/pkispawn -s CA -f /tmp/tmp84nCYk' returned non-zero exit
status 1
ipaserver.install.dogtaginstance: CRITICAL See the installation logs and the
following files/directories for more information:
ipaserver.install.dogtaginstance: CRITICAL   /var/log/pki/pki-tomcat
  [error] RuntimeError: CA configuration failed.

Your system may be partly configured.
Run /usr/sbin/ipa-server-install --uninstall to clean up.

CA configuration failed.

Cause: Because of unsuccessful install attemps which was already made on this
server (due to other reasons) and it has not been cleaned up with this
command:
$ ipa-server-install --uninstall to clean up.
 

Resolution:

1- Run /usr/sbin/ipa-server-install --uninstall to clean up.

2- Remove the followings references:

rm -rf /var/log/pki/pki-tomcat
rm -rf /etc/sysconfig/pki-tomcat
rm -rf /etc/sysconfig/pki/tomcat/pki-tomcat
rm -rf /var/lib/pki/pki-tomcat
rm -rf /etc/pki/pki-tomcat

3- Install it again.

===================================================================================
Issue 3 - Error while joining to the existing Domain as ipa client.

Cause: 

If you have already tried to join to the domain and you have received errors,
you may encounter errors while joining to the Domain.

Resolution:

add --force-join to the command:
$ ipa-client-install --force-join




References:

4.5. CREATING THE REPLICA: INTRODUCTION
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/linux_domain_identity_authentication_and_policy_guide/creating-the-replica#creating-the-replica-from-scratch


D.4. PROMOTING A REPLICA TO A MASTER CA SERVER
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/linux_domain_identity_authentication_and_policy_guide/moving-crl-gen-old#promote-cert-renewal-old

14.5. MANUAL ID RANGE EXTENSION AND ASSIGNING A NEW ID RANGE
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/linux_domain_identity_authentication_and_policy_guide/man-set-extend-id-ranges

