# Azure VPN ECMP Testing Report

Azure VPN gateway support high redundancy architecture that customer can have multiple devices connect to Azure VPN via multiple tunnels. You can refer Azure [documentation]( https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-highlyavailable#activeactiveonprem) for more details. <br>
This document recorded my testing steps when I simulate this topology at Global Azure. There are some tips and lesson learn. <br>

## Topology

![](https://github.com/yinghli/AzureVPNECMP/blob/master/ECMP.jpg)

We use this topology to simulate high available customer scenarios. VNET2 is customer on Azure resource. It includes one VPN gateway and IP prefix 10.2.0.0/16. VNET3 and VNET33 simulate customer two on premise VPN devices but share with same customer IP prefix 10.3.0.0/16. Detail setup PowerShell can be refer [here](https://github.com/yinghli/AzureVPNECMP/blob/master/vpn1.txt).<br>

Parameters      | Azure          |OnPrem1       |OnPrem2    
----------------| -------------  |-----------   |---------     
BGP ASN         |65002           |65003         |65003         
BGP Public IP   |40.83.97.14     | 13.70.19.201 |13.75.118.3
BGP peer IP     |10.2.255.254    |10.3.255.254  |10.3.254.254    
Local Network   |10.2.0.0/16     |10.3.0.0/16   |10.3.0.0/16  

```
Tips:
1.	VNET3 and VNET33 should have same IP range. VM setup should have same IP address.
2.	Gateway subnet should be different. VPN gateway BGP ASN should be same. 
```

## Setup Verification 
After IPsec tunnle is up, you can check VPN gateway BGP status via PowerShell. <br>
```
Get-AzVirtualNetworkGatewayBGPPeerStatus -VirtualNetworkGatewayName vnet2gw -ResourceGroupName testrg2

LocalAddress      : 10.2.255.254
Neighbor          : 10.3.254.254
Asn               : 65003
State             : Connected
ConnectedDuration : 02:52:19.1202268
RoutesReceived    : 1
MessagesSent      : 199
MessagesReceived  : 202

LocalAddress      : 10.2.255.254
Neighbor          : 10.3.255.254
Asn               : 65003
State             : Connected
ConnectedDuration : 00:32:44.5785787
RoutesReceived    : 1
MessagesSent      : 2341
MessagesReceived  : 220
```

After BGP is up, you can check routing information via PowerShell. IP prefix 10.3.0.0/16 is learned from two BGP peer that represent customer on prem VPN devices.
```
Get-AzVirtualNetworkGatewayLearnedRoute -VirtualNetworkGatewayName vnet2gw -ResourceGroupName testrg2
...
LocalAddress : 10.2.255.254
Network      : 10.3.0.0/16
NextHop      : 10.3.255.254
SourcePeer   : 10.3.255.254
Origin       : EBgp
AsPath       : 65003
Weight       : 32768

LocalAddress : 10.2.255.254
Network      : 10.3.0.0/16
NextHop      : 10.3.254.254
SourcePeer   : 10.3.254.254
Origin       : EBgp
AsPath       : 65003
Weight       : 32768
...
```

We setup one sender VM2(10.2.0.4/24) at VNET2, one receiver VM3(10.3.0.4/24) at VNET3 and one receiver VM33(10.3.0.4/24) at VNET33. Those two receivers have same IP address and will check traffic distribution in our next testing. If check VM2 effective route table, you will see “10.3.0.0/16” have two next hops. Both are pointed to VPN gateway public IP address. <br>
![](https://github.com/yinghli/AzureVPNECMP/blob/master/RT.jpg)

PING from VM2 to 10.3.0.4 is OK. Either VM3 or VM33 can reply this PING.<br>

But PING from VM3 or VM33 to VM2 may be failed. This is by designed in our topology, cause return traffic may be routed via different way. <br>

## Testing Scenarios 1 : Single sender with multiple TCP flows.

On both receiver side, run `iperf -s -D` and `iperf -s -u -D`. iperf server will listen on tcp port 5001 and udp port 5001.<br>
On sender side, run `iperf -c 10.3.0.4 -b 1m -P 8` to send 8 concurrent flows, each flow sends with 1Mbps bandwidth. <br>
From the result, we saw that 8 flows distribute into 2 IPSec tunnels. Each receiver has 4 flows. <br>

![](https://github.com/yinghli/AzureVPNECMP/blob/master/TCP.jpg)

## Testing Scenarios 2 : Single sender with multiple UDP flows.

For udp testing, on sender side, run ` iperf -c 10.3.0.4 -b 1m -P 8 -u -l 1400`. Sender send 8 concurrent flows, each flow with 1Mbps bandwidth. Please note that packet length is 1400 bytes, this will avoid IP fragmentation at IPSec VPN gateway. <br>
From the result, we saw that 8 flows are all in one IPSec tunnel. One receiver has 8 flows, the other has 0 flows.<br>

![](https://github.com/yinghli/AzureVPNECMP/blob/master/UDP.jpg)

## Testing Scenarios 3: Multiple sender with multiple TCP/UDP flows.

To check UDP flows load sharing in step 2, we simulate multiple sender and receiver with mixed TCP and UDP flows. On sender and receiver side, we add multiple secondary IP address. One sender side, we also add second NIC in the system. <br>
Sender = ('10.2.0.5' '10.2.2.4' '10.2.0.6' '10.2.0.8' '10.2.2.5' '10.2.2.6' '10.2.2.7' '10.2.0.9') <br>
Receiver = ('10.3.0.4' '10.3.0.5') <br>
Protocol = ('TCP' 'UDP') <br>
We can simulate total 8 * 2 * 2 = 32 flows <br>
