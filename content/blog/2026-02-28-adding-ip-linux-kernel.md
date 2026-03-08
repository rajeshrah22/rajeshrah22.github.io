+++
title = "Under the Hood: How the Linux Kernel Adds an IP Address"
date = 2026-02-28
description = "A brief dive into the kernel-space journey of adding an IP address, from Netlink sockets to routing table updates."
[taxonomies]
tags = ["linux", "kernel", "networking"]
+++

In nmstate kernel-only mode, we wanted users to be able to control the order of IP addresses in YAML:
```yaml
interfaces:
- name: lo
  type: loopback
  ipv4:
    enabled: true
    address:
    - ip: 127.0.0.1
      prefix-length: 8
    - ip: 127.0.0.2
      prefix-length: 32
  ipv6:
    enabled: true
    address:
    - ip: ::1
      prefix-length: 128
    - ip: ::2
      prefix-length: 128
```

However, when retrieving addresses from the kernel, the order sometimes came back different. After some testing, it became clear that the kernel does its own ordering when inserting IPv4 addresses.

See [nmstate#3083](https://github.com/nmstate/nmstate/issues/3083).

In this post, we will walk through the kernel data structures and C code involved when adding an address to an interface.

### 1. The Netlink Message
The **ip** command crafts a Netlink message of type **RTM_NEWADDR** (Route Message: New Address) containing the IP, subnet, and target interface index. It sends this to the kernel over an **AF_NETLINK** socket.

See **man 7 rtnetlink** for the protocol details.

### 2. Kernel Entry Point
The kernel's **rtnetlink** subsystem receives the message, checks the address family (**AF_INET** for IPv4), and dispatches it to the designated handler: [inet_rtm_newaddr()](https://elixir.bootlin.com/linux/v6.19.3/source/net/ipv4/devinet.c#L960). This function unpacks the Netlink attributes and validates the request.

### 3. Core Data Structures
Before modifying the system, the kernel interacts with three primary networking structures:
* [**struct net_device**](https://elixir.bootlin.com/linux/v6.19.3/source/include/linux/netdevice.h#L1787): The generic hardware or virtual interface (for example, **eth0**). It carries a lot of networking state and is shared by multiple subsystems.
* [**struct in_device**](https://elixir.bootlin.com/linux/v6.19.3/source/include/linux/inetdevice.h#L25): The IPv4-specific configuration block attached to the **net_device**.
* [**struct in_ifaddr**](https://elixir.bootlin.com/linux/v6.19.3/source/include/linux/inetdevice.h#L143): Represents the actual IPv4 address being added, holding the IP, mask, broadcast address, and scope.

### 4. Allocation and Insertion
If the interface is valid, [inet_rtm_newaddr()](https://elixir.bootlin.com/linux/v6.19.3/source/net/ipv4/devinet.c#L960) allocates a new **in_ifaddr** structure using **inet_alloc_ifa()**. Once populated, the kernel calls [inet_insert_ifa()](https://elixir.bootlin.com/linux/v6.19.3/source/net/ipv4/devinet.c#L490) to apply the changes.

The interesting ordering logic lives here.

Let's take a closer look at [__inet_insert_ifa()](https://elixir.bootlin.com/linux/v6.19.3/source/net/ipv4/devinet.c#L490):
```c
static int __inet_insert_ifa(struct in_ifaddr *ifa, struct nlmsghdr *nlh,
                             u32 portid, struct netlink_ext_ack *extack)
{
	struct in_ifaddr __rcu **last_primary, **ifap;
	struct in_device *in_dev = ifa->ifa_dev;
	struct net *net = dev_net(in_dev->dev);
	struct in_validator_info ivi;
	struct in_ifaddr *ifa1;
	int ret;

	ASSERT_RTNL();

	ifa->ifa_flags &= ~IFA_F_SECONDARY;
	last_primary = &in_dev->ifa_list;

	/* Don't set IPv6 only flags to IPv4 addresses */
	ifa->ifa_flags &= ~IPV6ONLY_FLAGS;

	ifap = &in_dev->ifa_list;
	ifa1 = rtnl_dereference(*ifap);

	while (ifa1) {
		if (!(ifa1->ifa_flags & IFA_F_SECONDARY) &&
		    ifa->ifa_scope <= ifa1->ifa_scope)
			last_primary = &ifa1->ifa_next;
		if (ifa1->ifa_mask == ifa->ifa_mask &&
		    inet_ifa_match(ifa1->ifa_address, ifa)) {
			if (ifa1->ifa_local == ifa->ifa_local) {
				inet_free_ifa(ifa);
				return -EEXIST;
			}
			if (ifa1->ifa_scope != ifa->ifa_scope) {
				NL_SET_ERR_MSG(extack, "ipv4: Invalid scope value");
				inet_free_ifa(ifa);
				return -EINVAL;
			}
			ifa->ifa_flags |= IFA_F_SECONDARY;
		}

		ifap = &ifa1->ifa_next;
		ifa1 = rtnl_dereference(*ifap);
	}
	
	// ... truncated ...
	
	if (!(ifa->ifa_flags & IFA_F_SECONDARY))
		ifap = last_primary;

	rcu_assign_pointer(ifa->ifa_next, *ifap);
	rcu_assign_pointer(*ifap, ifa);
	
	// ... truncated ...

	/* Send message first, then call notifier.
	   Notifier will trigger FIB update, so that
	   listeners of netlink will know about new ifaddr */
	rtmsg_ifa(RTM_NEWADDR, ifa, nlh, portid);
	blocking_notifier_call_chain(&inetaddr_chain, NETDEV_UP, ifa);

	return 0;
}
```

### 5. Why the order changes
The linked list **in_dev->ifa_list** is not always append-only. During insertion, the kernel walks the existing list and tracks **last_primary**. Later, if the new address is *not* marked secondary, it inserts at **last_primary** instead of at the tail:

```c
if (!(ifa->ifa_flags & IFA_F_SECONDARY))
	ifap = last_primary;
```

This means insertion order from userspace is not guaranteed to be preserved for IPv4 in all cases. Scope, prefix relationship, and whether an address is treated as primary/secondary can move where a new address lands in the interface list.

### 6. Duplicate checks and scope validation
Inside the same loop, the kernel also:

* rejects exact duplicates with **-EEXIST**
* rejects scope mismatches with **-EINVAL**
* marks matching subnet entries as secondary (**IFA_F_SECONDARY**) when needed

Those checks happen before final insertion, so both correctness and ordering are decided in one pass.

### 7. Routing and notification
After insertion, the kernel emits an **RTM_NEWADDR** notification and triggers notifier callbacks:

```c
rtmsg_ifa(RTM_NEWADDR, ifa, nlh, portid);
blocking_notifier_call_chain(&inetaddr_chain, NETDEV_UP, ifa);
```

This is where other networking components (including FIB/routing consumers) observe the new address and update related state.

### 8. Practical takeaway for nmstate and tooling
If your users care about deterministic YAML order, do not assume the kernel will return IPv4 addresses in the same order they were requested. Instead:

* treat kernel-returned order as kernel-defined
* sort or normalize addresses in userspace before comparing desired vs current state
* compare semantically (address/prefix/scope/flags), not by list position alone

That behavior explains the discrepancy seen in nmstate kernel-only mode and helps avoid false-positive diffs in reconciliation loops.
