# Azure VPN ECMP Testing Report

Azure VPN gateway support high redundancy architecture that customer can have multiple devices connect to Azure VPN via multiple tunnels. You can refer Azure [documentation]( https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-highlyavailable#activeactiveonprem) for more details. <br>
This document recorded my testing steps when I simulate this topology at Global Azure. There are some tips and lesson learn. <br>

## Topology

![](https://github.com/yinghli/AzureVPNECMP/blob/master/ECMP.jpg)

We use this topology to simulate high available customer scenarios. VNET2 is customer on Azure resource. It includes one VPN gateway and IP prefix 10.2.0.0/16. VNET3 and VNET33 simulate customer two on premise VPN devices but share with same customer IP prefix 10.3.0.0/16.<br>

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


