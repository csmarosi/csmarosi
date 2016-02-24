---
layout: post
title: Tor (client) in Docker for transparent proxying
---

I wanted to create an environment where the only way a container
 can access network is through Tor.
This means I can run a program (e.g. browser)
 and be certain that it have no way revealing the connection.
That is, I want to run *some of the applications* through Tor, but not others.


## Related containers

All the possible related work is done by @jfrazelle.
One can [transparent proxy all the traffic](https://blog.jessfraz.com/post/routing-traffic-through-tor-docker-container/)
 or create a [local proxy](https://blog.jessfraz.com/post/tor-socks-proxy-and-privoxy-containers/)

The latter is almost what we want except that
 the application must speak SOCKS, and is not forced to use the proxy.
There is even a [video](https://www.youtube.com/watch?v=4u8egQ7rvMk)
 about this latter approach.


## Implementation

The idea is to start a new container with enough capability
 to change iptables rules.
One can simply copy the [transparent proxy scripts](https://trac.torproject.org/projects/tor/wiki/doc/TransparentProxy)
 form the Tor wiki and run it inside that container.
Any process in this container must go through the iptables rules, thus use Tor.

You can check [my implementation here](https://github.com/csmarosi/dockerFiles/tree/master/root_tor).
To start it:

    docker run -d --cap-add NET_ADMIN --name=tor root_tor

The container sets some iptables rules after start;
 at least the following is needed:

    iptables -t nat -A OUTPUT -m owner --uid-owner $_tor_uid -j RETURN
    iptables -t nat -A OUTPUT -p tcp --syn -j REDIRECT --to-ports 9040
    iptables -A OUTPUT -m owner --uid-owner $_tor_uid -j ACCEPT
    iptables -A OUTPUT -d 127.0.0.0/8 -j ACCEPT
    iptables -A OUTPUT -m state --state ESTABLISHED -j ACCEPT
    iptables -A OUTPUT -j REJECT

Check (from inside) that it really acts as a transparent proxy:

    curl -k https://check.torproject.org/api/ip

With recent enough docker, one can create a special network with this container;
 with older version (from 1.0), one can still
 put containers (e.g. Chrome) into the network namespace of this Tor container.

    docker run --rm -it \
        --net=container:tor \
        -e DISPLAY=:0 -v /tmp/.X11-unix/X0:/tmp/.X11-unix/X0:ro \
        -u=$(id -u) -e HOME=/tmp \
        --name=chrome \
        jess/chrome --no-sandbox

Happy browsing!
Beware: Chrome can SIGBUS because `/dev/shm` in Docker is small (only 64M).
