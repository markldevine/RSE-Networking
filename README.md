My development environment, "Raku Services Environment" (RSE), is hosted on my laptop, which
is Windows 11 Pro, WSL2 (OpenSUSE Tumbleweed; networking=mirrored), and five Hyper-V VMs:

I created an internal virtual switch, "vSwitch-Dev", intended for internal traffic only (no
external routing), with static IP address '172.19.2.98'.
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


- factory (app development; container builds)
    - eth0: Windows 11 "Default Switch" (NAT, DNS inherited from WiFi adapter DHCP)
    - eth1: Virtual switch, "vSwitch-Dev", 172.19.2.99
- mos01 (worker node)
    - eth0: Windows 11 "Default Switch" (NAT, DNS inherited from WiFi adapter DHCP)
- mos02 (worker node)
    - eth0: Windows 11 "Default Switch" (NAT, DNS inherited from WiFi adapter DHCP)
- mos03 (worker node)
    - eth0: Windows 11 "Default Switch" (NAT, DNS inherited from WiFi adapter DHCP)
- repository (podman image repository; dnsmasq server)
    - eth0: Windows 11 "Default Switch" (NAT, DNS inherited from WiFi adapter DHCP)
