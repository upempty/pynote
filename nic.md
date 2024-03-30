```
- Rx Engine->Rx FIFO->Rules Checkers->Frame Buffers---------Nic Rx Producer Ring------DMA----Host Rx Producer Ring
- Rx Engine->Rx FIFO->Rules Checkers->Frame Buffers---------Full BD---List Initiator---DMA---Host Rx Return Ring
  -- PHY -> internal FIFO-> NIC ineternal memory -> classifies the frame and checks for the rules matchers, perform offloaded checksum calculations.
  
- Host Tx Producer Rings->NIC send Ring Ring(Tx data -> buffer0/1...)->Tx FIFO-> Tx MAC
  -- NIC inernal memory -> Tx FIFO -> PHY


```

```
why to add frame header(56bits) and starting frame(8bits) + [dest addr+src addr....] + FCS(verfication for errors)
101010......1011: last two bits is 11, NIC indentifies this as frame starting point.
ading heades and tails, then to sendout, so on receiver side, NIC knows the frame starting point.
```
