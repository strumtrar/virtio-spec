# 1. Introduction

This document describes the overall requirements for virtio net device
improvements for upcoming release 1.4. Some of these requirements are
interrelated and influence the interface design, hence reviewing them
together is desired while updating the virtio net interface.

# 2. Summary
1. Device counters visible to the driver
2. Low latency tx and rx virtqueues for PCI transport

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
