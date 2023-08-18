# 1. Introduction

This document describes the overall requirements for virtio net device
improvements for upcoming release 1.4. Some of these requirements are
interrelated and influence the interface design, hence reviewing them
together is desired while updating the virtio net interface.

# 2. Summary
1. Device counters visible to the driver

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
