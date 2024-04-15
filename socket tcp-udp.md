-
- PCB(protocol control block)  
- sk buffer  
- socket connection to use by application (TCP or UDP)  
  - connect: if connect failed within timeout, re-connect with a defined timer periodly.  
  - listen, accept: spawn child to monitor the accepted socket. if listen failed, it shall be tranport failure.  

