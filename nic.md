- Rx Engine->Rx FIFO->Rules Checkers->Frame Buffers---------Nic Rx Producer Ring------DMA----Host Rx Producer Ring
- Rx Engine->Rx FIFO->Rules Checkers->Frame Buffers---------Full BD---List Initiator---DMA---Host Rx Return Ring
  -- PHY -> internal FIFO-> NIC ineternal memory -> classifies the frame and checks for the rules matchers, perform offloaded checksum calculations.
  
- Host Tx Producer Rings->NIC send Ring Ring(Tx data -> buffer0/1...)->Tx FIFO-> Tx MAC
  -- NIC inernal memory -> Tx FIFO -> PHY

  Note: FIFO is probably arrarys pointing to memory buffer.
