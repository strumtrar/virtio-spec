# 1. Introduction

This document describes the overall requirements for virtio net device
improvements for upcoming release 1.4. Some of these requirements are
interrelated and influence the interface design, hence reviewing them
together is desired while updating the virtio net interface.

# 2. Summary
1. Device counters visible to the driver
2. Low latency tx and rx virtqueues for PCI transport
3. Virtqueue notification coalescing re-arming support
4  Virtqueue receive flow filters (RFF)
5. Device timestamp for tx and rx packets
6. Header data split for the receive virtqueue

# 3. Requirements
## 3.1 Device counters
1. The driver should be able to query the device and/or per vq counters for
   debugging purpose using a virtqueue directly from driver to device for
   example using a control vq.
2. The driver should be able to query which counters are supported using a
   virtqueue command, for example using an existing control vq.
3. If this device is migrated between two hosts, the driver should be able
   get the counter values in the destination host from where it was left
   off in the source host.
4. If a virtio device is a group member device, it must be possible to query
   all of the group member counters via the group owner device.
5. If a virtio device is a group member device, it must be possible to query
   all of the group member counter attributes via the group owner device.

### 3.1.1 Per receive queue counters
1. le64 rx_oversize_pkt_errors: Packet dropped due to receive packet being
    oversize than the buffer size
2. le64 rx_no_buffer_pkt_errors: Packet dropped due to unavailability of the
    buffer in the receive queue
3. le64 rx_gso_pkts: Packets treated as receive GSO sequence by the device
4. le64 rx_pkts: Total packets received by the device

### 3.1.2 Per transmit queue counters
1. le64 tx_gso_pkts: Packets send as transmit GSO sequence
2. le64 tx_pkts: Total packets send by the device

### 3.1.3 More counters
More counters discussed in [1].

[1] https://lists.oasis-open.org/archives/virtio-comment/202308/msg00176.html

## 3.2 Low PCI latency virtqueues
### 3.2.1 Low PCI latency tx virtqueue
0. Design goal
   a. Reduce PCI access latency in packet transmit flow
   b. Avoid O(N) descriptor parser to detect a packet stream to simplify device
      logic
   c. Reduce number of PCI transmit completion transactions and have unified
      completion flow with/without transmit timestamping
   d. Avoid partial cache line writes on transmit completions

1. Packet transmit descriptor should contain data descriptors count without any
   indirection and without any O(N) search to find the end of a packet stream.
   For example, a packet transmit descriptor (called vnet_tx_hdr_desc
   subsequently) to contain a field num_next_desc for the packet stream
   indicating that a packet is located in N data descriptors.

2. Packet transmit descriptor should contain segmentation offload-related fields
   without any indirection. For example, packet transmit descriptor to contain
   gso_type, gso_size/mss, header length, csum placement byte offset, and
   csum start.

3. Packet transmit descriptor should be able to place a small size packet that
   does not have any L4 data after the vnet_tx_hdr_desc in the virtqueue memory.
   For example a TCP ack only packet can fit in a descriptor memory which
   otherwise consume more than 25% of metadata to describe the packet.

4. Packet transmit descriptor should be able to place a full GSO header (L2 to
   L4) after header descriptor and before data descriptors. For example, the
   GSO header is placed after struct vnet_tx_hdr_desc in the virtqueue memory.
   When such a GSO header is positioned adjacent to the packet transmit
   descriptor, and when the GSO header is not aligned to 16B, the following
   data descriptor to start on the 8B aligned boundary.

5. An example of the above requirements at high level is:

```
struct virtio_packed_q_desc {
   /* current desc for reference */
   u64 address;
   u32 len;
   u16 id;
   u16 flags;
};

/* Constant size header descriptor for tx packets */
struct vnet_tx_hdr_desc {
   u16 flags; /* indicate how to parse next fields */
   u16 id; /* desc id to come back in completion */
   u8 num_next_desc; /* indicates the number of the next 16B data desc for this
		      * buffer.
		      */
   u8 gso_type;
   le16 gso_hdr_len;
   le16 gso_size;
   le16 csum_start;
   le16 csum_offset;
   u8 inline_pkt_len; /* indicates the length of the inline packet after this
		       * desc
		       */
   u8 reserved;
   u8 padding[];
};

/* Example of a short packet or GSO header placed in the desc section of the vq
 */
struct vnet_tx_small_pkt_desc {
   u8 raw_pkt[128];
};

/* Example of header followed by data descriptor */
struct vnet_tx_hdr_desc hdr_desc;
struct vnet_data_desc desc[2];

```

6. Ability to zero pad the transmit completion when the transmit completion is
   shorter than the CPU cache line size.

7. Ability to write per packet timestamp and also write multiple
   transmit completions using single PCIe transcation.

8. A generic feature of the virtqueue, to contain such header data inline for virtio
   devices other than virtio-net.

9. A flow filter virtqueue also similarly need the ability to inline the short flow
   command header.

### 3.2.2 Low latency rx virtqueue
0. Design goal:
   a. Keep packet metadata and buffer data together which is consumed by driver
      layer and make it available in a single cache line of cpu
   b. Instead of having per packet descriptors which is complex to scale for
      the device, supply the page directly to the device to consume it based
      on packet size
1. The device should be able to write a packet receive completion that consists
   of struct virtio_net_hdr (or similar) and a buffer id using a single DMA write
   PCIe TLP.
2. The device should be able to perform DMA writes of multiple packets
   completions in a single DMA transaction up to the PCIe maximum write limit
   in a transaction.
3. The device should be able to zero pad packet write completion to align it to
   64B or CPU cache line size whenever possible.
4. An example of the above DMA completion structure:

```
/* Constant size receive packet completion */
struct vnet_rx_completion {
   u16 flags;
   u16 id; /* buffer id */
   u8 gso_type;
   u8 reserved[3];
   le16 gso_hdr_len;
   le16 gso_size;
   le16 csum_start;
   le16 csum_offset;
   u16 reserved2;
   u64 timestamp; /* explained later */
   u8 padding[];
};
```
5. The driver should be able to post constant-size buffer pages on a receive
   queue which can be consumed by the device for an incoming packet of any size
   from 64B to 9K bytes.
6. The device should be able to know the constant buffer size at receive
   virtqueue level instead of per buffer level.
7. The device should be able to indicate when a full page buffer is consumed,
   which can be recycled by the driver when the packets from the completed
   page is fully consumed.
8. The device should be able to consume multiple pages for a receive GSO stream.

## 3.3 Virtqueue notification coalescing re-arming support
0. Design goal:
   a. Avoid constant notifications from the device even in conditions when
      the driver may not have acted on the previous pending notification.
1. When Tx and Rx virtqueue notification coalescing is enabled, and when such
   a notification is reported by the device, the device stops sending further
   notifications until the driver rearms the notifications of the virtqueue.
2. When the driver rearms the notification of the virtqueue, the device
   to notify again if notification coalescing conditions are met.

## 3.4 Virtqueue receive flow filters (RFF)
0. Design goal:
   To filter and/or to steer packet based on specific pattern match to a
   specific destination to support application/networking stack driven receive
   processing.
1. Two use cases are: to support Linux netdev set_rxnfc() for ETHTOOL_SRXCLSRLINS
   and to support netdev feature NETIF_F_NTUPLE aka ARFS.

### 3.4.1 control path
1. The number of flow filter operations/sec can range from 100k/sec to 1M/sec
   or even more. Hence flow filter operations must be done over a queueing
   interface using one or more queues.
2. The device should be able to expose one or more supported flow filter queue
   count and its start vq index to the driver.
3. As each device may be operating for different performance characteristic,
   start vq index and count may be different for each device. Secondly, it is
   inefficient for device to provide flow filters capabilities via a config space
   region. Hence, the device should be able to share these attributes using
   dma interface, instead of transport registers.
4. Since flow filters are enabled much later in the driver life cycle, driver
   will likely create these queues when flow filters are enabled.
5. Flow filter operations are often accelerated by device in a hardware. Ability
   to handle them on a queue other than control vq is desired. This achieves near
   zero modifications to existing implementations to add new operations on new
   purpose built queues (similar to transmit and receive queue). Some devices
   may not support flow filter queues and may want to support flow filter operations
   over existing cvq, this gives the ability to utilize an existing cvq.
   Therefore,
   a. Flow filter queues and flow filter commands on cvq are mutually exclusive.
   b. When flow filter queues are supported, the driver should use the flow filter
      queues flow filter operations.
      (Since cvq is not enabled for flow filters, any flow filter command coming
      on cvq must fail).
   c. If driver wants to use flow filters over cvq, driver must explicitly
      enable flow filters on cvq via a command, when it is enabled on the cvq
      driver cannot use flow filter queues. This eliminates any synchronization
      needed by the device among different types of queues.
6. The filter masks are optional; the device should be able to expose if it
   support filter masks.
7. The driver may want to have priority among group of flow entries; to facilitate
   the device support grouping flow filter entries by a notion of a flow group.
   Each flow group defines priority in processing flow.
8. The driver and group owner driver should be able to query supported device
   limits for the receive flow filters.
9. Query the flow filter capabilities of the member device by the owner device
   using administrative command.

### 3.4.2 flow operations path
1. The driver should be able to define a receive packet match criteria, an
   action and a destination for a packet. For example, an ipv4 packet with a
   multicast address to be steered to the receive vq 0. The second example is
   ipv4, tcp packet matching a specified IP address and tcp port tuple to
   be steered to receive vq 10.
2. The match criteria should include exact tuple fields well-defined such as mac
   address, IP addresses, tcp/udp ports, etc.
3. The match criteria should also optionally include the field mask.
4. Action includes (a) dropping or (b) forwarding the packet.
5. Destination is a receive virtqueue index.
6. Receive packet processing chain is:
   a. filters programmed using cvq commands VIRTIO_NET_CTRL_RX,
      VIRTIO_NET_CTRL_MAC and VIRTIO_NET_CTRL_VLAN.
   b. filters programmed using RFF functiionality.
   c. filters programmed using RSS VIRTIO_NET_CTRL_MQ_RSS_CONFIG command.
   Whichever filtering and steering functionality is enabled, they are applied
   in the above order.
7. If multiple entries are programmed which has overlapping filtering attributes
   for a received packet, the driver to define the location/priority of the entry.
8. The filter entries are usually short in size of few tens of bytes,
   for example IPv6 + TCP tuple would be 36 bytes, and ops/sec rate is
   high, hence supplying fields inside the queue descriptor is preferred for
   up to a certain fixed size, say 96 bytes.
9. A flow filter entry consists of (a) match criteria, (b) action,
    (c) destination and (d) a unique 32 bit flow id, all supplied by the
    driver.
10. The driver should be able to query and delete flow filter entry
    by the flow id.

### 3.4.3 interface example

1. Flow filter capabilities to query using a DMA interface such as cvq
using two different commands.

```
struct virtio_net_rff_cmd {
	u8 class; /* RFF class */
	u8 commands; /* 0 = query cap
		      * 1 = query packet fields mask
		      * 2 = enable flow filter operations over cvq
		      * 3 = add flow group
		      * 4 = del flow group
		      * 5 = flow filter op.
		      */
	u8 command-specific-data[];
};

/* command 1 (query) */
struct flow_filter_capabilities {
	le16 start_vq_index;
	le16 num_flow_filter_vqs;
	le16 max_flow_groups; /* valid group id = max_flow_groups - 1 */
	le16 max_group_priorities; /* max priorities of the group */
	le32 max_flow_filters_per_group;
	le32 max_flow_filters; /* max flow_id in add/del 
				* is equal = max_flow_filters - 1.
				*/
	u8 max_priorities_per_group;
	u8 cvq_supports_flow_filters_ops;
};

/* command 2 (query packet field masks) */
struct flow_filter_fields_support_mask {
	le64 supported_packet_field_mask_bmap[1];
};

```

2. Group add/delete cvq commands:

```
/* command 3 */
struct virtio_net_rff_group_add {
	le16 priority;	/* higher the value, higher priority */
	le16 group_id;
};


/* command 4 */
struct virtio_net_rff_group_delete {
	le16 group_id;

```

3. Flow filter entry add/modify, delete over flow vq:

```
struct virtio_net_rff_add_modify {
	u8 flow_op;
	u8 priority;	/* higher the value, higher priority */
	u16 group_id;
	le32 flow_id;
	struct match_criteria mc;
	struct destination dest;
	struct action action;

	struct match_criteria mask;	/* optional */
};

struct virtio_net_rff_delete {
	u8 flow_op;
	u8 padding[3];
	le32 flow_id;
};

```

### 3.4.4 For incremental future
a. Driver should be able to specify a specific packet byte offset, number
   of bytes and mask as math criteria.
b. Support RSS context, in addition to a specific RQ.
c. If/when virtio switch object is implemented, support ingress/egress flow
   filters at the switch port level.

## 3.5 Packet timestamp
1. Device should provide transmit timestamp and receive timestamp of the packets
   at per packet level when the timestamping is enabled in the device.
2. Device should provide the current frequency and the frequency unit for the
   software to synchronize the reference point of software and the device using
   a control vq command.

### 3.5.1 Transmit timestamp
1. Transmit completion must contain a packet transmission timestamp when the
   device is enabled for it.
2. The device should record the packet transmit timestamp in the completion at
   the farthest egress point towards the network.
3. The device must provide a transmit packet timestamp in a single DMA
   transaction along with the rest of the transmit completion fields.

### 3.5.2 Receive timestamp
1. Receive completion must contain a packet reception timestamp when the device
   is enabled for it.
2. The device should record the received packet timestamp at the closet ingress
   point of reception from the network.
3. The device should provide a receive packet timestamp in a single DMA
   transaction along with the rest of the receive completion fields.

## 3.6 Header data split for the receive virtqueue
1. The device should be able to DMA the packet header and data to two different
   memory locations, this enables driver and networking stack to perform zero
   copy to application buffer(s).
2. The driver should be able to configure maximum header buffer size per
   virtqueue.
3. The header buffer to be in a physically contiguous memory per virtqueue
4. The device should be able to indicate header data split in the receive
   completion.
5. The device should be able to zero pad the header buffer when the received
   header is shorter than cpu cache line size.
