---
comments: true
tags:
  - docker
  - swarm
  - macvlan
  - networking
  - consul
  - autoscaling
  - orbiter
  - underlay
---
## TL;DR 
This will get you routable containers with IPs on your existing subnets, advertising to Consul. They will also be scalable and placed across a cluster of Swarm hosts. It's assumed that you are already running Consul, so if not, there are a ton of tutorials out there. It's also assumed you know how to install Docker and various Linux kernels.

Bonus: We add an autoscaling API called <a href="https://gianarb.it/blog/orbiter-the-swarm-autoscaler-moves">Orbiter</a>.

## I *just* want to run containers, like now, on my existing infrastructure and networks!
So you have an existing environment. You use Consul for service discovery. Life is good. Containers are now a thing and you want to work them in without having to worry about overlay networking or reverse proxies. You also don't want to add extra latency (as some naysayers could use it as fuel to kill your hopes and dreams). Lastly, you don't have a lot of time to invest in a complex orchestration tool, such as Kuberenetes. With Macvlan support added to Docker 17.06, we can rock 'n roll and get shit done.

I'd consider this as a *bridge the gap* solution that may not be long lived. You may end up with k8s in the end, using overlay networking, enjoying the modularity it brings. This solution brings the least risk, easiest implementation and highest performance when you need it. You need to prove that the most basic Docker experience wont degrade the performance of your apps. If you were to jump straight to overlays, reverse proxies, etc, you are taking on a lot of extra baggage without having even traveled anywhere.

Lastly, Kelsey Hightower thinks (or thought) running without all the fluff was cool, last year. 
<blockquote class="twitter-tweet" data-conversation="none" data-lang="en"><p lang="en" dir="ltr">ipvlan and pure L3 network automation will unwind the mess that is container networking. No more bridges. No more NAT. No more overlays.</p>&mdash; Kelsey Hightower (@kelseyhightower) <a href="https://twitter.com/kelseyhightower/status/716987913917964288">April 4, 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>


## Networking (Macvlan)
You may already have a bunch of VLANs and subnets. Perhaps you want containers to ride on your existing VLANs and subnets. Macvlan (and Ipvlan) allow you to slap a MAC and an IP (Macvlan) or just an IP (ipvlan) on a container. You also don't want to introduce any more complexity and overhead into packets getting to your containers (such as overlay networking and reverse proxies).

## Service Discovery (Consul)
In my case, I'm using Consul. It's ubiquitous. Sure, Kubernetes and Swarm both provide internal service discovery mechanisms, but what about integrating that with your existing apps? What if you have an app you want to eventually containerize, but it may run on bare metal or ec2, advertising to Consul. How does everything "outside" talk to services that only exist "inside"?

## Orchestration (Swarm)
Swarm can be setup in minutes. You mark a node (preferably more than one to maintain quorum) as a manager and it will provide you a tokened url you can paste onto future worker nodes. Kubernetes, being more modular and feature packed, suffers as it's installation is a major pain. It also comes bundled with a bunch of features that are completely unnecessary for a minimalist orchestration layer.

***
## Configure OS Networking

##### **Trunk existing VLANs to your Dockerhosts**
* You dont *have* to, but I would recommend dedicating at least one VLAN/Subnet for containers to ride on
* You also don't have to use trunking

##### **Create tagged interfaces for each VLAN that you've trunked**
If you just want to ride on a single ip'd interface on your host, that's cool too.
* As far as trunked VLANs that you've created tagged interfaces for, you can leave them un-ip'd.
   
```
[root@xxx network-scripts]# ip a|grep bond0.40
8: bond0.40@bond0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP qlen 1000
```
   
##### **Enable Promiscuous mode for each interface (this is a requirement for Docker to be able to use them for Macvlan)**

```ip link set dev bond0.40 promisc on```

##### **..and in the case of CentOS, persist the change**

```echo "PROMISC=yes >> /etc/sysconfig/network-scripts/ifcfg-bond0.40"```
   
##### **If you want your host to be able to talk to it's containers, make sure you add a dummy bridged interface described below**

* Create an interface on the same vlan/subnet as your host. Note that the route is to the network range that you specified for the containers to sit on.
```
#ip link add macnet252 link bond0.252 type macvlan mode bridge
#ifconfig macnet252 up
#ip route add 10.90.255.0/24 dev macnet252
```

***
## Install Docker 17.06 (docker-ce)
* I recommend running a recent kernel, ie: 4.12.x

***
## Configure Docker Networking

##### **Split your subnet into a few chunks, so you can assign those chunks to each Docker host**
* Why in hell do we have to chunk it out? Why can't we specify a global range and leave the IPAM up to Docker? As it sits, Macvlan and IPvlan have dependencies on the underlay network configuration, which swarm manager doesn't currently manage. The current solution is to configure on a per-host basis or you can take advantage of the remote ipam driver, which would allow you to manage via a custom ipam server or something like infoblox (who has written a driver). This issue is on Docker's radar and they are actively working on a more elegant solution (they wanted to get macvlan +  swarm support in our hands in the meantime, as quickly as possible). https://github.com/eyz is also working on a generic IPAM server that might get open sourced. 
* CIDRs are used to define ranges, not separate, routable subnets. To the traditional network engineer/admin, this might seem odd, but it's just the way they elected to define ranges.
* There is an <a href="https://gist.github.com/nerdalert/3d2b891d41e0fa8d688c">experimental DHCP driver</a> that will allow you to set a global Macvlan scope, but I havent had a chance to try it.
* If you have 5 Docker hosts, split out your ranges in whatever chunks you feel appropriate. If you think you'll only ever run 5 containers on each host, maybe a /29 is fine for you, for that subnet. 


##### **Create per-node subnet ranges / networks**
My network is 172.80.0.0/16. I'm going to give each host a /24. I have preexisting hosts already on part of the first /24 of that network, so im going to start at 1.0 and move on. I dont need a network or broadcast address because the ranges fall inside the larger /16 supernet. The network name (in this case `vlan40_net` is arbitrary).

```
manager1# docker network create --config-only --subnet 172.80.0.0/16 --gateway 172.80.0.1 -o parent=bond0.40 --ip-range 10.90.1.0/24 vlan40_net
```
```
worker1# docker network create --config-only --subnet 172.80.0.0/16 --gateway 172.80.0.1 -o parent=bond0.40 --ip-range 10.90.2.0/24 vlan40_net
```
```
worker2# docker network create --config-only --subnet 172.80.0.0/16 --gateway 172.80.0.1 -o parent=bond0.40 --ip-range 10.90.3.0/24 vlan40_net
```

##### **Create swarm network**
Now I'm going to create the swarm enabled network, on the manager. This network references the config-only per-node networks we just created. The network name (swarm-vlan40_net) is arbitary.

```
manager1# docker network create -d macvlan --scope swarm --config-from vlan40_net swarm-vlan40_net
```

##### **Bask in the glory of Macvlan + Swarm**
```
manager1# docker network ls|grep vlan40
0c1e0ab98806        vlan40_net            null                local
znxv8ab5t3n1        swarm-vlan40_net      macvlan             swarm
```

***
## You can now use Registrator with Macvlan

##### **TL;DR**
* Use ```gliderlabs/registrator:master``` it contains a PR that was merged to master after 4/16 (the last official release). 
* Do NOT use ```gliderlabs/registrator:latest```. 
* ```EXPOSE``` proper port in your ```Dockerfile```
See <a href="http://killcity.io/2017/09/14/Registrator-support-for-Macvlan-exists.html">for details.</a>

## ~~Bundle the Consul agent inside your container and advertise in the same fashion you're used to.~~
* ~~But what about <a href="https://github.com/gliderlabs/registrator">Registrator</a>? The project is stale and doesn't seem to work with Macvlan enabled Swarm. I spent hours trying to get it to work with no avail. Works fine with `--network="host"`, but not with Macvlan/Swarm. Let me know if you are able to get it to work.~~
* ~~Don't worry: <a href="https://docs.docker.com/engine/admin/multi-service_container/">It's ok to bundle the agent inside the container along with your app</a>, at least for now. You'll still be a hero.~~
* ~~Since your container will have a real ip, it will appear in Consul as a host and will be routable.~~
* ~~Consul needs to run last and stay running, it will get executed via an entrypoint script with `exec`. It needs to be run with `exec` so it gets the honor of running as `PID 1` (so it can receive SIGTERMs when it's time has come). This allows the agent to leave the cluster gracefully. See below.~~

***
## Running a container as a service
```
manager1# docker service create --network swarm-vlan40_net --name portainer portainer/portainer
manager1# nkbu2j5suypr        portainer             replicated          1/1                 portainer/portainer:latest
```

**And to see what IPs are used by your containers on a specific host:**

```
manager1# docker network inspect swarm-vlan40_net
[
    {
        "Name": "swarm-vlan40_net",
        "Id": "znxv8ab5t3n1vdb86jtlie823",
        "Created": "2017-08-11T07:50:12.488791524-04:00",
        "Scope": "swarm",
        "Driver": "macvlan",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.80.0.0/16",
                    "IPRange": "172.80.1.0/24",
                    "Gateway": "172.80.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": "vlan40_net"
        },
        "ConfigOnly": false,
        "Containers": {
            "6a74fe709c30e517ebb2931651d6356bb4fddac48f636263182058036ce73d75": {
                "Name": "portainer.1.a4dysjkpppuyyqi18war2b4u6",
                "EndpointID": "777b2d15e175e70aaf5a9325fa0b4faa96347e4ec635b2edff204d5d4233c506",
                "MacAddress": "02:42:0a:5a:23:40",
                "IPv4Address": "172.80.1.0/24",
                "IPv6Address": ""
            },
        "Options": {
            "parent": "bond0.40"
        },
        "Labels": {}
    }
]
```
***
## UI
If you would dig a UI, I recommend Portainer. https://github.com/portainer/portainer

Another killer way to visualize your cluster is with, Visualizer, naturally. https://hub.docker.com/r/dockersamples/visualizer/tags/

***
## Bonus time
Orbiter. https://github.com/gianarb/orbiter

Scaling based on an API call. Trigger a webhook from your favorite monitoring software...

This will let you make a call to the HTTP API with an "up" or "down" and it will automatically scale the app. You can also adjust how many containers scale at those times. It really works!

## Not working?
I'm always happy to help if anyone needs a hand. I can be found on dockercommunity.slack.com @killcity.



{% if page.comments %} 
<div id="disqus_thread"></div>
<script>

/**
*  RECOMMENDED CONFIGURATION VARIABLES: EDIT AND UNCOMMENT THE SECTION BELOW TO INSERT DYNAMIC VALUES FROM YOUR PLATFORM OR CMS.
*  LEARN WHY DEFINING THESE VARIABLES IS IMPORTANT: https://disqus.com/admin/universalcode/#configuration-variables*/
/*
var disqus_config = function () {
var disqus_identifier = "{{ page.url }}";
var disqus_url = '{{ site.url }}{{ page.url }}';
this.page.url = disqus_url;
this.page.identifier = disqus_identifier;
};
*/
(function() { // DON'T EDIT BELOW THIS LINE
var d = document, s = d.createElement('script');
s.src = 'https://killcity.disqus.com/embed.js';
s.setAttribute('data-timestamp', +new Date());
(d.head || d.body).appendChild(s);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
                            
{% endif %}
