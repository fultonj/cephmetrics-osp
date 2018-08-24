Explanation of Version 1 Networking
===================================

How can an instance on the overcloud reach the Ceph cluster, which was set up on an isolated storage network, to read Ceph data?

Explanation
-----------

A more specific version of this question is, if you SSH to the instance created by [create.sh](https://github.com/fultonj/cephmetrics-osp/blob/915e994a72518a8398a3014ce13b70ed18d58ff8/instance/create.sh#L68), then why does the following command return the metrics data instead of timing out? 

```
[centos@centos ~]$ curl -s http://192.168.24.17:9283/metrics | tail -3
ceph_osd_op_w{ceph_daemon="osd.1"} 176107.0
ceph_osd_op_w{ceph_daemon="osd.2"} 239969.0
ceph_osd_op_w{ceph_daemon="osd.0"} 485703.0
[centos@centos ~]$ 
```

- The Ceph nodes were deployed with an isolated storage network (172.18.0.0/24) [1][2]
- The Ceph Mgr Prometheus exporter [3] listens on the controllers on port 9283 on _all_ IPs [4]
- Thus, we can query mgr the exporter at its provisioning network IP: 192.168.24.17:9283
- The controller node has routes for the provisioning network and the 10.0.0.0/24 network [5]
- The ceph-metrics interface is running on an instance on a private vxlan network [6]
- The same instance has a floating IP on the 10.0.0.0/24 network [7]

Thus, the ceph-metrics instance uses its route to public network (10.0.0.0/24) and then uses the controller's route [5] to the provisioning network and is then able to query the ceph mgr prometheus exporter running on the controller node [8].

Footnotes
---------

```
[1] 
[root@overcloud-controller-2 ~]# lsof -Pnl +M -i4 | grep ceph | grep LISTEN
ceph-mgr   36826      167   42u  IPv4 180553029      0t0  TCP 172.18.0.20:6800 (LISTEN)
ceph-mon   79524      167   21u  IPv4 201963579      0t0  TCP 172.18.0.20:6789 (LISTEN)
[root@overcloud-controller-2 ~]# 

[2]
[root@overcloud-controller-2 ~]# ss -lptn 'sport = :6800' 
State       Recv-Q Send-Q                                                            Local Address:Port                                                                           Peer Address:Port              
LISTEN      0      128                                                                 172.18.0.20:6800                                                                                      *:*                   users:(("ceph-mgr",pid=36826,fd=42))
[root@overcloud-controller-2 ~]# 

[3] http://docs.ceph.com/docs/mimic/mgr/prometheus/

[4]
[root@overcloud-controller-2 ~]# ss -lptn 'sport = :9283' 
State       Recv-Q Send-Q                                                            Local Address:Port                                                                           Peer Address:Port              
LISTEN      0      5                                                                            :::9283                                                                                     :::*                   users:(("ceph-mgr",pid=36826,fd=43))
[root@overcloud-controller-2 ~]# 

[5]
[root@overcloud-controller-2 ~]# ip r | egrep "10.0.0.0/24|192.168.24.0/24"
10.0.0.0/24 dev vlan10 proto kernel scope link src 10.0.0.15 
192.168.24.0/24 dev br-ex proto kernel scope link src 192.168.24.17 
[root@overcloud-controller-2 ~]# 

[6]
openstack subnet create --network $PRIV_NET_NAME $PRIV_SUBNET_NAME \
                        --dns-nameserver $DNS_IP \
                        --subnet-range 192.168.99.0/24
[7]
openstack subnet create --network $PUB_NET_NAME $PUB_SUBNET_NAME --subnet-range 10.0.0.0/24 \
                         --allocation-pool start=10.0.0.20,end=10.0.0.250 \
                         --dns-nameserver $DNS_IP --gateway 10.0.0.1 \
                         --no-dhcp
[8]
[root@centos ~]# tcptraceroute -n 192.168.24.17
traceroute to 192.168.24.17 (192.168.24.17), 30 hops max, 60 byte packets
 1  192.168.99.1  1.628 ms  6.897 ms  6.682 ms
 2  10.0.0.1  6.382 ms  6.086 ms  5.776 ms
 3  192.168.24.17 <rst,ack>  5.511 ms  5.352 ms  5.781 ms
[root@centos ~]#
```
