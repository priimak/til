How to proxy all tcp traffic for a given port from one node to another
======================================================================

Consider a use case where a given node (let say `main-node`) has access to a certain other service runing
on port 80 another node (let sat that its dns name is `service-node`) and you want to access this service 
from your laptop which has access to `main-node` but not to `service-node`. We can use [goproxy](https://github.com/snail007/goproxy) tool to accomplish that. On the `main-node` install `goproxy` command line tool

```shell
curl -L https://raw.githubusercontent.com/snail007/goproxy/master/install_auto.sh | bash
```

This will install the tool that can be accessed as `proxy`. To establish proxy on `main-node` run

```shell
proxy tcp -p ":8081" -T tcp -P "service-node:80"
```

You can also use ip address instead of dns name `service-node`. Now you can access this service from your laptop like so

```shell
curl http://main-node:8081/
```

