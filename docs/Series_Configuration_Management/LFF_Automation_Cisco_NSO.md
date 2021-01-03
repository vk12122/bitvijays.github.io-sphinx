# Network Documentation and Automation

## Network Documentation

### CDP/LLDP Neighbors

Easiest way to gather information on physical connections is to review a device’s CDP or LLDP neighbor table. This list will tell you about directly connected devices and the interfaces used to connected them. It may also include the remote device hostnames, and possibly even model numbers, capabilities, and management IP addresses.

When starting from scratch on a new diagram and an unknown network, begin at a “core” device used to physically connect many other infrastructure pieces. Look at the CDP and LLDP neighbor tables and insert each device into the drawing one by one. Once complete with all entries in the tables on the core device, log into one of the neighbors and rinse/repeat until you reach the edges of your network.

### MAC/ARP Tables

After the CDP and LLDP neighbor tables have been exhausted and you have diagrammed all [infrastructure] devices from them, you will need to move on to tracking devices down using the MAC and ARP tables.

Move back to the core of the network, where you [assumingly] have some routing happening. Begin by looking at the routing table and making note of all the next-hop addresses. If you are running a dynamic routing protocol, pull up a list of the dynamic neighbors. If only static routes are used, look at the config for the static routes. Make a list of the next-hop address, being sure to remove all duplicates. Working on a single next-hop IP at a time, check the ARP entry for that next-hop IP. Note the MAC address tied to that ARP entry and check it against the MAC address table to find out which physical port is used for forwarding traffic to that MAC address. Once found, you now have a next-hop IP address mapped to a physical port. Add that device to the diagram (use the router icon and the IP in the label if you’re not sure about the model and function) and move on to the next next-hop IP. Recurse this process until all routing nodes (next-hop devices) are documented.

### Tracing Cables

Once you have exhausted the first two options, both of which can be done at the comfort of your desk, it’s time to head over to the hot aisle and start tracing those cables. I find it easiest to write notes down onto paper when tracing cables then transcribe those notes onto the diagram when done. Make sure to double-check what you found in the first two steps by tracing the cables to make sure they end up where you think they do.

### Subnets

Subnets are the cornerstone of a logical network diagram. They represent an IP network where nodes can hold L3 addresses and communicate via IP.

#### Logical Diagram Subnets

There are three important pieces of information to hold in a subnet object: VLAN Name, VLAN ID, and assigned IP block in CIDR format. The VLAN name and ID information assume that the subnet is contained within a VLAN on a switch. When this is not the case (like with a point-to-point link between two routers), omit the VLAN ID and name and include only the subnet. These different pieces of information are distinguished on the subnet with different font types.

## Cisco NSO

Creating and configuring network services is a complex task that often requires multiple integrated configuration changes to every device in the service chain. Additionally changes need to be made concurrently across all devices with service implementation being either completely successful or completely removed from the network. All configurations need to be maintained completely synchronized and up to date. NSO solves these problems by acting as interface between the configurators (network operators and automated systems) and the underlying devices in the network.

The typical work flow when using the network CLI in NSO is as follows:

1. All changes are initially made to a (logical) copy of the NSO database of configurations.
2. Changes are validated against network policies and integrity constraints using the "validate" command. Changes can be viewed and verified by the network operator prior to committing them.
3. The changes are committed, meaning that the changes are copied to the NSO database and pushed out to the network. Changes that violate integrity constraints or network policies will not be committed. The changes to the devices are done in a holistic distributed atomic transaction, across all devices in parallell.
4. Changes either succeed and remain committed or fail and are rolled back as a whole returning the entire network to the uncommitted state.

## Installation

```shell
 sh nso-4.0.linux.x86_64.installer.bin $HOME/nso-4.0
```

```shell
source $HOME/ncs-VERSION/ncsrc
```

Create a runtime directory for NSO to keep its database, state files, logs and other files. In these instructions we assume that this directory is $HOME/ncs-run.

```shell
 ncs-setup --dest $HOME/ncs-run
```

Start the NSO daemon ncs.

```shell
 cd $HOME/ncs-run $ ncs
```

Start the NSO CLI as user "admin" with a Cisco XR style CLI:

```shell
ncs_cli -C -u admin
```

Note the use of the -u parameter which tells NSO which user to authenticate towards NSO.  This user must be configured in NSO AAA (Authentication, Authorization and Accounting).

 there is an operational mode and a configuration mode. Show commands displays different data in those modes. A show in configuration mode displays network configuration data from the NSO configuration database, the CDB. Show in operational mode shows live values from the devices and any operational data stored in the CDB. The CLI starts in operational mode.

NSO organize all managed devices as a list of devices. The path to a specific device is devices device DEVICE-NAME.

## Authentication Groups

When NSO connects to a managed device, it requires the SSH authentication information for that device.

```shell
show full-configuration devices authgroups
```

Add a new user password to connect to devices?

```shell
 devices authgroups group default default-map remote-name Username remote-password Password
```

Password - it's better to put in " "

### Add a device

```shell
devices device Device_Name address Device_IP authgroup default
```

authgroup (default) can be different depending what's the name of authgroup.

```shell
 device-type cli ned-id cisco-ios
 state admin-state unlocked
 commit
```

### Delete the device (not added in groups)

```shell
 no devices device <device-name>
```

After adding, we need to do

```shell
 ssh fetch-host-keys
```

and

```shell
 sync-from
```

Refer [Creating your First Network](https://github.com/NSO-developer/nso-5-day-training/blob/master/Day1Labs/lab1_Creating_Your_First_Network.md)

### Show packages shows the packages NED loaded

```shell
 packages reload
```

### To see the device running config

```shell
 show running-config devices device
```

### To connect all the devices

```shell
 devices connect
```

The following command will synchronize the configurations of the devices with the CDB and respond with "true" if successful:

```shell
 admin@ncs# devices sync-from sync-result {    device c0    result true }....
```

### View the configuration of the "c0" device using the command

```shell
admin@ncs# show running-config devices device c0 config
```

### Show a particular piece of configuration from several devices

```shell
admin@ncs# show running-config devices device c0..2 config ios:router
```

```shell
admin@ncs# show running-config devices device config ios:router | display xml | save router.xml
```

The above command shows the router config of all devices as xml and then saves it to a file router.xml

### Show list of devices

```shell
show devices list
```

### Sync all devices (together)

```shell
 devices sync-from
```

### Fetch SSH-Keys (together)

```shell
 devices fetch-ssh-host-keys
```

### Do commit

```shell
commit label labelname
```

### Perform rollback

```shell
do file show logs/rollback<Tab>
rollback configuration rollbacknumber
commit
```

### Backup

```shell
ncs-backup (works in system install)
```

The backup will be stored in Run Directory `/var/opt/ncs/backups/ncsVERSION@DATETIME.backup`

### NSO Restore

First stop NSO

```shell
 /etc/init.d/ncs stop
```

Then take the backup

```shell
 ncs-backup --restore
```

Select the backup to be restored from the available list of backups. The configuration and database with run-time state files are restored in /etc/ncs and /var/opt/ncs. • Start NSO.

```shell
 /etc/init.d/ncs start
```

### Commit list NSO

```shell
 show configuration commit list
```

### Show configuration changes

::

 show configuration commit changes <rollback_id>

### Show rollback changes

```shell
 show configuration rollback changes <rollback_id>
```

### Rename device after adding it into NSO

```shell
  rename devices device ce2 ce022
  commit no-networking
```
