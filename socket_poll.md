switch port state monitoring   
- norminal socket
```
  1 - tcp socket (127.0.0.1+port) server in switch daemon.
      via vendor's api(with driver) to get port state, then send event via socket to monitoring client.  
  2 - tcp socket client to reflect the state to end user.
      via open(socket), poll, (If revents & POLLIN, ....) to get, close api to get port state

```

- fuse socket (user space defination for function of poll, get etc)  
```
  customized fuse eventfd socket created in context of switch daemon, and directly refered by client end user.

```
