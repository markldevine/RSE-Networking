My development environment, "Raku Services Environment" (RSE), is hosted on my laptop, which
is Windows 11 Pro, WSL2 (OpenSUSE Tumbleweed; networking=mirrored), and five Hyper-V VMs.

All participants use "Default Switch" to reach the internet.  DNS resolution is working well
for internet lookups on all entities.

I created an internal virtual switch, "vSwitch-Dev", intended for internal traffic only (no
external routing), configured in Windows with a static IP address '172.19.2.98'.
<vSwitch-Dev>
Ethernet adapter vEthernet (vSwitch-Dev):

   Connection-specific DNS Suffix  . :
   Description . . . . . . . . . . . : Hyper-V Virtual Ethernet Adapter #2
   Physical Address. . . . . . . . . : 00-15-5D-01-39-0B
   DHCP Enabled. . . . . . . . . . . : No
   Autoconfiguration Enabled . . . . : Yes
   Link-local IPv6 Address . . . . . : fe80::e96b:6aa6:d089:dbdf%27(Preferred)
   IPv4 Address. . . . . . . . . . . : 172.19.2.98(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . :
   DHCPv6 IAID . . . . . . . . . . . : 671094109
   DHCPv6 Client DUID. . . . . . . . : 00-01-00-01-2F-F1-21-E5-60-7D-09-B4-91-56
   DNS Servers . . . . . . . . . . . : 172.19.2.100
                                       172.19.2.101
                                       172.19.2.102
                                       172.19.2.103
   NetBIOS over Tcpip. . . . . . . . : Enabled
</vSwitch-Dev>

<Hyper-V VMs>
- factory (app development; container builds)
    - eth0: Windows 11 "Default Switch" (NAT, DNS inherited from WiFi adapter DHCP)
    - eth1: Virtual switch, "vSwitch-Dev", 172.19.2.99
- repository (podman image repository; dnsmasq server)
    - eth0: Windows 11 "Default Switch" (NAT, DNS inherited from WiFi adapter DHCP)
    - eth1: Virtual switch, "vSwitch-Dev", 172.19.2.100
- mos01 (worker node)
    - eth0: Windows 11 "Default Switch" (NAT, DNS inherited from WiFi adapter DHCP)
    - eth1: Virtual switch, "vSwitch-Dev", 172.19.2.101
- mos02 (worker node)
    - eth0: Windows 11 "Default Switch" (NAT, DNS inherited from WiFi adapter DHCP)
    - eth1: Virtual switch, "vSwitch-Dev", 172.19.2.102
- mos03 (worker node)
    - eth0: Windows 11 "Default Switch" (NAT, DNS inherited from WiFi adapter DHCP)
    - eth1: Virtual switch, "vSwitch-Dev", 172.19.2.103
</Hyper-V VMs>

'repository' acts as a dnsmasq server, with the worker nodes acting as replicas (rsync/timer).

I am running systemd-resolved in an attempt to achieve the DNS resolution I seek.  It is not
working yet.

<systemd-resolved>
factory:~> cat /etc/systemd/resolved.conf
Cache=no-negative
</systemd-resolved>

<resolvectl status>
factory:~> resolvectl status | grep -A3 'Link' | grep -E 'DNS|Domain'
    Current Scopes: DNS LLMNR/IPv4 LLMNR/IPv6 mDNS/IPv4 mDNS/IPv6
         Protocols: +DefaultRoute +LLMNR +mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 172.26.96.1
    Current Scopes: DNS LLMNR/IPv4 mDNS/IPv4
         Protocols: -DefaultRoute +LLMNR +mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 172.19.2.100
<resolvectl status>

<repository:/etc/dnsmasq.d/rse.conf>
root@repository:~> grep -v -e ^$ -e ^# /etc/dnsmasq.d/rse.conf 
interface=eth1
bind-interfaces
domain=rse.local
expand-hosts
local=/rse.local/
domain-needed
no-resolv
server=172.28.16.1
host-record=factory.rse.local,172.19.2.99
host-record=repository.rse.local,172.19.2.100
host-record=mos01.rse.local,172.19.2.101
host-record=mos02.rse.local,172.19.2.102
host-record=mos03.rse.local,172.19.2.103
host-record=valkey-vip.rse.local,172.19.2.254
</repository:/etc/dnsmasq.d/rse.conf>

<resolution status>
root@factory:~> time getent hosts www.ibm.com
2600:141b:f000:148b::1e89 e7817.dscx.akamaiedge.net www.ibm.com outer-global-dual.ibmcom-tls12.edgekey.net
2600:141b:f000:1481::1e89 e7817.dscx.akamaiedge.net www.ibm.com outer-global-dual.ibmcom-tls12.edgekey.net

real    0m0.037s
user    0m0.001s
sys     0m0.002s
root@factory:~> 

root@factory:~> time getent hosts mos01
172.26.107.123  mos01.mshome.net

real    0m3.891s
user    0m0.001s
sys     0m0.001s
root@factory:~> time getent hosts mos01.rse.local
172.19.2.101    mos01.rse.local

real    0m10.016s
user    0m0.000s
sys     0m0.003s
</resolution status>
