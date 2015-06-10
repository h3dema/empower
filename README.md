EmPOWER
=======

EmPOWER is a Software Defined Enterprise WLAN framework. EmPOWER provides a
WiFi datapath implementation, a reference SD-WLAN controller, and an SDK for 
developing enterprise WLAN applications.

Requirements
-----------
- one server-class machine running a recent Linux distribution (I'm currently 
using Fedora 22 and Debian Wheezy). 
- one or more WiFi Access Points capable of running OpenWRT Barrier Breaker 
(14.07) with at least one Atheros-based Wireless NIC supported by the ath9k 
driver.

Building EmPOWER from sources
---------------------

Clone the EmPOWER repository and pull the submodules:

```
  $: git clone http://github.com/rriggio/empower
  $: cd empower
  $: git submodule init
  $: git submodule update
```

Change to the "empower-openwrt" directory and update and install all feeds:

```
  $: cd empower-openwrt
  $: ./scripts/feeds update -a
  $: ./scripts/feeds install -a
```

EmPOWER Access Points need to connect to the controller for all their 
operations. However, the default configuration does not specify the address
of the controller. In order to avoid having to configure manually all access
points, you can automatically create an image with the configuration that
suits your needs. NOTE: the following configuration is for a PCEngines ALIX 2D
board equipped with a single Wireless NIC.

In order to do so, create a directory named "files":

```
  $: mkdir files
```

The OpenWRT buildroot will copy all the content of the "files" directory to the
files image. 

Create a "etc/config" directory:

```
  $: mkdir -p files/etc/config
```

Then create the following files:

 - EmPOWER configuration (etc/config/empower)

```
config empower general
    option "master_ip" "<IP ADDRESS OF YOUR CONTROLLER>"
    option "master_port" "4433"
    option "network" "wan"
    list "ifname" "empower0"
    list "debugfs" "/sys/kernel/debug/ieee80211/phy0/ath9k/bssid_extra"
```

 - Network configuration (etc/config/network)

```
config interface 'loopback'
	option ifname 'lo'
	option proto 'static'
	option ipaddr '127.0.0.1'
	option netmask '255.0.0.0'

config device
	option name 'br-ovs'
	option type 'ovs'
	list ifname 'eth0'

config interface 'wan'
	option ifname 'br-ovs'
	option proto 'dhcp'

config interface lan
	option ifname	eth1
	option proto	static
	option ipaddr	192.168.250.1
	option netmask	255.255.255.0
```

 - Wireless configuration (etc/config/wireless)

```
config wifi-device  radio0
        option type     mac80211
        option channel  36
        option hwmode   11a
        option path     'pci0000:00/0000:00:0c.0'

config wifi-iface empower0
        option device   radio0
        option mode     monitor
        option ifname   moni0
```

Run the configuration application:

```
  $: make menuconfig
```

From the menu select the Target System (e.g. x86) and the Subtarget (e.g. 
PCEngines alix2). Then select "Network -> empower-agent" and 
"Network -> openvswitch-switch".

Save the configuration and exit, then start the compilation. If you have a 
multi core machine you can increase the compilation speed by increasing the
number of parallel builds with th "-j" option.

```
  $: make -j 2
```

Once the compilation is done, the compiled image can be found in the 
bin/<target> directory. For example in the case of a PCEngines Alix platform,
the image will be:

```
  bin/x86/openwrt-x86-alix2-combined-squashfs.img
```

The method of loading the image on the wireless router (flashing) changes 
according to the brand/model of the wireless router. You can find the flashing 
instruction for all supported models [here](http://wiki.openwrt.org/toh/start). 

In the case of the PCEngines Alix board you need to insert the Compact Flash
card in a compact flash reader attached to your laptop and then run:

```
  $: dd if=./bin/x86/openwrt-x86-alix2-combined-squashfs.img of=/dev/sdb
```

Note this assumes that the compact flash device is "/dev/sdb"

Running EmPOWER
---------------

The EmPOWER WLAN controller must be executed from a central server that can be 
reached from all wireless APs. From the main repository enter the controller
directory:

```
  $: cd empower-controller
```

and start the controller:

```
  $: ./empower.py
```

If all the python dependencies are satisfied you should see an output like
this:

```
INFO:root:Importing module: empower.maps.ncqm
INFO:root:Importing module: empower.triggers.summary
INFO:root:Importing module: empower.maps.ucqm
INFO:root:Importing module: empower.events.wtp_up
INFO:root:Importing module: empower.counters.bytes_counter
INFO:root:Importing module: empower.triggers.rssi
INFO:root:Importing module: empower.counters.packets_counter
INFO:root:Importing module: empower.events.wtp_down
INFO:root:Importing module: empower.events.lvap_join
INFO:root:Importing module: empower.link_stats.link_stats
INFO:core:Registering 'empower.lvapp.lvappserver'
INFO:lvapp.lvappserver:EmPOWER LVAPP Server available at 4433
INFO:core:Registering 'empower.web.restserver'
INFO:web.restserver:EmPOWER LVAPP Server available at 8888
INFO:core:Registering 'empower.energino.energinoserver'
INFO:energino.energinoserver:EmPOWER Energino Server available at 5533
INFO:core:Registering 'empower.maps.ucqm'
INFO:core:Registering 'empower.maps.ncqm'
INFO:core:Registering 'empower.link_stats.link_stats'
INFO:core:Registering 'empower.triggers.summary'
INFO:core:Registering 'empower.triggers.rssi'
INFO:core:Registering 'empower.events.wtp_up'
INFO:core:Registering 'empower.events.wtp_down'
INFO:core:Registering 'empower.events.lvap_join'
INFO:core:Registering 'empower.counters.packets_counter'
INFO:core:Registering 'empower.counters.bytes_counter'
```

The controller web interface can be reached at the following URL:

```
http://127.0.0.1:8888/
```

Configuring the Controller
--------------------------

The first time the controller is initialized, three accounts are automatically
created:

```
USERNAME PASSWORD
root     root
foo      foo
bar      bar
```
The first account has administrator privileges, while the other two are user
account.

Log in as root and click on the WTP tab (WTP stands for Wireless Termination 
Points and, in the EmPOWER terminology, is the equivalent to a Wireless AP).

You must then specify the MAC addresses of the WTPs that are authorized to
connect to this controller. You can find the MAC address by logging into the
WTPs using ssh and by using the ifconfig command. The EmPOWER platform uses
the MAC address of the first wireless interface (typically moni0). Add all your
WTPs using this procedure.

After all WTPs have been added and if the controller is reachable by the WTPs on 
the ip address specified in the "etc/config/empower" then you should see an 
output like this in the controller terminal:

```
INFO:lvapp.lvappserver:Incoming connection from ('192.168.0.133', 54476)
INFO:lvapp.lvappserver:Hello from 192.168.0.133 seq 46852
INFO:lvapp.lvappserver:Sending caps request to 04:F0:21:09:F9:93
INFO:lvapp.lvappserver:Received caps response from 04:F0:21:09:F9:93
```

Creating your first Slice
-------------------------

The SD WLAN Controller can run multiple virtual networks, or Pools, on top of 
the same physical infrastructure. A Pool is a virtual network with its own set 
of WTPs. Network Apps run on top of the SD WLAN Controller in their own Pool 
and exploit the programming primitives through either a REST API or a native 
Python API.

Pools are requested by regular users and must be apporved by a network 
administrator. You can request your first slice using the controller web 
interface by logging with one of the default user account (e.g. foo).

After logging you will be presented with the list of your Pools (which should
be empty). Click on the "+" icon and then specify a name (this will be your
WLAN SSID), an optional short description, and select at least one of the WTPs
in the list. The click on the button "Request pool".

You should now logout from the web interface and login again as administrator.
Click on the "Request" tab and approve the request.

Finally, the opensource version of EmPOWER does not support WPA authentication,
so you must specify the list of MAC address that can connect to any of the 
Pool defined in the controller (by default anybody can associate). In order to
do so click on the "ACL" tab and specify the clients MAC addresses that are 
allowed and/or denied to use the network.

At this point any of the clients that are authorized to use the network should
see the new SSID and should be able to associate to it and to reach the public
Internet (if the wireless APs have a backhaul that is connected to the 
Internet).

Please refer to the wiki for the REST and Python API documentation and in 
general for the documentation on how to develop applications.

