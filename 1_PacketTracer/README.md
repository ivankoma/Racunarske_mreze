# Cisco Packet Tracer

> My course notes for the [Cisco CCNA Packet Tracer Ultimate labs](https://www.udemy.com/course/cisco-ccna-packet-tracer-ultimate-labs-ccna-exam-prep-labs/) online course.

- `?` help
- `wr` or `copy running-config startup-config` to save the configuration before shutting down the devices (in Packet Tracer this is not necessary)

### Example: How to configure 2 routers to communicate with each other

```
Router1>en
Router1#conf t
Router1(config)#
Router1(config)# host Router1
Router1(config)# int g0/0/1
Router1(config-if)# ip address X.X.X.X M.M.M.M
// Router1(config-if)# ip address dhcp
Router1(config-if)# no shut
```

Switch
```
host Switch1
int vlan 1
no shut
ip address X.X.X.X M.M.M.M
ip default-gateway R.R.R.R
```

```
Router1# sh ip route
Router1# sh ip int brief
```

Enable DNS lookup on the router:

```
enable
config t
ip domain-lookup
ip name-server 8.8.8.8
end
ping cisco.com
```

`show` commands:

```
sh run
sh ip route
sh ip int brief
```

## DHCP

```
R1# conf t
R1(config)# ip dhcp pool pc
R1(dhcp-config)# network 10.1.1.0 255.255.255.0
R1(dhcp-config)# default-router 10.1.1.254
R1(dhcp-config)# dns-server 10.1.1.254
R1(dhcp-config)# exit
R1(config)# ip dhcp excluded-address 10.1.1.1 10.1.1.100
```

Test:

```
R1# sh run
R1# sh ip dhcp binding
R1# sh ip dhcp pool
```

---

## ACL

|Standard|Extended|
|---|---|
|Filters by source address only|Filters by source and destination|
|Permit or deny all IP/TCP|Specify IP, protocol and port number|
|Range 1-99 1300-1999|Range 100-199 2000-2699|


|Port|Protocol|Transport|
|---|---|---|
|20|FTP data|TCP|
|21|FTP command|TCP|
|22|SSH|TCP|
|23|Telnet|TCP|
|25|SMTP|TCP|
|53|DNS|UDP|
|67|DHCP server|UDP|
|68|DHCP client|UDP|
|80|HTTP|TCP|
|109|POP|TCP|
|110|POP3|TCP|
|443|HTTPS|TCP|

Standard ACL:

`access-list NUMBER {permit|deny} SOURCE_IP INVERSE_MASK`

Extended ACL:

`access-list NUMBER {permit|deny} PROTOCOL host SOURCE_IP host DEST_IP eq PORT`

`access-list NUMBER {permit|deny} PROTOCOL SOURCE_IP INVERSE_MASK DEST_IP INVERSE_MASK eq PORT`

`PROTOCOL`: `tcp` includes Telnet and `ip` includes TCP, UDP, ICMP

|Command|Same as|
|---|---|
|access-list 1 permit 0.0.0.0 255.255.255.255|access-list 1 permit any|
|access-list 1 permit 172.30.16.29 0.0.0.0|access list 1 permit host 172.30.16.29|

```
Router1>en
Router1#conf t
Router1(config)#access-list 100 permit tcp host 10.1.2.101 host 10.1.1.100 eq 80
```

Test:
```
Router1(config)#do sh run | i access
or
Router1#sh access-list
```

### Examples


> Devices on subnet 10.1.2.0/24 can't access subnet 10.1.1.0/24

```
Router1(config)#access-list 100 deny ip 10.1.2.0 0.0.0.255 10.1.1.0 0.0.0.255
```

> Hosts on subnet 10.1.2.0/24 can access any other network

```
Router1(config)#access-list 100 permit ip 10.1.2.0 0.0.0.255 any
```

> Any external device can access internal HTTP Servers using HTTP or HTTPS

```
Router1(config)#access-list 101 permit tcp any 10.1.1.0 0.0.0.255 eq 80
```


> Internal PCs can connect to outside DNS server

```
Router1(config)#access-list 101 permit udp host 8.8.8.8 eq 53 10.1.2.0 0.0.0.255
```

---

> Internal PCs can connect to outside servers (servers can return traffic of sessions to the server: webpage or DNS request). Place this before denying all other traffic.

```
Router1(config)#access-list 101 permit tcp any 10.1.2.0 0.0.0.255 established
```

`established` means that `ACK` bit is set in the return traffic.

> No external device can access the user subnet 10.1.2.0/24

```
Router1(config)#access-list 101 deny ip any 10.1.2.0 0.0.0.255
```

### Delete ACL list

```
Router1#sh access-list
Router1#conf t
Router1(config)#ip access-list extended 100
Router1(config-ext-nacl)#no 30 (deletes only one entry)
Router1(config-ext-nacl)#exit
Router1(config)#no access-list 100 (deletes whole ACL)
```

### Bind ACL to port

```
Router1>en
Router1#conf t
Router1(config)#int gigabitEthernet 0/0/0
Router1(config-if)#ip access-group 100 {in|out}
```

### Access to two servers in one ACL

PC: `10.1.2.101`

Server1: `10.1.1.100`

Server2: `10.1.1.101`

```
Router1(config)#access-list 100 permit tcp host 10.1.2.101 10.1.1.100 0.0.0.1 eq 80
```

`0.0.0.1` = `00000000.00000000.00000000.00000001`

Inverse mask: `0` means match, `1` means don't match

Place external access list on the outside interface.

---

## NAT

Test:

```
Router1#sh ip nat translations
Router1# clear ip nat translation *
```

### Static NAT

> 23. NAT Lab 2

```
Router# conf t

// Enable DNS server (why?)
Router(config)#ip route 0.0.0.0 0.0.0.0 DNS_SERVER

// NAT only one required port so that outside devices can access internal HTTP server:
Router(config)#ip nat inside source static tcp PRIVATE_IP PORT PUBLIC_IP PORT

// Full static NAT (all ports)
Router(config)#ip nat inside source static PRIVATE_SERVER_IP EXTERNAL_NAT_IP

Router(config)#int g0/0/0
Router(config-if)#ip nat inside
Router(config)#int g0/0/1
Router(config-if)#ip nat outside
```

### Dynamic NAT

> 24. NAT Lab 3 (so that PCs can access outer Internet)

```
Router(config)#ip nat inside source list 1 interface g0/0/1 overload
Router(config)#access-list 1 permit any
```

|Step|Command|
|---|---|
|1|`Router(config)# ip nat pool <name> <start_ip> <end_ip> {netmask <mask> | prefix-length <prefixlen>}`|
|2|`Router(config)# access-list <acl_id> permit <source> [<source wildcard>]`|
|3|`Router(config)# ip nat inside source list <acl_id> pool <name>`|
|4|`Router(config-if)# ip nat inside`|
|5|`Router(config-if)# ip nat outside`|

### PAT (Overloading)

|Step|Command|
|---|---|
|1|`Router(config)# access-list <acl_id> permit <source> [<source wildcard>]`|
|2a|`Router(config)# ip nat inside source list <acl_id> interface <int> overload` or|
|2b|`Router(config)# ip nat pool <name> <start_ip> <end_ip>{netmask <mask> | prefix-length <prefixlen>}` `Router(config)# ip nat inside source list <acl_id> pool <name> overload`|
|4|`Router(config-if)# ip nat inside`|
|5|`Router(config-if)# ip nat outside`|

> 22. NAT Lab 1 (Basic PAT configuration):

```
Router1# conf t
Router1(config)# int g0/0/0
Router1(config-if)# ip nat inside
Router1(config-if)# int g0/0/1
Router1(config-if)# ip nat outside
Router(config-if)# exit
Router1(config)# ip nat inside source list 1 interface g0/0/1 overload
Router1(config)# access-list 1 permit any
```

---

## Static routes

Adds information about networks that is not directly connected to the router.

```
Route(config)#ip route <NETWORK_IP> <MASK> <NEXT_HOP_IP>
```

Test:

```
Router#sh ip route
```

---

## RIP

```
R1(config)#router rip
R1(config-router)#network 192.168.1.0
R1(config-router)#network 192.168.2.0
R2(config-router)#no auto-summary
R1(config-router)#version 2
R1(config-router)#end
```

Test:

```
R1#sh ip prot
R1#sh ip rip database 
```