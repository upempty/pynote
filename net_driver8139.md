To understand the NIC driver for transmit and receive packets,   
so to view different version for varies thought, and note that legacy code  may have bugs.


old====for basic understanding, Tx buffer may be overwriten when DMA is slow in extreme case:
https://github.com/willdurand/ArvernOS/blob/8b183a51311591158fd4f20c5a08a73c69dd1b03/src/kernel/arch/x86_64/drivers/rtl8139.c
vs
https://elixir.bootlin.com/linux/v6.0/source/drivers/net/ethernet/realtek/8139too.c
vs
https://elixir.bootlin.com/linux/2.0.37pre2/source/drivers/net/rtl8139.c#L204

PCI vs PCIe
![image](https://github.com/upempty/pynote/assets/52414719/b80c8559-fd91-460a-9142-5a056e71d3db)
