# Physical Networking Ideal POC Setup #

## Terminology ##
* vDS/vSS or nVDS : Is the network/vlan controlled by vsphere (vDS/vSS) or nVDS (logical within NSX-T)
* VTEP - The virtual tunnel network created by NSX that creates the overlay.

## VLANS ##

* Two vlans are required.  They can be existing or dedicated.
	* VTEP - VLAN
		* Routable? : No
		* Size : /28
		* vDS/vSS or nVDS : both
			* Edge devices need access to this vlan
		* Used for the overlay network for the ESX hosts and Edge devices to create the VTEP.
		* Must support >=1600 MTU
		* Trunked or access port connected to a dedicated VMNIC on each host
		* Dedicated /28 is desirable or enough IP addresses for each host and edge device.
	* Edge Uplink - VLAN2
		* Routable : Yes
		* Size : /29
		* vDS/vSS or nVDS : vDS/vSS
		* Used for the uplink between the Edge devices and the physical switch
		* Dedicated /29 is desirable or enough IP addresses for each Edge device +1 for the HA VIP.

## PKS Networks ##

* PKS-MGMT
	* Routable? : Optional, but YES preferred.
		* If not routed, will have to manually create NAT rules within NSX
	* Size : /28 or 10 IP addresses
	* vDS/vSS or nVDS : nVDS
	* If routed, static route from physical switch to Edge HA VIP
	* Used for PKS control plane virtual machines
* PKS-Nodes
	* Routable? : Optional
		* Usually non-routable for a POC.
	* Size : /16
	* vDS/vSS or nVDS : nVDS
	* If routed, static route from physical switch to Edge HA VIP
	* Used for kubernetes master and worker nodes
	* Each cluster created will be assigned a /24
* PKS-Pods
	* Routable? : Optional
		* Usually non-routable for a POC
	* Size : /16
	* vDS/vSS or nVDS : nVDS
	* If routed, static route from physical switch to Edge HA VIP
	* Used for kubernetes pods/containers
	* Each kubernetes namespace will be assigned a /24
* PKS-VIP
	* Routable? : Yes
	* Size : /24
	* vDS/vSS or nVDS : nVDS
	* Static route from physical switch to Edge HA VIP
	* Used for:
		* NAT in non-routable environments
		* Kubernetes services and ingress