**ALL Credit goes to ttmcmurry from https://forum.netgate.com/**

**Source:** https://forum.netgate.com/topic/153288/multiple-ipv6-prefix-delegation-over-at-t-residential-gateway-for-pfsense-2-4-5 


**Multiple IPv6 Prefix Delegation over AT&amp;T Residential Gateway for pfSense 2.4.5** 

I've (ttmcmurry) been working on this one for a while. This is the result of others posting their work across various forums, reading BSD docs, and plenty of testing as a result of needing something to do while being stuck at home. :) 

The purpose of this is to make it easier for AT&T customers who wish to assign more than one IPv6 prefix delegation inside their pfSense firewall to more than one internal network interface. I am providing an example dhcp.conf script and explaining what's needed step-by-step. AT&T customers must have been furnished a Residential Gateway (Pace 5268AC / Arris BGW210-700, possibly others) and have configured the RG in DMZ+/IP Passthrough mode. This has been written with pfSense 2.4.5 in mind. 

Why do this? In short, AT&T U-Verse & Fiber customer equipment is assigned a /60 and can only hand out eight /64 prefix delegations. It is not possible to request a larger PD, however it is possible to request multiple /64 PDs from pfSense's WAN interface. Since the pfSense UI does not expose this functionality directly, it is possible to take advantage of it by supplying a dhcp.conf to override pfSense DHCP6 behavior available from the UI. 

I welcome improvements and feedback and will be happy to update this to help make other's lives easier and work within pfSense native functionality as much as possible. 

Once this script is in place, if you need to reassign interfaces & prefix delegations, the script has to be updated. You will need to edit the IPv6 Track Interface Prefix ID on the LAN/OPT interfaces with the IA-PD you specify in the .conf file. 

# Assumptions

1. The WAN interface IPv6 Configuration type is configured for "none" or, 

2. The WAN interface IPv6 Configuration type is configured for DHCP6 and IPv6 Prefix Delegation is set /60 

3. The WAN interface IPv6 DHCP6 Client Option "Do not allow PD/Address release" is UNCHECKED 

4. The LAN/OPT interfaces' DHCP6 option is set to "none" 

5. DHCPv6 Server & RA -> DHCP6 Server -> Disabled 

6. DHCPv6 Server & RA -> Router Advertisements -> Defaults (Router Mode: Assisted) 

# Steps 
### 1. Make a local copy of the code below 
```
interface *WANInterface* {
	send ia-na 0;
	send ia-pd 0;
	send ia-pd 1;
	send ia-pd 2;
	send ia-pd 3;
	send ia-pd 4;
	send ia-pd 5;
        send ia-pd 6;
	send ia-pd 7;
	request domain-name-servers;
	request domain-name;
	script "/var/etc/dhcp6c_wan_script.sh";
};
id-assoc na 0 { };
id-assoc pd 0 {
	prefix-interface *LANInterface* {
		sla-id 0;
		sla-len 0;
	};
};
id-assoc pd 1 { };
id-assoc pd 2 { };
id-assoc pd 3 { };
id-assoc pd 4 { };
id-assoc pd 5 { };
id-assoc pd 6 { };
id-assoc pd 7 { };

``` 

### 2. Update the "interface" stanza (Line 1)

	- Look at Interfaces -> Assignments -> Network Port for the adapter associated to the WAN interface 

	- Replace the *WANInterface* in the interface stanza (Line 1) below with the WAN adapter network port name below, e.g. hn0, igb0, vmx0, eth0, etc 

	- If using VLANs, remember to use numerical subinterface number e.g. hn0.10 for VLAN 10 

	- IA-NA Note: The IA-NA is an arbitrary number. A unique number must be chosen for each device connected to the AT&T residential gateway (RG) which will request a prefix 	delegation from the RG. If only one device will be requesting PDs from the RG (i.e. this pfSense firewall), then "ia-na 0" is fine. 

### 3. Update the "ia-pd" stanzas

	Look at Interfaces -> Assignments -> Network Port for the adapter associated to each LAN/OPT interface(s) 

	Replace the prefix-interface *LANINTERFACEX* with LAN/OPT adapter network port name 
		[Uncomment Stanzas if you need more than one interface.]

	If using VLANs, remember to use numerical subinterface number e.g. hn0.10 for VLAN 10 

	You can arbitrarily assign adapters to PDs, staring with PD 0 and working down the list 

	SLA-ID is always zero (0) 

	SLA-LEN is always zero (0) 

	Formatting is specific. Each new PD stanza needs to be formatted exactly as PD0; only update the adapter name 

	If you are not assigning an adapter to a PD, leave it blank as shown below 


**Note:** You do not need to request all 8 PDs if you don't need them. You may remove any "send ia-pd" & "id-assoc pd x" statements where you aren't assigning them to interfaces. 

**Note:**  Assigned PDs will result in numerically different networks, depending on the RG 

	Pace 5268AC first assigns F then decrements to 8 to PD 0-7, i.e. PD0 = ::xxxF::/64 

	Arris BGW210-700 first assigns 8 then increments to F to PD 0-7, i.e. PD0 = ::xxx8::/64 


### 4. Add the script to pfSense

	- Create this file on pfSense under Diagnostics -> Edit File 

	- Copy and paste your edited script into the text window 

	- In the grey filename box, enter /usr/local/etc/rc.d/att-rg-dhcpv6-pd.conf 

	- Click on Save 

### 5. Edit the WAN interface

	- Set IPv6 Configuration Type to "DHCP6" (it may already be set, see "assumptions" above) 

	- Under DHCP6 client configuration, change DHCPv6 Prefix Delegation size from 64 to 60

	- Under DHCP6 client configuration, select Configuration Override 

	- Enter the following in Configuration File Override: /usr/local/etc/rc.d/att-rg-dhcpv6-pd.conf 

	- Click on Save and Apply the changes 

	- Step five: Edit the LAN/OPT interface(s), one at a time 

	- Set the IPv6 Configuration Type to "Track Interface" 

	- Set the Track IPv6 Interface -> IPv6 Interface to the WAN's interface name ("WAN" is the default name) 

	- Set the IPv6 Prefix ID to the PD number configured in the .conf file 

	- Click on Save and Apply the changes 

### 6. Enable pfSense DHCPv6 Server & Test

	- For each configured interface.. 

	- DHCPv6 Server & RA -> DHCPv6 Server -> Enable -> Save 

	- DHCPv6 Server & RA -> Router Advertisements -> Router Mode -> Managed 

	- Test a client in each configured, connected network 



# State Limits


AT&T Residential gateways have a state table that is far smaller than pfSense's defaults, which can result in problems once the RG begins tracking more states than available. pfSense should be set to never go above that limit. pfSense will adjust how states are managed based on its default adaptive algorithm from "Firewall Adaptive Timeouts." There is no need to adjust pfSense default Adaptive Timeout behavior, only the maximum number of states pfSesnse can use.

The values below are from known hardware & firmware capabilities. Depending on the # of devices directly plugged into the RG, like U-Verse set-top-boxes and devices NOT behind pfSense, you may need to adjust pfSense's maximum states downward. This information can be found on the RG under Settings -> Diagnostics -> NAT.

* **Pace 5268AC** - Firmware v11.5.1.532678-att - 15460 states max - Set pfSense to 15000 states
* **Arris NVG599** - Firmware v9.2.2h0d79 - 4096 states max - Set pfSense to 3500 states
* **Arris BGW210-700** - Firmware 1.9.16 - 8000 states max - Set pfSense to 7500 states
* **Motorola NVG589** - Firmware ? - 8192 states max - Set pfSense to 7600 states
* **HUMAX BGW320-500** - Firmware 2.10.6 - 8192 states max - Set pfSense to 7600

Set the pfSense state limit in Advanced -> Firewall & NAT -> Firewall Maximum States

Note: If anyone has more up-to-date information about RG firmware and state capabilities, let me know and I'll update this table.
