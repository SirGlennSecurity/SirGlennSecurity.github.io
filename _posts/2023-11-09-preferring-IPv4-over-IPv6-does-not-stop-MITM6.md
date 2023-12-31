### Why Preferring IPv4 over IPv6 does not stop the MITM6 attack

If you have ever tried to fix the somewhat infamous IPv6 attack often known as [mitm6](https://blog.fox-it.com/2018/01/11/mitm6-compromising-ipv4-networks-via-ipv6/) then you have likely stumbled upon [Microsoft's guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/configure-ipv6-in-windows) to prefer IPv4 over IPv6 instead of outright disabling IPv6. This fails to stop attackers though, in fact, it doesn't even slow them down.


## Overview

As a brief reminder of how the attack works. An attacker obtains a rouge or compromised host on a subnet with Windows-based devices. These devices, by default, will attempt to find an IPv6 DHCP server. This DHCP6 server can also provide DNS settings or even only DNS settings through the use of a router advertisement  packet using RDNSS. The affected client now uses the provided IPv6 DNS server which either says that there is no IP address for that name or responds that they do have a DNS record, and it's the attacker's IP address. Presuming the attacker won the race condition with the normal DNS server, this causes the client to make its desired request to the server who can then ask the client to authenticate.

So why doesn't Prefering IPv4 over IPv6 solve this issue? I mean, if we don't use IPv6 we won't use the attacker's machine as a DNS server, right?

## Technical details

Preferring IPv4 over IPv6 refers to the [Default Address Selection for Internet Protocol version 6](https://www.ietf.org/rfc/rfc3484.txt) or rfc3484. This RFC describes two algorithms that help computers decide if they should *initiate* a connection via IPv6 or IPv4.


We can see the baseline weighting Windows gives when selecting an IPv6 or IPv4 address by running the command

`netsh interface ipv6 show prefixpolicies`

![Here we can see the results of that command with the default windows settings](/assets/IPv6/regWithIPv6-IPv4.PNG)

if we change the value with Window's recommended command:

`reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" /v DisabledComponents /t REG_DWORD /d <value> /f`

and restart the device we can see the values change.

![The Precedence of the IPv4 prefix, ::ffff:0:0/96 is changed to 1](/assets/IPv6/regWithIPv4-IPv6.PNG)

You can find a list of what these prefixes mean in [RFC-6890](https://datatracker.ietf.org/doc/html/rfc6890#section-2.2.3)

The problem is this does not control if we prefer our DNSv6 server's answer over our DNSv4 server's answer. Only if we get back both an IPv4 and an IPv6 address, we prefer the IPv4 addresses over the IPv6. In addition, browsers such as Chrome won't even use this setting and instead use a method known as [Happy Eyeballs](https://datatracker.ietf.org/doc/html/rfc8305). Which essentially queries both DNS servers and whoever responds first is the one used.

The attacker-provided DNS server has a couple of advantages in this race. First, browsers appear to prefer the IPv6 DNS server over the IPv4.

![The DNS server with a IPv6 address gets a very small advantage by going first](/assets/IPv6/dnsrequestTimes.png)

Second, the attacker has to be on the same subnet as the affected host due to the DHCPv6 requests being local subnet only. The DNS server might be both logically and physically further away.

Finally, and quite possibly the most important, the attacker's DNS server does not need to look up any IP addresses. If the attacker is spoofing the selected DNS string, it replies with the attacker's IP address. If the DNS string is not being spoofed, it replies with an empty answer field.

This all results in a strong likelihood that the attacker will win the race condition and its IPv4 address will be used.


So if preferring IPv4 over IPv6 does not remediate the IPv6 attack, what does?


## Remediation:

Well first and foremost if you can, just implement DHCPv6-Shield, or RFC7610. Cisco calls this DHCPv6 Guard and I am sure other vendors call it something else. [https://www.rfc-editor.org/rfc/rfc7610] The idea is simple though, if DHCPv6 traffic does not come from a port specifically allowed to send DHCPv6 traffic, drop it.

Second, if you simply cannot support DHCPv6-Shield, consider using the Windows firewall instead to block all DCHPv6 traffic. We can employ these configs en-mass using GPOs. Since this is a Windows-centric attack, it does the job.

Lastly, and certainly the worst option, you can simply disable IPv6 with the same registry we mentioned in the opening. The trouble is Windows has IPv6 baked into its DNA now and turning it off can manifest into weird, hard-to-diagnose problems. If you're going to go this route, make sure to keep it in the back of your mind when solving issues. The reason a client cannot connect to the SMBshare might just be because Windows is trying to use IPv6 for "something".
