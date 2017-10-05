Connect overlap subnet between Azure and on premise using VxLAN
==============================

Background<br>
--------------------
When customer migrate their application from on premise data center to Azure data center, it may have a short time that same IP subnet in both Azure and on premise side.<br>
By design, Azure didn’t support overlapping IP subnet. Below is a short term workaround by using VxLAN to bridge same IP subnet in both side.<br>

Topology<br>
-------------------
Customer have one subnet 10.2.0.0/24 on premise and Azure.<br>
Azure VM1 have IP 10.2.0.4/24, VM2 have IP 10.2.0.6/24.<br>
On premise VM3 have IP 10.2.0.10/24.<br>

![](https://github.com/yinghli/CSR1000v-VxLAN/blob/master/topology.png)


Design Guideline<br>
-------------------------
Azure network use overlay technology. GRE is tunnel protocol using by Azure.<br>
VxLAN or IPSec is the option that can be used by customers.<br> 
IPSec encryption and decryption will cost more compute cycle and add latency.<br> 
VxLAN is a UDP encapsulation technology and can be support both commercial and open source software.<br>
This design is using VxLAN as overlay technology and Cisco CSR1000v as commercial solution.<br>

Configuration<br>
----------------------
On premise side, CSR1000v (On Premise) will act as subnet 10.2.0.0/24 gateway and VxLAN tunnel.<br>
Interface G1 will have 10.2.0.1/24 and VM3 will point this IP as default gateway.<br> 
Interface G2 is internet facing and can be 1:1 NAT to public IP address.<br>
Tunnel interface is used to encapsulate packet with VxLAN.<br>

On Premise CSR1000v sample configuration:
VxLAN tunnel setup:
```
interface Tunnel2
 ip address 10.0.0.1 255.255.255.0
 tunnel source <G1 interface IP>
 tunnel mode vxlan-gpe ipv4
 tunnel destination <Azure CSR1000v primary interface public IP>
```
Point to Azure VM Static route
```
ip route 10.2.0.4 255.255.255.255 Tunnel2
ip route 10.2.0.6 255.255.255.255 Tunnel2
```
On Azure side, CSR1000v (Azure) will act VxLAN tunnel role.<br>
Interface G1 will be primary interface that can be accessed from internet.<br> 
Interface G2 is VM inside facing.<br>
Tunnel interface is used to encapsulate packet with VxLAN.<br>
UDR will apply in subnet 10.2.0.0/24, destination is VM IP in on premise side (VM3: 10.2.0.10/32), next hop is interface G2 ip address.<br>
Azure CSR1000v sample configuration<br>
```
interface Tunnel2
 ip address 10.0.0.2 255.255.255.0
 tunnel source <G1 interface IP>
 tunnel mode vxlan-gpe ipv4
 tunnel destination <On Premise CSR1000v public IP>
```
Routing configuration<br>
```
ip route 10.2.0.10 255.255.255.255 Tunnel2
```
After setup VxLAN, ping from both side CSR1000v to the other will be ok.<br>
Also, ping from VM1/VM2 to VM3 will be ok.<br>

Insight<br>
-----------------
In this case, we use VxLAN general protocol extension (GPE) instead of VxLAN.<br>
VxLAN GPE is design for encapsulate other protocol inside VxLAN, but without complex configuration.<br>
Detail information https://www.ietf.org/id/draft-ietf-nvo3-vxlan-gpe-04.txt <br>
If look inside VxLAN-GPE packet format, it encapsulates ICMP directly into VxLAN.Don’t need to assign VNI information.<br>
From CSR1000v configuration, just setup tunnel interface doesn’t need to take care inner packet MAC lookup.<br>
![](https://github.com/yinghli/CSR1000v-VxLAN/blob/master/packet.png)

Host ARP<br>
------------------------
By default, if VM1/VM2 want to talk with VM3 in same subnet, it will generate ARP for querying destination MAC.<br>
After we test it, Windows and CentOS can send the packet with default gateway MAC instead of destination MAC.<br> 
Ubuntu will not use default gateway MAC, we must add static ARP in this case.<br> 
Azure default behavior:
Azure VM sent traffic with default destination MAC (1234.5678.9abc)<br>
All packet sent to VM with default destination MAC (547f.ee75.b5bc)<br>

