**Network Setup**
```
docker network create --subnet=172.24.0.0/16 instar-net
```

**DNS Server Configuration**
```
mkdir -p /opt/bind9/configuration
nano /opt/bind9/configuration/named.conf.options
```

This will make sure that BIND is listening on all interfaces and will use the Google DNS Servers as forwarders:
```
options {
    directory "/var/cache/bind";

    recursion yes;
    listen-on { any; };

    forwarders {
            8.8.8.8;
            4.4.4.4;
    };
};
```

Next, I will Define a Zone called instar-net.io, that points to /etc/bind/zones/db.instar-net.io Zone File:
```
nano /opt/bind9/configuration/named.conf.local
```
```
zone "instar-net.io" {
    type master;
    file "/etc/bind/zones/db.instar-net.io";
};
```

The **Zone File** called db.instar-net.io lists all the services that need to be managed (e.g. Docker container on the Docker network) and assigns them a hostname and an IP address:
```
nano /opt/bind9/configuration/db.instar-net.io
```
```
$TTL    1d ; default expiration time (in seconds) of all RRs without their own TTL value
@       IN      SOA     ns1.instar-net.io. root.instar-net.io. (
                  3      ; Serial
                  1d     ; Refresh
                  1h     ; Retry
                  1w     ; Expire
                  1h )   ; Negative Cache TTL

; name servers - NS records
     IN      NS      ns1.instar-net.io.

; name servers - A records
ns1.instar-net.io.             IN      A      172.24.0.2

service1.instar-net.io.        IN      A      172.24.0.3
service2.instar-net.io.        IN      A      172.24.0.4
```

# Build the Docker Image

```
nano /opt/bind9/Dockerfile
```

I can now build and tag the BIND image

```
docker build -t ddns-master .
```

# Run the Docker Container

The container has now to be created inside the Docker network instar-net with the IP address assigned to it inside db.instar-net.io:

```
docker run -d --rm --name=ddns-master --net=instar-net --ip=172.24.0.2 ddns-master
```
I can now verify my server configuration
```
docker exec -ti ddns-master /bin/bash
named-checkconf
named-checkzone instar-net.io /etc/bind/zones/db.instar-net.io
zone instar-net.io/IN: loaded serial 3
OK
```

# Connecting Services
Now it is possible to run the two service container using the dns-server container as a DNS server (I am using NGINX container because I already have the image. Use whatever container you want

```
docker run -d --rm --name=service1 --net=instar-net --ip=172.24.0.3 --dns=172.24.0.2 nginx:1.21.6-alpine /bin/ash -c "while :; do sleep 10; done"
```

```
sudo docker run -d --rm --name=service2 --net=instar-net --ip=172.24.0.4 --dns=172.24.0.2 nginx:1.21.6-alpine /bin/ash -c "while :; do sleep 10; done"
```

All container now run on the same network

```
docker network inspect instar-net

[
    {
        "Name": "instar-net",
        "IPAM": {
            "Config": [
                {
                    "Subnet": "172.24.0.0/16"
                }
            ]
        },
        "Containers": {
            "04bd7e3b3a033fd643d36fff787cda485dc5f3d4468212568b8ff4498e776993": {
                "Name": "ddns-master",
                "IPv4Address": "172.24.0.2/16"
            },
            "14deb32e260d15ff8543571f2c5fd1d99eeb9ba97042a97c34d9b933525ca8aa": {
                "Name": "service2",
                "IPv4Address": "172.24.0.4/16"
            },
            "cb6840cfd76d360dfe4cefc96486a11cd4b73f405d114c2830fd792c4883dd8b": {
                "Name": "service1",
                "IPv4Address": "172.24.0.3/16"
            }
        },
    }
]
```

I can test the DNS Service by connecting to one of the client service and ping the other

```
docker exec -it service1 nslookup service2.instar-net.io                                                         
Server:         127.0.0.11
Address:        127.0.0.11:53

Name:   service2.instar-net.io
Address: 172.24.0.4
```

Also the forwarder is doing it's job allowing me to resolve domains outside of the defined zone

```
docker exec -it service1 nslookup google.com                                                                     
Server:         127.0.0.11
Address:        127.0.0.11:53

Non-authoritative answer:
Name:   google.com
Address: 142.250.185.238
```