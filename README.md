= Start --&gt; HERE &lt;--  =

== Build MAYO with CentOS 7  ==
*Main Server anaconda file:

 Put data here...

*Update the OS:

 yum update

== Install Needed Tools  ==

*Named Server

 yum install bind bind-utils

*DHCPD Server

 <source lang="bash">yum -y install dhcp</source>

*NIS/NIS+ (YP) Server

 <source lang="bash">yum -y install ypserv</source>

*TFTP Boot Services

 <source lang="bash">yum -y install tftp-server</source>

*XFS File Tools

 <source lang="bash">yum -y install xfsdump xfsprogs kmod-xfs</source>

*Openmotif (Needed for SGE)

 <source lang="bash">yum -y install openmotif</source>

<br>

== Configure Services  ==

=== NFS /local  ===

*Setup and install services

 yum install nfs-utils

 systemctl enable rpcbind
 systemctl enable nfs-server
 systemctl enable nfs-lock
 systemctl enable nfs-idmap
 systemctl start rpcbind
 systemctl start nfs-server
 systemctl start nfs-lock
 systemctl start nfs-idmap

*Edit the /etc/exports file
<pre>/home bread.blt.lclark.local(rw,no_wdelay,async,insecure,no_root_squash) *.blt.lclark.local(rw,no_wdelay,async,insecure) 192.168.0.*(rw,no_wdelay,async,insecure)
</pre>
<br>

 systemctl restart nfs-server

*Firewall
 firewall-cmd --permanent --zone=public --add-service=nfs
 firewall-cmd --reload

=== BIND / Named Server  ===

*After install you need to create a configuration file:

 /etc/named.conf

<br>

*Example File:

 //
 // named.conf
 //
 // Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
 // server as a caching only nameserver (as a localhost DNS resolver only).
 //
 // See /usr/share/doc/bind*/sample/ for example named configuration files.
 //
 acl "trusted" {
 192.168.0.0/24;
 };

 options {
    listen-on port 53 { 127.0.0.1; 192.168.0.1; };
    #listen-on-v6 port 53 {&nbsp;::1; };
    directory       "/var/named";
    dump-file       "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    allow-query     { trusted; };

    /*
     - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
     - If you are building a RECURSIVE (caching) DNS server, you need to enable
       recursion.
     - If your recursive DNS server has a public IP address, you MUST enable access
       control to limit queries to your legitimate users. Failing to do so will
       cause your server to become part of large scale DNS amplification
       attacks. Implementing BCP38 within your network would greatly
       reduce such attack surface
    */
    recursion yes;
    allow-recursion { trusted; };
    dnssec-enable yes;
    dnssec-validation yes;
    forwarders {
            149.175.1.2;
            192.54.243.2;
    };
    /* Path to ISC DLV key */
    bindkeys-file "/etc/named.iscdlv.key";

    managed-keys-directory "/var/named/dynamic";

    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";
 };

 logging {
    channel default_debug {
            file "data/named.run";
            severity dynamic;
    };
 };

 zone "." IN {
    type hint;
    file "named.ca";
 };

 include "/etc/named.rfc1912.zones";
 include "/etc/named.root.key";
 include "/var/named/named.conf.local";

<br>

*named.conf.local

 # cat /var/named/named.conf.local
 zone "blt.lclark.local" IN {
        type master;
        file "zones/blt.lclark.local";
        allow-update { none; };
 };

 zone "0.168.192.in-addr.arpa" IN {
        type master;
        file "zones/0.168.192.in-addr.arpa";
        allow-update { none; };
 };

<br>

*Next you need to create zone file to reflect the configuration file abov. All file need to be located in /var/named/chroot/var/named/zones/master

 [root@ named]# cd /var/named
 [root@ named]# ls -al
 total 24
 drwxr-x---  5 root  named 4096 May 11 11:29 .
 drwxr-xr-x 29 root  root  4096 May 11 20:37 ..
 drwxrwx---  6 root  named 4096 May 11 09:53 chroot
 drwxrwx---  2 named named 4096 Jan  4  2005 data
 lrwxrwxrwx  1 root  root    44 May 11 09:42 localdomain.zone -&gt; /var/named/chroot/var/named/localdomain.zone
 lrwxrwxrwx  1 root  root    42 May 11 09:42 localhost.zone -&gt; /var/named/chroot/var/named/localhost.zone
 lrwxrwxrwx  1 root  root    43 May 11 09:42 named.broadcast -&gt; /var/named/chroot/var/named/named.broadcast
 lrwxrwxrwx  1 root  root    36 May 11 09:42 named.ca -&gt; /var/named/chroot/var/named/named.ca
 lrwxrwxrwx  1 root  root    43 May 11 09:42 named.ip6.local -&gt; /var/named/chroot/var/named/named.ip6.local
 lrwxrwxrwx  1 root  root    39 May 11 09:42 named.local -&gt; /var/named/chroot/var/named/named.local
 lrwxrwxrwx  1 root  root    38 May 11 09:42 named.zero -&gt; /var/named/chroot/var/named/named.zero
 drwxrwx---  2 named named 4096 Jan  4  2005 slaves
 lrwxrwxrwx  1 root  root    33 May 11 09:42 zones -&gt; /var/named/chroot/var/named/zones
 [root@ master]# cd /var/named/chroot/var/named/zones/master/
 [root@ master]# ls -arlt
 total 40
 drwxrwx--- 3 named named 4096 Nov 21  2008 ..
 -rw-r--r-- 1 named named 8889 May 11 09:48 0.168.192.in-addr.arpa
 -rw-r--r-- 1 named named 6382 May 11 11:45 blt.lclark.local
 drwxrwx--- 2 named named 4096 May 11 11:45 .
 [root@ master]#


*The file in this directory are the files listed in the configuration file /etc/named.conf and contain the DNS data for the ARP and RARP requests.

*Main file for "blt.lclark.local" domain and 0.168.192.in-addr.arpa:

 [root@ named]# cat zones/blt.lclark.local
 $ORIGIN blt.lclark.local.
 $TTL 86400
 @       IN      SOA     blt.lclark.local. hostmaster.blt.lclark.local. (
                    2017092501 &nbsp;; serial, todays date + todays serial #
                    10800 &nbsp;; 8H - refresh, seconds
                    3600 &nbsp;; 2H - retry, seconds
                    604800 &nbsp;; 1W - expire, seconds
                    86400 ) &nbsp;; 1D - minimum, seconds - TTL

 ; Name Servers (The name '@' is implied)
            IN      NS      blt.lclark.local.      &nbsp;; Inet Addr of 1st name server
            IN      A       192.168.0.1 &nbsp;; IP of Domain


 ; Local Address for Intra-Net 192.168.0.0
 ; Teaching infrastructure
 mayo           IN      A       192.168.0.1
 bread          IN      A       192.168.0.2
 bacon          IN      A       192.168.0.101
 lettuce        IN      A       192.168.0.102
 tomato         IN      A       192.168.0.103


 [root@ named]# cat zones/0.168.192.in-addr.arpa
 $TTL 86400
 @ IN SOA blt.lclark.local. admin.blt.lclark.local. (
    2016083101;
    10800  &nbsp;;
    3600   &nbsp;;
    604800 &nbsp;;
    86400 )&nbsp;;

    IN NS   blt.lclark.local.

 ;Teaching address PTR's
 1         IN PTR  mayo.blt.lclark.local.
 2         IN PTR  bread.blt.lclark.local.
 101       IN PTR  bacon.blt.lclark.local.
 102       IN PTR  lettuce.blt.lclark.local.
 103       IN PTR  tomato.blt.lclark.local.



==== Start service  ====

 systemctl enable named
 systemctl start named

=== NAT / Firewalld  ===

*Set ipv4 forward for the kernel

 echo 'net.ipv4.ip_forward = 1' &gt; /etc/sysctl.d/01-sysctl.conf

 [root@ ~]# cat /etc/sysctl.d/01-sysctl.conf
 net.ipv4.ip_forward = 1

*Verify it worked...<br>

 sysctl -p
<pre>[root@avery ~]#  cat  /proc/sys/net/ipv4/ip_forward
1

</pre>
*First, we need to permanently assign each NIC to its own firewall zone. Firewalld has several zones already pre-defined, which can be listed using the following command:
<pre># firewall-cmd --get-zones
block dmz drop external home internal public trusted work

</pre>
*The pre-defined default zone is the '''public''' zone:
<pre># firewall-cmd --get-default-zone
public</pre>
*The simplest approach is to use the default '''public''' zone for the external network, and to assign the internal network to the pre-defined '''internal''' zone.
<pre>By default, firewalld automatically assigns all interfaces to the default zone:
firewall-cmd --list-all

public (default, active)
interfaces: p2p1 p2p2
sources:
services: dhcpv6-client ssh ntp http https ldap ldaps kerberos kpasswd dns nfs
ports: 732/tcp 53/tcp
masquerade: no
forward-ports:
icmp-blocks:
rich rules:
</pre>
Here, firewalld knows about two interfaces, '''p2p1''' and '''p2p2''', and both are assigned to the '''public''' zone. We want to keep '''p2p1''' in the '''public''' zone and assign '''p2p2''' to the '''internal''' zone.
<pre>firewall-cmd --permanent --zone=public --remove-interface=p2p2
firewall-cmd --permanent --zone=internal --add-interface=p2p2
</pre>
The problem with this approach is that every time firewalld or the server is restarted, all interfaces are reassigned back to the '''public''' zone despite the use of the '''–permanent''' option in the above commands.

It turns out that the only way to permanently assign an interface to a zone is to edit the interface’s configuration file ('''/etc/sysconfig/network-scripts/ifcfg-p2p2''' in this case) and add a ZONE option as follows:
<pre>ZONE=internal</pre>
*<br>

==== NAT Rules  ====

The following rules allow machines on the isolated internal network (on p2p2) to send NATed packets to the college network (on p2p1), and also allow responses back. Machines on the college network cannot initiate communications with student machines on the internal network:<br>
<pre>firewall-cmd --permanent --zone=internal --add-port=53/udp
firewall-cmd --permanent --zone=internal --add-port=67/udp
firewall-cmd --permanent --zone=internal --add-port=123/udp
firewall-cmd --permanent --direct --add-rule ipv4 nat POSTROUTING 0 -o p2p1 -j MASQUERADE
firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 0 -i p2p2 -o p2p1 -j ACCEPT
firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 0 -i p2p1 -o p2p2 -m state --state RELATED,ESTABLISHED -j ACCEPT
firewall-cmd --reload

firewall-cmd --permanent --zone=internal --add-port=111/udp
firewall-cmd --permanent --zone=internal --add-port=1039/udp
firewall-cmd --permanent --zone=internal --add-port=1047/udp
firewall-cmd --permanent --zone=internal --add-port=1048/udp
firewall-cmd --permanent --zone=internal --add-port=2049/udp

firewall-cmd --permanent --zone=internal --add-port=111/tcp
firewall-cmd --permanent --zone=internal --add-port=1039/tcp
firewall-cmd --permanent --zone=internal --add-port=1047/tcp
firewall-cmd --permanent --zone=internal --add-port=1048/tcp
firewall-cmd --permanent --zone=internal --add-port=2049/tcp
firewall-cmd --reload
</pre>
<br>

<br>

=== DCHPD Server  ===

*After install you need to create a configuration file:

 /etc/dhcp/dhcpd.conf

*Example File:

 #
 # DHCP Server Configuration file.
 #   see /usr/share/doc/dhcp*/dhcpd.conf.example
 #   see dhcpd.conf(5) man page
 #
 ddns-update-style none;
 subnet 192.168.0.0 netmask 255.255.0.0 {
      range 192.168.0.250 192.168.0.254;
      default-lease-time 86400;
      max-lease-time 86400;
      option routers 192.168.0.1;
      option ip-forwarding on;
      option broadcast-address 192.168.255.255;
      option domain-name "blt.lclark.local";
      option subnet-mask 255.255.0.0;
      option domain-name-servers 192.168.0.1;
      option ntp-servers 128.193.10.15;
      option netbios-name-servers 192.168.0.1;
      option netbios-dd-server 192.168.0.1;
      option netbios-node-type 8;
      option netbios-scope "";
      deny unknown-clients;

   group {
    next-server 192.168.0.1;
    filename "linux/pxelinux.0";

      host mayo {
      hardware ethernet A0:36:9F:BF:4B:62;
      fixed-address 192.168.0.1;
      }

      host bread {
      hardware ethernet A0:36:9F:BF:47:4A;
      fixed-address 192.168.0.2;
      }


      host bacon {
      hardware ethernet 00:25:90:59:64:46;
      fixed-address 192.168.0.101;
      }

      host lettuce {
      hardware ethernet 00:0e:1e:0d:d8:58;
      fixed-address 192.168.0.102;
      }

      host tomato {
      hardware ethernet 00:1B:21:29:CD:95;
      fixed-address 192.168.0.103;
      }

  }
 }


==== Start Service  ====

 systemctl enable dhcpd
 systemctl start dhcpd

=== TFTP Boot Service  ===

The TFTP Boot service will allow for PXE boot and network install of the nodes when we want to rebuild machines. The service file is locate in:
<pre>yum install tftp tftp-server syslinux wget
</pre>
We need to change the yes below to a "no".<br>

 /etc/xinetd.d/tftp

 OFF
 [root@avery ~]# cat /etc/xinetd.d/tftp
 # default: off
 # description: The tftp server serves files using the trivial file transfer \
 #       protocol.  The tftp protocol is often used to boot diskless \
 #       workstations, download configuration files to network-aware printers, \
 #       and to start the installation process for some operating systems.
 service tftp
 {
       socket_type             = dgram
       protocol                = udp
       wait                    = yes
       user                    = root
       server                  = /usr/sbin/in.tftpd
       server_args             = -s /var/lib/tftpboot
       disable                 = '''yes'''
       per_source              = 11
       cps                     = 100 2
       flags                   = IPv4
 }

 ON
 [root@avery ~]# cat /etc/xinetd.d/tftp
 # default: off
 # description: The tftp server serves files using the trivial file transfer \
 #       protocol.  The tftp protocol is often used to boot diskless \
 #       workstations, download configuration files to network-aware printers, \
 #       and to start the installation process for some operating systems.
 service tftp
 {
       socket_type             = dgram
       protocol                = udp
       wait                    = yes
       user                    = root
       server                  = /usr/sbin/in.tftpd
       server_args             = -s /var/lib/tftpboot
       disable                 = '''no'''
       per_source              = 11
       cps                     = 100 2
       flags                   = IPv4
 }


The main data is locate in:

 /var/lib/tftpboot

You configure the service using the DHCPD service and others like NFS. Inside the dhcpd.conf file you will find an entry for the "next-server". This is the IP address of the TFTP Boot server. Next you will find a "filename" entry. This is the path on the tftp server where the PXE boot loader is located. The file below is the full linux folder with the binary files needed to boot the PXE device.

 [[Image:Tftpboot-linux.tar.gz]]

Inside the "/tftpboot/linux" folder you will find initrd's and vmlinuz files for the versions of centos we will use for installs on the cluster.

 rsync -av -e 'ssh -p 732' /var/lib/tftpboot/ root@doolittle2.cgrb.oregonstate.edu:/var/lib/tftpboot/

 # ls -la
 total 138024
 drwxr-xr-x 3 root root     4096 Oct  4 12:58 .
 drwxr-xr-x 5 root root       49 Oct  4 12:47 ..
 -rw-r--r-- 1 root root      991 Nov 18  2016 boot.msg
 -r--r--r-- 1 root root 38508192 Dec  9  2015 initrd-centos7-1511.img
 -rw-r--r-- 1 root root 43372552 May  4 13:28 initrd-CentOS-7-x86_64-Everything-1611.img
 -rw-r--r-- 1 root root 43372552 May  3 14:33 initrd-CentOS-7-x86_64-Minimal-1611.img
 -rw-r--r-- 1 root root    20020 Jan  8  2007 memdisk
 -rw-r--r-- 1 root root    80324 May 11  2010 memtest
 -rw-r--r-- 1 root root      277 May 18  2010 options.msg
 -rw-r--r-- 1 root root    11304 May 11  2010 pxelinux.0
 drwxr-sr-x 2 root root       21 Oct  4 12:58 pxelinux.cfg
 -r-xr-xr-x 1 root root  5156528 Nov 19  2015 vmlinuz-centos7-1511
 -rwxr-xr-x 1 root root  5392080 May  4 13:28 vmlinuz-CentOS-7-x86_64-Everything-1611
 -rwxr-xr-x 1 root root  5392080 Jun 14 12:39 vmlinuz-CentOS-7-x86_64-Minimal-1611




<br> The boot.msg file contains the menu you get at PXE boot and let you know what to type at the prompt based on what you want to install. The entries in this file need to match what you put into file "pxelinux.cfg/defatult" file. This file holds all the configuration info for the install like the kick start files and such.This information will point at the NFS&nbsp;server holding all the goods and tell it what to do next.<br>

 # cat pxelinux.cfg/default
 default centos7all
 prompt 1
 timeout 100
 display boot.msg
 F1 boot.msg
 F2 options.msg

 label mem
 kernel memtest

 # CentOS-7
 label centos7
 kernel vmlinuz-centos7-1511
 append ks=nfs:192.168.0.1:/local/cluster/RedHat_OS/linux/centos7-1511_new.ks.cfg initrd=initrd-centos7-1511.img ksdevice=eth0

 # CentOS-7
 label centos7dell
 kernel vmlinuz-centos7-1511
 append ks=nfs:192.168.0.1:/local/cluster/RedHat_OS/linux/centos7dell-minimal.ks.cfg initrd=initrd-centos7-1511.img ksdevice=em1

 # CentOS-7
 label centos7server
 kernel vmlinuz-centos7-1511
 append ks=nfs:192.168.0.1:/local/cluster/RedHat_OS/linux/centos7-1511_server.ks.cfg initrd=initrd-centos7-1511.img

 # CentOS-7
 label centos7minimal
 kernel vmlinuz-centos7-1511
 append ks=nfs:192.168.0.1:/local/cluster/RedHat_OS/linux/centos7-1511-minimal.ks.cfg initrd=initrd-centos7-1511.img

 # CentOS-7
 label centos7all
 kernel vmlinuz-centos7-1511
 append ks=nfs:192.168.0.1:/local/cluster/RedHat_OS/linux/centos7-1511-everything.ks.cfg initrd=initrd-centos7-1511.img

 # CentOS-7
 label centos7dellall
 kernel vmlinuz-centos7-1511
 append ks=nfs:192.168.0.1:/local/cluster/RedHat_OS/linux/centos7-1511-dell-everything.ks.cfg initrd=initrd-centos7-1511.img

 # CentOS-7
 label centos7dellactf
 kernel vmlinuz-centos7-1511
 append ks=nfs:192.168.0.1:/local/cluster/RedHat_OS/linux/centos7-1511-dell-teaching.ks.cfg initrd=initrd-centos7-1511.img

 # CentOS-7
 label centos7dellweb
 kernel vmlinuz-centos7-1511
 append ks=nfs:192.168.0.1:/local/cluster/RedHat_OS/linux/centos7dell-minimal.ks.cfg initrd=initrd-centos7-1511.img ksdevice=p2p2

 # CentOS-7-ppc64le-Minimal-1611
 label c7ppcmin
 kernel vmlinuz-centos7-ppc64le-Minimal-1611
 append ks=nfs:192.168.0.1:/local/cluster/RedHat_OS/linux/centos7-ppc64-minimal-1611.ks.cfg initrd=initrd-centos7-ppc64le-Minimal-1611.img console=hvc0 console=tty0

 # CentOS-7-ppc64le-Minimal-1611
 label c7ppcmin-vm
 kernel vmlinuz-centos7-ppc64le-Minimal-1611
 append ks=nfs:192.168.0.1:/local/cluster/RedHat_OS/linux/centos7-vm-ppc64-minimal-1611.ks.cfg initrd=initrd-centos7-ppc64le-Minimal-1611.img console=hvc0 console=tty0

 # CentOS-7-x86-Minimal-1611 VM
 label c7x86min-vm
 kernel vmlinuz-CentOS-7-x86_64-Minimal-1611
 append ks=nfs:192.168.0.1:/local/cluster/RedHat_OS/linux/centos-7-vm-x86_64-minimal-1611.ks.cfg initrd=initrd-CentOS-7-x86_64-Minimal-1611.img console=hvc0 console=tty0

 # CentOS-7-x86-Everything-1611 VM
 label c7x86all-vm
 kernel vmlinuz-CentOS-7-x86_64-Everything-1611
 append ks=nfs:192.168.0.1:/local/cluster/RedHat_OS/linux/centos-7-vm-x86_64-everything-1611.ks.cfg initrd=initrd-CentOS-7-x86_64-Everything-1611.img console=hvc0 console=tty0

 # CentOS-7-x86-Everything-1611 Xen VM
 label c7x86all-xvm
 kernel vmlinuz-CentOS-7-x86_64-Everything-1611
 append ks=nfs:192.168.0.1:/local/cluster/RedHat_OS/linux/centos-7-xvm-x86_64-everything-1611.ks.cfg initrd=initrd-CentOS-7-x86_64-Everything-1611.img console=hvc0 console=tty0

 # CentOS-7-x86-Minimal-1611 Xen VM
 label c7x86min-xvm
 kernel vmlinuz-CentOS-7-x86_64-Minimal-1611
 append ks=nfs:192.168.0.1:/local/cluster/RedHat_OS/linux/centos-7-xvm-x86_64-minimal-1611.ks.cfg initrd=initrd-CentOS-7-x86_64-Minimal-1611.img console=hvc0 console=tty0
 [root@avery linux]#
<pre>systemctl restart rpcbind.service
firewall-cmd --add-port=69/udp --permanent
firewall-cmd --add-service=dhcp --permanent
firewall-cmd --add-port=4011/udp --permanent
firewall-cmd --reload
</pre>
 systemctl start xinetd
 systemctl start tftp
 systemctl enable xinetd
 systemctl enable tftp

=== Openmotif (Needed for SGE)  ===

We will need Openmotif for the X output of the SGE system. The install was done above.
<pre># yum install openmotif
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirror.sfo12.us.leaseweb.net
 * extras: mirrors.syringanetworks.net
 * updates: mirrors.kernel.org
Package motif-2.3.4-8.1.el7_3.x86_64 already installed and latest version
Nothing to do

</pre>

=== PXE Boot  ===

This PXE Boot system is handled by the TFTP Boot service and its configuration files.

<br>

== Sync /local  ==

 rsync -av -e 'ssh -p 732' /local/cluster/ root@avery.actf.oregonstate.edu:/local/cluster/


== NIS/YP Install ==
*Install tools:

 yum -y install ypserv rpcbind
 ypdomainname BLT-NIS
 echo "NISDOMAIN=BLT-NIS" >> /etc/sysconfig/network

*Add network to/var/yp/securenets

 cat /var/yp/securenets
 255.255.255.0   192.168.0.0

*Start Services:

 systemctl start rpcbind ypserv ypxfrd yppasswdd
 systemctl enable rpcbind ypserv ypxfrd yppasswdd

*YPINIT Startup:

 /usr/lib64/yp/ypinit -m


*Firewall Config:

 # tail /etc/sysconfig/network
 YPSERV_ARGS="-p 944"
 YPXFRD_ARGS="-p 945"

 # vi /etc/sysconfig/yppasswdd
 YPPASSWDD_ARGS="--port 946"


 systemctl restart rpcbind ypserv ypxfrd yppasswdd
 firewall-cmd --add-service=rpc-bind --permanent
 firewall-cmd --add-port=944/tcp --permanent
 firewall-cmd --add-port=944/udp --permanent
 firewall-cmd --add-port=945/tcp --permanent
 firewall-cmd --add-port=945/udp --permanent
 firewall-cmd --add-port=946/udp --permanent
 firewall-cmd --reload

== Install SGE  ==

*Add user sgeadmin to system:

 sgeadmin
 --homedir=/home/cgrb/sgeadmin
 --shell=/bin/csh
 --email=sgeadmin@blt.lclark.edu
 --uid=675
 --gid=4
<pre>Grid Engine cluster configuration

---------------------------------



Please give the basic configuration parameters of your Grid Engine

Installation:


</pre>
== Install Python 2.7.14 ==

= Install Node&nbsp; =

== PXE Boot Node ==

*You will need to obtain the MAC address for the machine you are trying to add to the cluster. Many times you can do this by having the machine boot from the network and hit "pause" once it starts to DHCP for an address. Once you have the MAC you will need to add that to the DNS and DHCP above. Once its in the DNS and DHCP and you have restarted those services&nbsp; you can now have the node finish the DHCP process and a menu should be presented to install. If the menu comes up you have everything correct and it will automatically install the machine.

