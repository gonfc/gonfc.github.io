---
uuid: 20220510110056
created: 2022-05-10T11:01
updated: 2022-05-10T11:00
tags: [meta/type/note, meta/template/frontmatter, meta/language/en] 
aliases: [[Networking]]
down:
# Anki
deck: IT::Networking::Router
topics: Router_config
# User
lessons_learned: ["Routing is hard work eh"]
---
up:: [[Router]]
next:: [[Switch configuration]]

Overview:: "1. Connect nodes; 2. Configure routers with endless dhcp pools; 3. Switch fails to start. Restart PT thrice, only understand it was the routers all along. 4. Ponder why VLANs keep blinking nonstop; 5. The one piece were the ~~friends~~ nodes we made all along"

# Router configuration
[[Router]]
[[Basic template for network configuration exercises]]
[[Switch configuration]]
[[Cisco 8.5.1 Lab - Configure DHCPv6]] 

> [!note]
> If Obsidian's '*Export to PDF*' is used, text, when copied, formats to *left align*  (i.e., no indentation; easy to paste into a terminal). 

## Full generic router configuration

```powershell
en 
conf t 
	enable secret cisco
	hostname RT_GW
	no ip domain-lookup 
	ip domain-name maisum.com
	username admin privilege 15 secret cisco
	crypto key generate rsa general-keys modulus 2048 
	service password-encryption 
	banner motd # Acao legal sera instaurada por qualquer uso nao autorizado | Authorized Access Only #
	line con 0  
		password cisco	 
		login local
		exit 
	line vty 0 4 
		password cisco 
		exec-timeout 5 30
		login local 
		access-class GESTAO in
		transport input ssh
		exit
	ip route 10.10.4.128 255.255.255.192 g0/1
	ip route 10.10.3.128 255.255.255.128 g0/1
	ip route 10.10.2.0 255.255.255.0 g0/1
	int g0/1
		no shut
		exit
	int g0/1.55
		encapsulation dot1q s55
		ip default-gateway 10.10.10.3
		ip add 10.10.4.129 255.255.255.192
		ip nat inside 
		exit 
	int g0/1.111
		encapsulation dot1q 111
		ip default-gateway 10.10.10.3
		ip add 10.10.2.1 255.255.255.0
		ip nat inside 
		exit
	int g0/1.12
		encapsulation dot1q 12
		ip default-gateway 10.10.10.3
		ip add 10.10.3.129 255.255.255.128
		ip nat inside 
		exit
	int f0/0
		no shut
		ip nat outside
		exit
	int range f0/2-24
		shut
		exit
	ip dhcp excluded-address 192.168.24.1
	ip dhcp excluded-address 192.168.34.1
	ip dhcp excluded-address 192.168.0.222
	ip dhcp pool 10.0.112.0
		network 10.0.112.0 255.255.240.0
		default-router 10.0.112.1
		exit
		access-list 10 permit 10.0.10.0 0.0.0.255 
		access-list 10 permit 10.0.20.0 0.0.0.255
		access-list 10 permit 10.0.30.0 0.0.0.255
		ip nat inside source list 10 interface g0/1 overload 
	# obligatory Router model 2811
	ip dhcp pool VOZ 
		network 10.0.10.0 255.255.255.0
		default-router 10.0.10.1
		option 150 ip 10.0.10.1 
		exit
	telephony-service
	max-dn 5
	max-ephones 5
	ip source-address 10.0.10.1 port 2000
	auto assign 1 to 5
	ephone-dn 1
		number 54001
	ephone-dn 2
		number 54002
	dial-peer voice 1 voip
		destination pattern 351.
		session target ipv4:10.2.2.1
```
### Common commands
- `do sh run | section line vty`
- `show cdp neighbors`
### DNS
1. Change DNs on PCs
2. On R1:
	```markdown
	# Example DNS server is 1.1.1.1
	ip name-server 1.1.1.1
	ip host R1 192.168.0.254
	ip host PC1 192.168.0.1
	ip host PC2 192.168.0.2
	ip host PC3 192.168.0.3
	```
- Can also be set with DHCP

## Interfaces

```markdown
int f0/2
	no shut
	ip add 10.0.0.1 255.255.192.0
	exit
```
### Sub interfaces
```
int g0/1
	no shut
	exit
int g0/1.55
	encapsulation dot1q 55
	ip add 10.10.4.129 255.255.255.192
	ip nat inside 
	exit 
int g0/1.111
	encapsulation dot1q 111
	ip add 10.10.2.1 255.255.255.0
	ip nat inside 
	exit
int g0/1.12
	encapsulation dot1q 12
	ip add 10.10.3.129 255.255.255.128
	ip nat inside 
	exit
int f0/0
	no shut
	ip nat outside
	exit
```

```
int g0/1.55
	ip add dhcp
	ip helper-address 192.168.100.3
```

### Interfaces without nat
```markdown
Interfaces w/ nat
int g0/0
	no shut
	exit
int g0/1.10
	encapsulation dot1q 10
	ip add 10.0.10.1 255.255.255.0
	exit 
int g0/1.20
	encapsulation dot1q 20
	ip add 10.0.20.1 255.255.255.0
	exit
int g0/1.30
	encapsulation dot1q 30
	ip add 10.0.30.1 255.255.255.0
	exit
int g0/0
	exit
```

```ad-note
title: Only the subinterfaces receive traffic from outside but...
collapse: yes
- ACLs **can** be used in g0/0
```

```ad-caution
Interface  g0/1 must be turned on for int g0/0.1 to work
```

### Subnet cheatsheet
![[Pasted image 20220511110005.png]]
[[Subnetting]]

| Subnet | Mask |
| ------ | ---- |
| /27    | .224 |
| /26    | .192 | 
| /25    | .168 |

##  Routes
- Format :: Incoming Network Address | Netmask | Destination Interface / IP
<!--ID: 1653066573840-->

```markdown
ip route 0.0.0.0 0.0.0.0 f0/0

ip route 10.0.135.0 255.255.255.128 s2/0
ip route 10.0.135.128 255.255.255.128 s2/0
ip route 10.0.96.0 255.255.240.0 s2/0
```

Enable an alternative static route using distance
```
ip route 10.0.0.0 255.0.0.0 s2/0 250 
ip route 10.0.0.0 255.0.0.0 s2/1
```

## DHCP

```markdown
ip dhcp excluded-address 192.168.24.1
ip dhcp excluded-address 192.168.34.1
ip dhcp excluded-address 192.168.0.222
ip dhcp pool 10.0.112.0
	network 10.0.112.0 255.255.240.0
	default-router 10.0.112.1
	exit
```

```
do show run | section dhcp
```
### What should be excluded from the pool
- Router
- Servers
### Other commands within the pool
```markdown
ip dhcp pool POOL
	network 110.0.112.0 /28
	dns-server 1.1.1.1
	domain-name jeremysitlab.com
	# Lease format: days hours minutes
	lease 0 5 30
```
### Other commands:
- `do show ip dhcp pool` 
- 

## NAT
[[Show command#ip show nat translations]]
### Commands
> [!Nome]
> 421412

> [!f3]- Show commands
> ```ios
> # inside global, inside local, outside local, outside global
> # with ports
> show ip nat translations
> show ip nat statistics
> show access-list 1 pool POOL1 refcount 6
> ```
> 
### Static NAT
[!f3]- Config
```
# translates IP addresses... one by one 
ip nat inside source static 192.168.1.2 180.10.1.15
ip f0/1
	ip nat inside
	exit
ip f0/0
	ip nat outside
	exit
conf t
	do show ip nat translation
```
### Dynamic NAT
> [!f3]- Config
> ```markdown
> conf t
> 	access-list 1 permit 102.168.0.0 0.0.255.255
> 	# ip nat pool POOL1 100.0.0.0 100.0.0.255 prefix-length 24
> 	ip nat pool POOL1 100.0.0.0 100.0.0.255 255.255.255.0
> 	# map the ACL to the pool
> 	ip nat inside source list 1 pool POOL1
> 	exit
> 
> ```
> 
<!--ID: 1653346582793-->




### PAT (NAT Overload)
> [!f3]- Concept
> - Translates both the IP address and the port number
> - Many hosts share a single IP => Useful for perserving IP addresses; 
<!--ID: 1653346582801-->


> [!f3]- Config
> The same as Dynamic NAT, with *overload*; uses dynamic ports instead of different IPs.
> ```ios
> int g0/1
> 	ip nat inside
> 	exit
> int g0/0
> 	ip nat outside
> 	exit
> config t
> 	# the addresses to be translated + wildcard
> 	access-list 1 permit 192.168.0.0 0.0.0.255
> 	ip nat inside source list 1 interface g0/0 overload 
> ```
<!--ID: 1653346582806-->


Alternate commands: 

```
conf t
	ip access-list standard IT_MANAGEMENT
		permit 192.168.0.0 0.0.255.255
		exit
	ip nat inside source list IT_MANAGEMENT  int g0/1 overload
```

[[Access list exercise]]

### Protocols table

| UDP or TCP | Port | Protocol | Notes      |
| ---------- | ---- | -------- | ---------- |
| TCP        | 21   | ftp      | L7, unused |
| TCP        | 22   | ssh      |            |
| TCP        | 23   | telnet   |            |
| TCP/UDP    | 25   | smtp     |            |
| TCP/UDP    | 53   | dns      |            |
| UDP        | 67   | dhcp     |            |
| TCP        | 80   | http     |            |
| TCP        | 110  | pop3     |            |
| UDP        | 161  | snmp     |            |
| TCP        | 443  | https    |            |
|            |      |          |            |

#### Arp inspection
```
# choose multiple or just one
# note that one overlaps with previous one
ip arp inspection validate src-dst-mac
```


| Command    | Function                    |     |
| ---------- | --------------------------- | --- |
| echo       | Can receive pings           |     |
| echo-reply | Can receive and reply pings |     |
|            |                             |     |



```ios
config t
	access-list 10 permit 10.0.10.0 0.0.0.255 
	access-list 10 permit 10.0.20.0 0.0.0.255
	access-list 10 permit 10.0.30.0 0.0.0.255
	# g0/1 is outside-faced
	ip nat inside source list 10 interface g0/1 overload 
```

```
access-list 100 permit tcp 12.7.26.128 0.0.0.127 any eq 80 
access-list 100 permit tcp 12.7.27.192 0.0.0.63 any eq 80
access-list 100 permit ip 12.7.27.0 0.0.0.127 any
```

#### What protocols should be permitted
- DHCP, HTTPS -> `access-list 100 permit udp any any eq 67|443`
- IP, ICMP, TCP -> `access-list 100 permit ip|icmp|tcp any host 192.168.27.22`
- Established connection with Servers (NAT) :::  `access-list 100 permit tcp any host 172.21.0.1 established` where *172.21.0.1* is the outside NAT interface's ip 
<!--ID: 1652301052035-->


#### What protocols should be denied
- 
	- [ ] dns server
	- [ ] other routers / switches
	- [ ] internet
	- [ ] ip spoofing
		- `access-list 100 deny ip any 12.7.26.0 0.0.254.255`  NAT inside int

## HSRP 
```md
int g0/0/0
	ip add 10.0.0.2
	standby 1 ip 10.0.0.1
	standby 1 priority 200
	standby 1 preempt
```
### Commands
#### priority
- higher is chosen
#### preempt
- if interface is shutdown, then restarted, it automatically comes back alive
- sets the router as the default virtual router  

## Ports
#### Disable unused ports
```markdown
en
	show ip ports all
	show control-plane host open-ports
conf t
	# Disable HTTP
	no ip http server
	line vty 0 15
		transport input ssh
	
```
## SNMP
```
snmp-server community public RO 
snmp-server community private RW 
```
## Troubleshooting
### Protocols
#### SSH
- Old ssh version (pt defaults to 1.99
	- ip ssh version 2
## Resources
[Jeremy Bai CCNA](https://www.youtube.com/playlist?list=PLxbwE86jKRgMpuZuLBivzlM8s2Dk5lXBQ)

### Quizzes
- NAT
	- https://youtu.be/kILDNs4KjYE?t=1318

## References

[^1]: https://youtu.be/kILDNs4KjYE?t=1435
