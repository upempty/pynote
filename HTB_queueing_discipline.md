
HTB queuing discipline

![image](https://github.com/upempty/pynote/assets/52414719/8619aaab-7b30-4a9e-a154-a158f2cabc25)

![image](https://github.com/upempty/pynote/assets/52414719/d8f2a8a2-4fa2-4810-9a9d-b7bb8638472f)

![image](https://github.com/upempty/pynote/assets/52414719/7684fd26-d784-4fe0-be99-e594b445e109)

```
HTB queuing discipline

To apply HTB discipline we have to go through the following steps
•	Define different classes with their rate limiting attributes
•	Add rules to classify the traffic in to different classes
In this example we will try to implement the same traffic sharing requirement as mentioned in the introduction section.
Web traffic will get 50% of the bandwidth while mail gets 30% and 20% is shared by all other traffic.
The following rules define the various classes with the traffic limits
tc qdisc add dev eth0 root handle 1: htb default 30
tc class add dev eth0 parent 1: classid 1:1 htb rate 100kbps ceil 100kbps
tc class add dev eth0 parent 1:1 classid 1:10 htb rate 50kbps ceil 100kbps
tc class add dev eth0 parent 1:1 classid 1:20 htb rate 30kbps ceil 100kbps
tc class add dev eth0 parent 1:1 classid 1:30 htb rate 20mbps ceil 100kbps
 
Now we must classify the traffic into their classes based on some match condition.
The following rules classify the web and mail traffic in to class 10 and 20. All other traffic are pushed to class 30 by default
tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 match ip dport 80 0xffff flowid 1:10 
tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 match ip dport 25 0xffff flowid 1:20

```
