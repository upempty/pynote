To understand the NIC driver for transmit and receive packets,   
so to view different version for varies thought, and note that legacy code  may have bugs.


old====for basic understanding, Tx buffer may be overwriten when DMA is slow in extreme case:
https://github.com/willdurand/ArvernOS/blob/8b183a51311591158fd4f20c5a08a73c69dd1b03/src/kernel/arch/x86_64/drivers/rtl8139.c
vs
https://elixir.bootlin.com/linux/v6.0/source/drivers/net/ethernet/realtek/8139too.c
vs
https://elixir.bootlin.com/linux/2.0.37pre2/source/drivers/net/rtl8139.c#L204

PCI vs PCIe  
```
PCI parallel connection (sync so that slow).  
PCIe uses switch: even serial trasmit but avoid sync, and  can use multiple lanes for different flow.
```
![image](https://github.com/upempty/pynote/assets/52414719/b80c8559-fd91-460a-9142-5a056e71d3db)


- Tx and Rx descriptor and buffers for descriptor.
```
check via ethtool -g interfacename to get numbers of Tx and Rx descriptors.
buffer size defined (TBC further) in NIC driver. Probably the total buffer size for DMA is
- BufferSize defined x TxDesc numbers = Tx total buffers;
- BufferSize defined x RxDesc numbers = Rx total buffers;

```
- Interface from DMAed packet from NIC HW in received buffer
```
- ??? rtl8139.c and rtl8139.h
```
- Interface for transmit buffer filled to be ready for DMAing to NIC HW.
```
- ??? rtl8139.c and rtl8139.h
```
- kmain_start-> initcall_init() -> initcall_foreach(call_fn) -> initcall_late_register(network_init) -> network_init()

```
// recv callback via register/interrupt
  if (rtl8139_init(device)) {
    net_driver = rtl8139_driver();
  }


// transmit func
load_network_config(kernel_cfg, net_driver);


void ethernet_send_frame(net_interface_t* interface,
                         uint8_t dst_mac[6],
                         uint16_t ethertype,
                         uint8_t* data,
                         uint32_t len)
{
  ethernet_header_t header = { .ethertype = htons(ethertype) };
  memcpy(header.src_mac, interface->mac, 6);
  memcpy(header.dst_mac, dst_mac, 6);

  uint32_t frame_len = sizeof(ethernet_header_t) + len;

  if (frame_len < 64) {
    frame_len = 64;
  }

  uint8_t* frame = (uint8_t*)calloc(1, frame_len);
  memcpy(frame, &header, sizeof(ethernet_header_t));
  memcpy(frame + sizeof(ethernet_header_t), data, len);

  interface->driver->transmit(frame, frame_len);

  free(frame);
}
```
