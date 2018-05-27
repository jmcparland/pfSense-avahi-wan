# pfSense-avahi-wan
Modify pfSense avahi installation to allow mDNS repeating from WAN to LAN.
## Problem Statement
I have a multi-VLAN network with an EdgeRouter-X at the root.  One VLAN, C, has several Chromecast devices on it.  Another VLAN, L, serves as the WAN for different lab networks, each isolated by a pfSense instance.  I would like wireless clients on one of the lab networks, N, to be able to cast to one particular Chromecast.
## Solution Preliminaries
Most of this problem is solved with general research.
1. On the EdgeRouter, enable the mDNS repeater for interfaces C and L. There is no top-level configuration for this service; rather, it is done through the configuration tree.  From the pfSense router, you should see 224.0.0.x traffic appearing and being blocked right away.
2. It's helpful to have a static address for the target Chromecast.  On the EdgeRouter, open a path and verify that you can ping the Chromecast from the pfSense instance.  Depending on your network, this may involve an inbound rule on the L interface and an outbound rule on the C interface.
3. Install the avahi package on the pfSense.  Avahi is an mDNS repeater/reflector implementation, just like on the EdgeRouter.  Here, though is the issue:  The avahi package, as beautiful as the interface is, is hard-coded to exclude the WAN interface as an option -- you know, because why in the world would anyone want to do that? :-p

## Proof of Concept
Here's where things start to get ugly.  This is a hack, not a final solution, but someone with the skills can take it the last few yards to alter the configuration or to repackage the hack as a modified package.  
1. This hack is going to involve modifying two files on the pfSense box itself.  You're going to need root access.  Either enter through the console, or go through the gyrations of enabling ssh access, installing the pfSense sudo package, and adding your account (on the pfSense sudo configuration window, to be clean) as a sudo user.
2. Navigate to the /usr/local/pkg/ directory on the pfSense.  Make backups of the two avahi files
   - avahi.xml which controls some interface parameters, and
   - avahi.inc which contains some software logic.
3. Modify the xml file to remove showing LAN as an option to exclude.  The designer's presumption is that we will use two or more LAN/VLAN interfaces and repeat multicast traffic between them.  In that use case, it makes sense that we might want to exclude some interfaces from repeating.  That's not *our* use case, though.
   - Find and change the line "<hideinterfaceregex>(wan|loopback)</hideinterfaceregex>" 
   - to "<hideinterfaceregex>(wan|*lan*|loopback)</hideinterfaceregex>"
4. Comment out the offending assumption in the inc file's software.  If you have trouble finding it, look for the developer's comment, "// Never pass along WAN. Bad." -- LOL!
   - //$denyinterfaces = $config['interfaces']['wan']['if'];
5. In pfSense, refresh the avahi configuration page.  If you have only WAN & LAN ports in play, you won't see anything in the option to exclude interfaces box. (It used to have "LAN" as an option.)
6. Start the avahi service.  You should see that multicast traffic is no longer being blocked at the WAN interface.
7. That's it.  Join the L network and see if the Chromecast is discovered.

## One or two more things?
1. In my case, my client computer was operating wirelessly, with a separate VLAN & RADIUS configuration putting connections on the LAN-side of the Lab pfSense instance.  In a case like this, make sure your wireless access point is not blocking the multicast traffic -- at least from your selected devices.  (On an Ubiquiti system, this involves using the controller to add the Chromecast's MAC address to broadcast whitelist.
2. Make sure that your device can contact (ping) the Chromecast from the Lab's LAN-side of the pfSense.  Check outbound pfSense LAN rulee, primary firewall/router rules (EdgeRouter, in my case), etc.
3. In my case, I am allowing traffice from the Lab LAN to the pfSense to be NAT'ed by the pfSense without any difficulty. If you like, you may add NONAT rules for the outbound traffic to the Chromecast; just make sure you remember to insert a static route or similar so that there is a return bath back to the LAN network. X-)

Hope that helps!
