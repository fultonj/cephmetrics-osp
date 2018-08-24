Proposal for Version 2 Networking
=================================

Why bother?
-----------

[Version 1 Networking](network1.md) has the following problems.

- We should not be relying on the provisioning network; it wasn't meant for that
- We should not be sending the ceph metrics traffic over the public network

What then?
----------

- Compose a CephMetrics network the same way we compose a [StorageNFS](https://github.com/openstack/tripleo-heat-templates/blob/5fadfd093fd2796b43719c02707ad2f514e9ff90/network_data_ganesha.yaml#L108-L116) when we use NFS Ganesha

- Deploy using TripleO with the newly composed CephMetrics network such that it is on all of the nodes running Ceph services; e.g. a third network to go with Storage and StorageMgmt. We don't want to use the existing Storage network used to reach OSDs because tenants shouldn't directly connect to OSDs. Instead the services which they use, e.g. Nova, Glance, Cinder, should connect directly to OSDs. We only want to share with the less secure overcloud tenant what we need to share. 

- As the overcloud admin (overcloudrc) create a non-shared provider network on the same VLAN as the StorageNFS network. Similar to the way [the TripleO old Manila docs](https://docs.openstack.org/tripleo-docs/latest/install/advanced_deployment/deploy_manila.html#network-isolation) use the Storage network (in the precomposable network days)
  
- The overcloud admin would also grant access to the cephmetrics _project_ within the overcloud.

- The instance booted to host the cephmetrics UI would get a NIC attached to the CephMetrics network so that it could access the prometheus exporters. It could share its UI via a public network however or by another provider network but the Ceph scraping plumbing would be properly handled.
  
- It would be nice if CephMetrics listened on the Ceph nodes not on *:9283 and *:9100 but on a specific IP from the CephMetrics Network.

![CephMetrics Logical Network Diagram](https://www.dropbox.com/s/qprrraef9hvvhzy/CephMetricsNetworkDiagram.png?raw=1)

