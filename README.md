# Commentary: Installing Openstack-helm All-In-One

I recently tried out the Openstack Helm project using the All-In-One Method.
It certainly was no easy copy/paste exercise.  But using some Kubernetes
expertise, I was able to get things to work for the most part.  I installed
this 3 times now and each took about 1.5 hours at least.

To install Openstack helm using the all-in-one (AIO) method,
follow the
[official instructions](https://docs.openstack.org/openstack-helm/latest/install/developer/index.html).

The summary:

```
git clone https://opendev.org/openstack/openstack-helm-infra.git
git clone https://opendev.org/openstack/openstack-helm.git
cd openstack-helm
./tools/deployment/developer/common/010-deploy-k8s.sh
./tools/deployment/developer/common/020-setup-client.sh
./tools/deployment/developer/common/030-ingress.sh
./tools/deployment/developer/nfs/040-nfs-provisioner.sh
./tools/deployment/developer/nfs/050-mariadb.sh
./tools/deployment/developer/nfs/060-rabbitmq.sh
./tools/deployment/developer/nfs/070-memcached.sh
./tools/deployment/developer/nfs/080-keystone.sh
./tools/deployment/developer/nfs/090-heat.sh
./tools/deployment/developer/nfs/100-horizon.sh
./tools/deployment/developer/nfs/120-glance.sh
./tools/deployment/developer/nfs/140-openvswitch.sh
./tools/deployment/developer/nfs/150-libvirt.sh
./tools/deployment/developer/nfs/160-compute-kit.sh
./tools/deployment/developer/nfs/170-setup-gateway.sh
```

But here is some commentary on some of the more interesting steps
to help you avoid and mitigate certain issues that came up for me.

I also go beyond the basic instructions to try things out on my own
to help me learn and understand what is going on.  I include that in my
commentary below.

Get a baremetal server  with a lot of CPU and RAM; the doc above mentions minimum of
8G and 4CPUs.  I went with 96G and 56 cores.

Install these [common requirements](https://docs.openstack.org/openstack-helm/latest/install/common-requirements.html).

Login as ubuntu (or any non-root user that has sudo priviledges).

Clone the relevant repos and create an alias for kubectl for the namespace
called "openstack" to save you some typing:

```
mkdir new ; cd new
git clone https://opendev.org/openstack/openstack-helm-infra.git
git clone https://opendev.org/openstack/openstack-helm.git
alias kk="kubectl -n openstack"
```

To help debug problematic Pods during the installation, do things like this:

```
kk get po |grep -v Completed |grep -v Running
# the problematic pods will probably show and be in Init, CrashLoop, or Error state.

kk logs (aPod)
kk logs (aPod in Init state) init
```

Before beginning, choose either deploy with NFS or Ceph; I chose NFS.
Since I chose NFS, I skipped the section regarding installing with Ceph and
skipped to the part entitled "Exercise the Cloud".

The first part about installing k8s and helm took a while so be prepared
for the wait.

If you are repeating the process for whatever reason, when you get to the part
about installing mariadb, confirm the "keystone" database is not there before
trying to install keystone.  If you don't, you will get errors when installing
keystone. Actually, I got errors anyway so I include instructions on how to
mitigate this below.

For example, I saw this after installing mariadb:

```
$ kk exec -ti mariadb-server-0 -- /bin/bash
mysql@mariadb-server-0:/$ mysql -uroot -ppassword
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 3109
Server version: 10.2.18-MariaDB-1:10.2.18+maria~bionic mariadb.org binary distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema | <-- note the keystone database is not there (good)
+--------------------+
3 rows in set (0.00 sec)
```

Here's a [link](https://kubernetes.slack.com/archives/C3WERB7DE/p1554227717015900)
to the openstack-helm slack channel where I read about
someone else having the porblem with the keystone database needing to be dropped.
Use this slack search to find other conversations "'credential' already exists in:#openstack-helm".

In summary, mitigate the problem like this:

```
$ helm delete --purge keystone
$ alias kk=“kubectl -n openstack”
$ kk exec -ti mariadb-server-0 -- sh
$ bash  
mysql@mariadb-server-0:/$ mysql -uroot -ppassword
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 538
Server version: 10.2.18-MariaDB-1:10.2.18+maria~bionic mariadb.org binary distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| keystone           |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.00 sec)

MariaDB [(none)]> drop database keystone;
Query OK, 19 rows affected (1.53 sec)
```

Then rerun the keystone installation command:

```
./tools/deployment/developer/nfs/080-keystone.sh
```

Installing the compute kit took a while and timed out in the "wait-for-pods.sh"
command.  In my case, I just let it sit for a while (and held off running the
next installation command); eventually, all pods were running and all jobs
complete.  I then ran the rest of the script where it left off before aborting.
Here's what the log looks like for the neutron-db-sync pod when it finished:

```
$ kk logs -f neutron-db-sync-wxzkx
+ neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> kilo, kilo_initial
INFO  [alembic.runtime.migration] Running upgrade kilo -> 354db87e3225, nsxv_vdr_metadata.py
INFO  [alembic.runtime.migration] Running upgrade 354db87e3225 -> 599c6a226151, neutrodb_ipam
INFO  [alembic.runtime.migration] Running upgrade 599c6a226151 -> 52c5312f6baf, Initial operations in support of address scopes
INFO  [alembic.runtime.migration] Running upgrade 52c5312f6baf -> 313373c0ffee, Flavor framework
INFO  [alembic.runtime.migration] Running upgrade 313373c0ffee -> 8675309a5c4f, network_rbac
INFO  [alembic.runtime.migration] Running upgrade 8675309a5c4f -> 45f955889773, quota_usage
INFO  [alembic.runtime.migration] Running upgrade 45f955889773 -> 26c371498592, subnetpool hash
INFO  [alembic.runtime.migration] Running upgrade 26c371498592 -> 1c844d1677f7, add order to dnsnameservers
INFO  [alembic.runtime.migration] Running upgrade 1c844d1677f7 -> 1b4c6e320f79, address scope support in subnetpool
INFO  [alembic.runtime.migration] Running upgrade 1b4c6e320f79 -> 48153cb5f051, qos db changes
INFO  [alembic.runtime.migration] Running upgrade 48153cb5f051 -> 9859ac9c136, quota_reservations
INFO  [alembic.runtime.migration] Running upgrade 9859ac9c136 -> 34af2b5c5a59, Add dns_name to Port
INFO  [alembic.runtime.migration] Running upgrade 34af2b5c5a59 -> 59cb5b6cf4d, Add availability zone
INFO  [alembic.runtime.migration] Running upgrade 59cb5b6cf4d -> 13cfb89f881a, add is_default to subnetpool
INFO  [alembic.runtime.migration] Running upgrade 13cfb89f881a -> 32e5974ada25, Add standard attribute table
INFO  [alembic.runtime.migration] Running upgrade 32e5974ada25 -> ec7fcfbf72ee, Add network availability zone
INFO  [alembic.runtime.migration] Running upgrade ec7fcfbf72ee -> dce3ec7a25c9, Add router availability zone
INFO  [alembic.runtime.migration] Running upgrade dce3ec7a25c9 -> c3a73f615e4, Add ip_version to AddressScope
INFO  [alembic.runtime.migration] Running upgrade c3a73f615e4 -> 659bf3d90664, Add tables and attributes to support external DNS integration
INFO  [alembic.runtime.migration] Running upgrade 659bf3d90664 -> 1df244e556f5, add_unique_ha_router_agent_port_bindings
INFO  [alembic.runtime.migration] Running upgrade 1df244e556f5 -> 19f26505c74f, Auto Allocated Topology - aka Get-Me-A-Network
INFO  [alembic.runtime.migration] Running upgrade 19f26505c74f -> 15be73214821, add dynamic routing model data
INFO  [alembic.runtime.migration] Running upgrade 15be73214821 -> b4caf27aae4, add_bgp_dragent_model_data
INFO  [alembic.runtime.migration] Running upgrade b4caf27aae4 -> 15e43b934f81, rbac_qos_policy
INFO  [alembic.runtime.migration] Running upgrade 15e43b934f81 -> 31ed664953e6, Add resource_versions row to agent table
INFO  [alembic.runtime.migration] Running upgrade 31ed664953e6 -> 2f9e956e7532, tag support
INFO  [alembic.runtime.migration] Running upgrade 2f9e956e7532 -> 3894bccad37f, add_timestamp_to_base_resources
INFO  [alembic.runtime.migration] Running upgrade 3894bccad37f -> 0e66c5227a8a, Add desc to standard attr table
INFO  [alembic.runtime.migration] Running upgrade 0e66c5227a8a -> 45f8dd33480b, qos dscp db addition
INFO  [alembic.runtime.migration] Running upgrade 45f8dd33480b -> 5abc0278ca73, Add support for VLAN trunking
INFO  [alembic.runtime.migration] Running upgrade 5abc0278ca73 -> d3435b514502, Add device_id index to Port
INFO  [alembic.runtime.migration] Running upgrade d3435b514502 -> 30107ab6a3ee, provisioning_blocks.py
INFO  [alembic.runtime.migration] Running upgrade 30107ab6a3ee -> c415aab1c048, add revisions table
INFO  [alembic.runtime.migration] Running upgrade c415aab1c048 -> a963b38d82f4, add dns name to portdnses
INFO  [alembic.runtime.migration] Running upgrade kilo -> 30018084ec99, Initial no-op Liberty contract rule.
INFO  [alembic.runtime.migration] Running upgrade 30018084ec99 -> 4ffceebfada, network_rbac
INFO  [alembic.runtime.migration] Running upgrade 4ffceebfada -> 5498d17be016, Drop legacy OVS and LB plugin tables
INFO  [alembic.runtime.migration] Running upgrade 5498d17be016 -> 2a16083502f3, Metaplugin removal
INFO  [alembic.runtime.migration] Running upgrade 2a16083502f3 -> 2e5352a0ad4d, Add missing foreign keys
INFO  [alembic.runtime.migration] Running upgrade 2e5352a0ad4d -> 11926bcfe72d, add geneve ml2 type driver
INFO  [alembic.runtime.migration] Running upgrade 11926bcfe72d -> 4af11ca47297, Drop cisco monolithic tables
INFO  [alembic.runtime.migration] Running upgrade 4af11ca47297 -> 1b294093239c, Drop embrane plugin table
INFO  [alembic.runtime.migration] Running upgrade 1b294093239c -> 8a6d8bdae39, standardattributes migration
INFO  [alembic.runtime.migration] Running upgrade 8a6d8bdae39 -> 2b4c2465d44b, DVR sheduling refactoring
INFO  [alembic.runtime.migration] Running upgrade 2b4c2465d44b -> e3278ee65050, Drop NEC plugin tables
INFO  [alembic.runtime.migration] Running upgrade e3278ee65050 -> c6c112992c9, rbac_qos_policy
INFO  [alembic.runtime.migration] Running upgrade c6c112992c9 -> 5ffceebfada, network_rbac_external
INFO  [alembic.runtime.migration] Running upgrade 5ffceebfada -> 4ffceebfcdc, standard_desc
INFO  [alembic.runtime.migration] Running upgrade 4ffceebfcdc -> 7bbb25278f53, device_owner_ha_replicate_int
INFO  [alembic.runtime.migration] Running upgrade 7bbb25278f53 -> 89ab9a816d70, Rename ml2_network_segments table
INFO  [alembic.runtime.migration] Running upgrade a963b38d82f4 -> 3d0e74aa7d37, Add flavor_id to Router
INFO  [alembic.runtime.migration] Running upgrade 3d0e74aa7d37 -> 030a959ceafa, uniq_routerports0port_id
INFO  [alembic.runtime.migration] Running upgrade 030a959ceafa -> a5648cfeeadf, Add support for Subnet Service Types
INFO  [alembic.runtime.migration] Running upgrade a5648cfeeadf -> 0f5bef0f87d4, add_qos_minimum_bandwidth_rules
INFO  [alembic.runtime.migration] Running upgrade 0f5bef0f87d4 -> 67daae611b6e, add standardattr to qos policies
INFO  [alembic.runtime.migration] Running upgrade 67daae611b6e -> 6b461a21bcfc, uniq_floatingips0floating_network_id0fixed_port_id0fixed_ip_addr
INFO  [alembic.runtime.migration] Running upgrade 6b461a21bcfc -> 5cd92597d11d, Add ip_allocation to port
INFO  [alembic.runtime.migration] Running upgrade 5cd92597d11d -> 929c968efe70, add_pk_version_table
INFO  [alembic.runtime.migration] Running upgrade 929c968efe70 -> a9c43481023c, extend_pk_with_host_and_add_status_to_ml2_port_binding
INFO  [alembic.runtime.migration] Running upgrade 89ab9a816d70 -> c879c5e1ee90, Add segment_id to subnet
INFO  [alembic.runtime.migration] Running upgrade c879c5e1ee90 -> 8fd3918ef6f4, Add segment_host_mapping table.
INFO  [alembic.runtime.migration] Running upgrade 8fd3918ef6f4 -> 4bcd4df1f426, Rename ml2_dvr_port_bindings
INFO  [alembic.runtime.migration] Running upgrade 4bcd4df1f426 -> b67e765a3524, Remove mtu column from networks.
INFO  [alembic.runtime.migration] Running upgrade b67e765a3524 -> a84ccf28f06a, migrate dns name from port
INFO  [alembic.runtime.migration] Running upgrade a84ccf28f06a -> 7d9d8eeec6ad, rename tenant to project
/var/lib/openstack/local/lib/python2.7/site-packages/sqlalchemy/dialects/mysql/base.py:3016: SAWarning: Unknown schema content: u'  CONSTRAINT `CONSTRAINT_1` CHECK (`shared` in (0,1))'
  util.warn("Unknown schema content: %r" % line)
/var/lib/openstack/local/lib/python2.7/site-packages/sqlalchemy/dialects/mysql/base.py:3016: SAWarning: Unknown schema content: u'  CONSTRAINT `CONSTRAINT_1` CHECK (`admin_state_up` in (0,1)),'
...
/var/lib/openstack/local/lib/python2.7/site-packages/sqlalchemy/dialects/mysql/base.py:3016: SAWarning: Unknown schema content: u'  CONSTRAINT `CONSTRAINT_1` CHECK (`enable_dhcp` in (0,1))'
  util.warn("Unknown schema content: %r" % line)
/var/lib/openstack/local/lib/python2.7/site-packages/sqlalchemy/dialects/mysql/base.py:3016: SAWarning: Unknown schema content: u'  CONSTRAINT `CONSTRAINT_1` CHECK (`dirty` in (0,1))'
  util.warn("Unknown schema content: %r" % line)
INFO  [alembic.runtime.migration] Running upgrade 7d9d8eeec6ad -> a8b517cff8ab, Add routerport bindings for L3 HA
INFO  [alembic.runtime.migration] Running upgrade a8b517cff8ab -> 3b935b28e7a0, migrate to pluggable ipam
INFO  [alembic.runtime.migration] Running upgrade 3b935b28e7a0 -> b12a3ef66e62, add standardattr to qos policies
INFO  [alembic.runtime.migration] Running upgrade b12a3ef66e62 -> 97c25b0d2353, Add Name and Description to the networksegments table
INFO  [alembic.runtime.migration] Running upgrade 97c25b0d2353 -> 2e0d7a8a1586, Add binding index to RouterL3AgentBinding
INFO  [alembic.runtime.migration] Running upgrade 2e0d7a8a1586 -> 5c85685d616d, Remove availability ranges.
Running upgrade for neutron ...
OK
```

That neutron-db-sync job needs to finish for other pods to finish their "init" stage.
Then you can run the rest of the script in `./tools/deployment/developer/nfs/160-compute-kit.sh`:

nova-db-sync job also took a long time; here is its log:

```
$ kk logs -f nova-db-sync-q2tm8
++ nova-manage --version
+ NOVA_VERSION=15.1.6
++ nova-manage api_db version
+ '[' 0 -gt 0 ']'
+ nova-manage api_db sync
/var/lib/openstack/local/lib/python2.7/site-packages/sqlalchemy/dialects/mysql/base.py:3016: SAWarning: Unknown schema content: u'  CONSTRAINT `CONSTRAINT_1` CHECK (`config_drive` in (0,1))'
  util.warn("Unknown schema content: %r" % line)
/var/lib/openstack/local/lib/python2.7/site-packages/sqlalchemy/dialects/mysql/base.py:3016: SAWarning: Unknown schema content: u'  CONSTRAINT `CONSTRAINT_1` CHECK (`disabled` in (0,1)),'
  util.warn("Unknown schema content: %r" % line)
/var/lib/openstack/local/lib/python2.7/site-packages/sqlalchemy/dialects/mysql/base.py:3016: SAWarning: Unknown schema content: u'  CONSTRAINT `CONSTRAINT_2` CHECK (`is_public` in (0,1))'
  util.warn("Unknown schema content: %r" % line)
+ manage_cells
+ '[' 15 -gt 14 ']'
+ nova-manage cell_v2 map_cell0
+ grep -q ' cell1 '
+ nova-manage cell_v2 list_cells
+ nova-manage cell_v2 create_cell --name=cell1 --verbose
dfef618b-50d1-4617-beee-9e288c33f2f8
++ nova-manage cell_v2 list_cells
++ awk -F '|' '/ cell1 / { print $3 }'
++ tr -d ' '
+ CELL1_ID=dfef618b-50d1-4617-beee-9e288c33f2f8
+ set +x
+ nova-manage db sync
/var/lib/openstack/local/lib/python2.7/site-packages/pymysql/cursors.py:166: Warning: (1831, u'Duplicate index `block_device_mapping_instance_uuid_virtual_name_device_name_idx`. This is deprecated and will be disallowed in a future release')
  result = self._query(query)
/var/lib/openstack/local/lib/python2.7/site-packages/sqlalchemy/dialects/mysql/base.py:3016: SAWarning: Unknown schema content: u'  CONSTRAINT `CONSTRAINT_1` CHECK (`delete_on_termination` in (0,1)),'
  util.warn("Unknown schema content: %r" % line)
/var/lib/openstack/local/lib/python2.7/site-packages/sqlalchemy/dialects/mysql/base.py:3016: SAWarning: Unknown schema content: u'  CONSTRAINT `CONSTRAINT_2` CHECK (`no_device` in (0,1))'
  util.warn("Unknown schema content: %r" % line)

...

/var/lib/openstack/local/lib/python2.7/site-packages/sqlalchemy/dialects/mysql/base.py:3016: SAWarning: Unknown schema content: u'  CONSTRAINT `CONSTRAINT_1` CHECK (`injected` in (0,1)),'
  util.warn("Unknown schema content: %r" % line)
/var/lib/openstack/local/lib/python2.7/site-packages/sqlalchemy/dialects/mysql/base.py:3016: SAWarning: Unknown schema content: u'  CONSTRAINT `CONSTRAINT_2` CHECK (`multi_host` in (0,1))'
  util.warn("Unknown schema content: %r" % line)
/var/lib/openstack/local/lib/python2.7/site-packages/pymysql/cursors.py:166: Warning: (1831, u'Duplicate index `uniq_instances0uuid`. This is deprecated and will be disallowed in a future release')
  result = self._query(query)
+ nova-manage db online_data_migrations
Running batches of 50 until complete
+---------------------------------------------+--------------+-----------+
|                  Migration                  | Total Needed | Completed |
+---------------------------------------------+--------------+-----------+
|    aggregate_uuids_online_data_migration    |      0       |     0     |
| delete_build_requests_with_no_instance_uuid |      0       |     0     |
|    migrate_aggregate_reset_autoincrement    |      0       |     0     |
|              migrate_aggregates             |      0       |     0     |
|      migrate_flavor_reset_autoincrement     |      0       |     0     |
|               migrate_flavors               |      0       |     0     |
|      migrate_instance_groups_to_api_db      |      0       |     0     |
|          migrate_instance_keypairs          |      0       |     0     |
|      migrate_instances_add_request_spec     |      0       |     0     |
|          migrate_keypairs_to_api_db         |      0       |     0     |
+---------------------------------------------+--------------+-----------+
+ echo 'Finished DB migrations'
Finished DB migrations
```

Once those jobs finish, you can run the rest of the script:

```
./tools/deployment/common/wait-for-pods.sh openstack

#NOTE: Validate Deployment info
export OS_CLOUD=openstack_helm
openstack service list
sleep 30 #NOTE(portdirect): Wait for ingress controller to update rules and restart Nginx

openstack compute service list
openstack network agent list
```

Here is some output below.  You will see several services and that they are alive (with a
smiley next to them) on openstack network agent list:

```
$ ./tools/deployment/common/wait-for-pods.sh openstack

$ #NOTE: Validate Deployment info
$ export OS_CLOUD=openstack_helm
$ openstack service list
+----------------------------------+-----------+----------------+
| ID                               | Name      | Type           |
+----------------------------------+-----------+----------------+
| 008254a8b4154f058b158f83b5443032 | keystone  | identity       |
| 16502d240f574f6587c67012e07f4c5d | neutron   | network        |
| 57ca64be081d4b4486b7ffc06cb5d52b | nova      | compute        |
| 6bf86232490e4304bf7fb13c92811fc8 | heat-cfn  | cloudformation |
| adb6f12f34b2433a9a9ce8047eba55d2 | glance    | image          |
| cdafff6c26e142919bee92db59645f28 | heat      | orchestration  |
| f2cce6a1fdb1473b97ca8f24a892efee | placement | placement      |
+----------------------------------+-----------+----------------+
$ sleep 30 #NOTE(portdirect): Wait for ingress controller to update rules and restart Nginx
$ 
$ openstack compute service list
+----+------------------+-----------------------------------+----------+---------+-------+----------------------------+
| ID | Binary           | Host                              | Zone     | Status  | State | Updated At                 |
+----+------------------+-----------------------------------+----------+---------+-------+----------------------------+
|  2 | nova-consoleauth | nova-consoleauth-55b5755f5b-8j7bz | internal | enabled | up    | 2019-07-28T19:19:19.000000 |
|  3 | nova-conductor   | nova-conductor-5757498876-qgwh5   | internal | enabled | up    | 2019-07-28T19:19:10.000000 |
|  4 | nova-scheduler   | nova-scheduler-6ddfb7b5d4-7bp5p   | internal | enabled | up    | 2019-07-28T19:19:10.000000 |
|  5 | nova-compute     | hstack1                           | nova     | enabled | up    | 2019-07-28T19:19:11.000000 |
+----+------------------+-----------------------------------+----------+---------+-------+----------------------------+
$ openstack network agent list
+--------------------------------------+--------------------+---------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host    | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+---------+-------------------+-------+-------+---------------------------+
| 3373ed4d-19b4-47f4-b7b8-b220bf0b3370 | L3 agent           | hstack1 | nova              | :-)   | UP    | neutron-l3-agent          |
| 73c796eb-a3e7-44c4-b4ac-a2233ed0c644 | DHCP agent         | hstack1 | nova              | :-)   | UP    | neutron-dhcp-agent        |
| aec0bc19-84eb-43cc-8c6d-603893d1cabe | Open vSwitch agent | hstack1 | None              | :-)   | UP    | neutron-openvswitch-agent |
| cd1294bf-aa48-47d5-b04a-c21538bd2240 | Metadata agent     | hstack1 | None              | :-)   | UP    | neutron-metadata-agent    |
+--------------------------------------+--------------------+---------+-------------------+-------+-------+---------------------------+
$ 
```

and things look good so far:

```
$ kk get job|grep -v "1/1"
NAME                              COMPLETIONS   DURATION   AGE
$ kk get po |grep -v Running|grep -v Comple
NAME                                          READY   STATUS      RESTARTS   AGE
```

More output:

```
$ ./tools/deployment/developer/nfs/170-setup-gateway.sh
+ OSH_BR_EX_ADDR=172.24.4.1/24
+ OSH_EXT_SUBNET=172.24.4.0/24
+ sudo ip addr add 172.24.4.1/24 dev br-ex
+ sudo ip link set br-ex up
+ sudo iptables -P FORWARD ACCEPT
++ sudo ip -4 route list 0/0
++ awk '{ print $5; exit }'
+ DEFAULT_ROUTE_DEV=bond0
+ sudo iptables -t nat -A POSTROUTING -o bond0 -s 172.24.4.0/24 -j MASQUERADE
+ sudo docker run -d --name br-ex-dns-server --net host --cap-add=NET_ADMIN --volume /etc/kubernetes/kubelet-resolv.conf:/etc/kubernetes/kubelet-resolv.conf:ro --entrypoint dnsmasq docker.io/openstackhelm/neutron:ocata --keep-in-foreground --no-hosts --bind-interfaces --resolv-file=/etc/kubernetes/kubelet-resolv.conf --address=/svc.cluster.local/172.24.4.1 --listen-address=172.24.4.1
729b1d3319945fc43f9a6db43ba6fce378b02a2ddc9b6e6ef78d4f8e081556bc
+ sleep 1
+ sudo docker top br-ex-dns-server
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
nobody              432166              432139              62                  14:22               ?                   00:00:00            dnsmasq --keep-in-foreground --no-hosts --bind-interfaces --resolv-file=/etc/kubernetes/kubelet-resolv.conf --address=/svc.cluster.local/172.24.4.1 --listen-address=172.24.4.1

$ route -n|grep -v  cali
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.171.201.229  0.0.0.0         UG    0      0        0 bond0
10.0.0.0        10.171.200.1    255.0.0.0       UG    0      0        0 bond0
10.171.200.0    0.0.0.0         255.255.252.0   U     0      0        0 bond0
161.26.0.0      10.171.200.1    255.255.0.0     UG    0      0        0 bond0
166.8.0.0       10.171.200.1    255.252.0.0     UG    0      0        0 bond0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.24.4.0      0.0.0.0         255.255.255.0   U     0      0        0 br-ex
192.168.171.192 0.0.0.0         255.255.255.192 U     0      0        0 *

$ brctl show
bridge name	bridge id		STP enabled	interfaces
docker0		8000.02422ecf8d71	no		

$ ip -d link show br-ex 
120: br-ex: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1
    link/ether f6:05:d1:fe:ca:4d brd ff:ff:ff:ff:ff:ff promiscuity 1 
    openvswitch addrgenmode eui64 
```

When all is up, goto the part about exercising the cloud
Run the part about creating the public network on 172.24.4.0/16 and run just that manually:

```
ubuntu@hstack1:~/new/openstack-helm$ openstack stack create --wait \
>   --parameter network_name=${OSH_EXT_NET_NAME} \
>   --parameter physical_network_name=public \
>   --parameter subnet_name=${OSH_EXT_SUBNET_NAME} \
>   --parameter subnet_cidr=${OSH_EXT_SUBNET} \
>   --parameter subnet_gateway=${OSH_BR_EX_ADDR%/*} \
>   -t ./tools/gate/files/heat-public-net-deployment.yaml \
>   heat-public-net-deployment
2019-07-28 11:22:19Z [heat-public-net-deployment]: CREATE_IN_PROGRESS  Stack CREATE started
2019-07-28 11:22:20Z [heat-public-net-deployment.public_net]: CREATE_IN_PROGRESS  state changed
2019-07-28 11:22:21Z [heat-public-net-deployment.public_net]: CREATE_COMPLETE  state changed
2019-07-28 11:22:21Z [heat-public-net-deployment.private_subnet]: CREATE_IN_PROGRESS  state changed
2019-07-28 11:22:22Z [heat-public-net-deployment.private_subnet]: CREATE_COMPLETE  state changed
2019-07-28 11:22:22Z [heat-public-net-deployment]: CREATE_COMPLETE  Stack CREATE completed successfully
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| id                  | 1b37277f-2092-49a0-83d3-74f44080188e |
| stack_name          | heat-public-net-deployment           |
| description         | No description                       |
| creation_time       | 2019-07-28T11:22:18Z                 |
| updated_time        | None                                 |
| stack_status        | CREATE_COMPLETE                      |
| stack_status_reason | Stack CREATE completed successfully  |
+---------------------+--------------------------------------+
```

Goto horizon via the IP address of your baremetal server on port 3100
(e.g., http://x.x.x.x:31000) where x.x.x.x is the IP address of your baremetal server.

The 31000 is a Kubernetes nodePort that you can view via `kk get svc | grep horizon`.

Goto the horizon UI and click on top right to Download the v3 api info.
Source that info on a shell on your host using user=admin, password=password.

NOTE: you can also set the variables as mentioned in the openstack-helm docs:

```
export OS_USERNAME='admin'
export OS_PASSWORD='password'
export OS_PROJECT_NAME='admin'
export OS_PROJECT_DOMAIN_NAME='default'
export OS_USER_DOMAIN_NAME='default'
export OS_AUTH_URL='http://keystone.openstack.svc.cluster.local/v3'
```

or

```
$ source admin-openrc.sh 
Please enter your OpenStack Password for project admin as user admin: 
```

Then run the openstack cli:

```
$ openstack flavor list
+--------------------------------------+-----------+-------+------+-----------+-------+-----------+
| ID                                   | Name      |   RAM | Disk | Ephemeral | VCPUs | Is Public |
+--------------------------------------+-----------+-------+------+-----------+-------+-----------+
| 2e9e0677-4e0c-40f4-815e-160edceb97b9 | m1.xlarge | 16384 |  160 |         0 |     8 | True      |
| 31df585f-b38e-48b6-9950-9c54fbdd21fa | m1.large  |  8192 |   80 |         0 |     4 | True      |
| ab3a2b9e-bc61-4627-bc72-059468f0eda3 | m1.small  |  2048 |   20 |         0 |     1 | True      |
| bfaf015d-f6a0-4064-a449-07afa9e89177 | m1.tiny   |   512 |    1 |         0 |     1 | True      |
| dc749671-671c-404b-944b-a07247229593 | m1.medium |  4096 |   40 |         0 |     2 | True      |
+--------------------------------------+-----------+-------+------+-----------+-------+-----------+

$ openstack image list 
+--------------------------------------+---------------------+--------+
| ID                                   | Name                | Status |
+--------------------------------------+---------------------+--------+
| 263bb905-11d8-4534-8ba3-1741df254ddb | Cirros 0.3.5 64-bit | active |
+--------------------------------------+---------------------+--------+

ubuntu@hstack1:~/new/openstack-helm$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.171.201.229  0.0.0.0         UG    0      0        0 bond0
10.0.0.0        10.171.200.1    255.0.0.0       UG    0      0        0 bond0
10.171.200.0    0.0.0.0         255.255.252.0   U     0      0        0 bond0
161.26.0.0      10.171.200.1    255.255.0.0     UG    0      0        0 bond0
166.8.0.0       10.171.200.1    255.252.0.0     UG    0      0        0 bond0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.24.4.0      0.0.0.0         255.255.255.0   U     0      0        0 br-ex


ubuntu@hstack1:~$ openstack network list
+--------------------------------------+--------+--------------------------------------+
| ID                                   | Name   | Subnets                              |
+--------------------------------------+--------+--------------------------------------+
| 23b7c302-d4b2-4fdb-8bf9-6e1ca4527509 | public | 630ff8c4-260a-4868-bbe4-b6e602033adb |
+--------------------------------------+--------+--------------------------------------+
````

Add a keypair on the cli:

```
$ echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDQBFZzxFzB3JZbVv7XZf5nvYiH6jzA1xKJnd6MHXooo84egPiW/ItS2+PNX9g/c1HGYK/2hu9421Qr9Cm3vfQxPKp4CKm5kDRisFqdaMuSXoER9mpOn+r8AdIwXv6z4Ufr/obYQppSmjiixmZUTQTu7dCortadhEyGeOMjZflEHeGVZJQpE32HxpJ0MgK4/AyMEsuq7KK60YrSKYcReikyoSvujaPSv5TvDZNooGVOWxLoOrnEY+mNDGVlUa7cEMQMpv9NJGA4XinMsT1zFTfboeXpVY9fZeOZEOVL+RldaPtvDvlO9mZWekUgyp/0pMm6x4mOcXoQzkucpFGE167b dperiquet@Mozart.local" > dp-pub-key
$ nova keypair-add --pub-key my_pub_key dp-key1
```

Make a VM:

```
$ nova boot vm1 --flavor m1.tiny --image "Cirros 0.3.5 64-bit" --key-name dp-key --nic net-name=public
```

Wait for it to run:

```
$ nova list
+--------------------------------------+------+--------+------------+-------------+-------------------+
| ID                                   | Name | Status | Task State | Power State | Networks          |
+--------------------------------------+------+--------+------------+-------------+-------------------+
| a58323ce-eae4-4e12-bbff-efa1be2792c1 | vm1  | BUILD  | spawning   | NOSTATE     | public=172.24.4.8 |
+--------------------------------------+------+--------+------------+-------------+-------------------+

$ nova list
+--------------------------------------+------+--------+------------+-------------+-------------------+
| ID                                   | Name | Status | Task State | Power State | Networks          |
+--------------------------------------+------+--------+------------+-------------+-------------------+
| a58323ce-eae4-4e12-bbff-efa1be2792c1 | vm1  | ACTIVE | -          | Running     | public=172.24.4.8 |
+--------------------------------------+------+--------+------------+-------------+-------------------+
```

Look around at some of the OVS stuff:

```
$ kk exec -ti openvswitch-vswitchd-c5spl -- bash

root@hstack1:/# ovs-vsctl list-ports br-ex 
phy-br-ex

root@hstack1:/# ovs-vsctl list-br         
br-ex
br-int
br-tun

root@hstack1:/# ovs-vsctl list-ports br-ex
phy-br-ex

root@hstack1:/# ovs-vsctl list-ports br-int
int-br-ex
patch-tun
qvo18b4abca-97

root@hstack1:/# ovs-vsctl list-ports br-tun
patch-int

ubuntu@hstack1:~$ brctl show
bridge name	bridge id		STP enabled	interfaces
docker0		8000.0242a94f462f	no		
qbr18b4abca-97 8000.eeaaac0aaaef	no		qvb18b4abca-97
                                              tap18b4abca-97
```

I created a router using horizon and now I have:

```
$ ip netns
qrouter-fed0b46c-a523-4687-89b0-55005b08ed32

$ sudo ip netns exec qrouter-fed0b46c-a523-4687-89b0-55005b08ed32 ifconfig
lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

qg-645e3b26-f9 Link encap:Ethernet  HWaddr fa:16:3e:d2:b7:18  
          inet addr:172.24.4.16  Bcast:172.24.4.255  Mask:255.255.255.0
          inet6 addr: fe80::f816:3eff:fed2:b718/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:3 errors:0 dropped:0 overruns:0 frame:0
          TX packets:16 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:140 (140.0 B)  TX bytes:1172 (1.1 KB)
```

I started looking around at the heat template to create my own VM:

```
ubuntu@hstack1:~/new/openstack-helm$ vi tools/gate/files/heat-basic-vm-deployment.yaml 
ubuntu@hstack1:~/new/openstack-helm$ openstack stack create --wait \
>     --parameter public_net=${OSH_EXT_NET_NAME} \
>     --parameter image="${IMAGE_NAME}" \
>     --parameter ssh_key=${OSH_VM_KEY_STACK} \
>     --parameter cidr=${OSH_PRIVATE_SUBNET} \
>     --parameter dns_nameserver=${OSH_BR_EX_ADDR%/*} \
>     -t ./tools/gate/files/heat-basic-vm-deployment.yaml \
>     dp-vm1
2019-07-28 12:23:28Z [dp-vm1]: CREATE_IN_PROGRESS  Stack CREATE started
2019-07-28 12:23:30Z [dp-vm1.port_security_group]: CREATE_IN_PROGRESS  state changed
2019-07-28 12:23:31Z [dp-vm1.port_security_group]: CREATE_COMPLETE  state changed
2019-07-28 12:23:31Z [dp-vm1.router]: CREATE_IN_PROGRESS  state changed
2019-07-28 12:23:32Z [dp-vm1.private_net]: CREATE_IN_PROGRESS  state changed
2019-07-28 12:23:33Z [dp-vm1.private_net]: CREATE_COMPLETE  state changed
2019-07-28 12:23:33Z [dp-vm1.flavor]: CREATE_IN_PROGRESS  state changed
2019-07-28 12:23:34Z [dp-vm1.router]: CREATE_COMPLETE  state changed
2019-07-28 12:23:34Z [dp-vm1.flavor]: CREATE_COMPLETE  state changed
2019-07-28 12:23:34Z [dp-vm1.private_subnet]: CREATE_IN_PROGRESS  state changed
2019-07-28 12:23:35Z [dp-vm1.private_subnet]: CREATE_COMPLETE  state changed
2019-07-28 12:23:37Z [dp-vm1.server_port]: CREATE_IN_PROGRESS  state changed
2019-07-28 12:23:37Z [dp-vm1.router_interface]: CREATE_IN_PROGRESS  state changed
2019-07-28 12:23:38Z [dp-vm1.server_port]: CREATE_COMPLETE  state changed
2019-07-28 12:23:40Z [dp-vm1.server]: CREATE_IN_PROGRESS  state changed
2019-07-28 12:23:40Z [dp-vm1.router_interface]: CREATE_COMPLETE  state changed
2019-07-28 12:23:41Z [dp-vm1.server_floating_ip]: CREATE_IN_PROGRESS  state changed
2019-07-28 12:23:44Z [dp-vm1.server_floating_ip]: CREATE_COMPLETE  state changed
2019-07-28 12:24:04Z [dp-vm1.server]: CREATE_COMPLETE  state changed
2019-07-28 12:24:04Z [dp-vm1]: CREATE_COMPLETE  Stack CREATE completed successfully
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| id                  | e3681fcf-1585-4272-a3c1-ad7853e238c5 |
| stack_name          | dp-vm1                               |
| description         | No description                       |
| creation_time       | 2019-07-28T12:23:28Z                 |
| updated_time        | None                                 |
| stack_status        | CREATE_COMPLETE                      |
| stack_status_reason | Stack CREATE completed successfully  |
+---------------------+--------------------------------------+
```

## Using the Environment

If you run the [Exercise the Cloud](https://docs.openstack.org/openstack-helm/latest/install/developer/exercise-the-cloud.html)
section, it will create a VM on a private subnet (10.0.0.0/8) and attach it to a routeer that is
on a public network (172.24.4.0/24) and
attach a FIP on it.  After that, the script does an ssh to the VM and tries various things.

This is nice because it tests public network, private network, FIPs and basic connectivity using
a basic image.  This is good for developer purposes and just trying things out.  However, as
mentioned in this [article](https://ask.openstack.org/en/question/7145/havana-unable-to-ping-instances-from-host-machine/)
not everyone understands what is going on.  And this setup may not be the use-case a lot of people
want (i.e., some VMs on a subnet all on a BM).

## Using a Provider Network

For my needs, I wanted to have a provider network (as opposed to a public network) and I wanted
to attach VMs there and then just run them.  I don't need FIPs.

To do what I wanted, I followed the
[instructions to wipe](https://docs.openstack.org/openstack-helm/latest/install/developer/cleaning-deployment.html),
rebooted, re-applied the openstack-helm instructions (to get a fresh install) except that
I applied these changes (in the openstack-helm repo) before running the "compute-kit" installation script (thus
mapping br-ex as a provider network an dnot a public network):
```
ubuntu@hstack1:~/new/openstack-helm$ git diff 
diff --git a/tools/deployment/developer/nfs/160-compute-kit.sh b/tools/deployment/developer/nfs/160-compute-kit.sh
index 7e2a7fd..f339401 100755
--- a/tools/deployment/developer/nfs/160-compute-kit.sh
+++ b/tools/deployment/developer/nfs/160-compute-kit.sh
@@ -54,15 +54,15 @@ conf:
   plugins:
     ml2_conf:
       ml2_type_flat:
-        flat_networks: public
+        flat_networks: provider
     openvswitch_agent:
       agent:
         tunnel_types: vxlan
       ovs:
-        bridge_mappings: public:br-ex
+        bridge_mappings: provider:br-ex
     linuxbridge_agent:
       linux_bridge:
-        bridge_mappings: public:br-ex
+        bridge_mappings: provider:br-ex
```

I skipped the "Exercise the Cloud" script.

Follow this [doc](https://docs.openstack.org/mitaka/install-guide-ubuntu/launch-instance-networks-provider.html)
starting at "Create the provider network" and do this:

```
$ openstack network create  --share --external \
   --provider-physical-network provider \
   --provider-network-type flat provider

$ openstack subnet create --network provider \
  --allocation-pool start=172.24.4.10,end=172.24.4.40 \
  --dns-nameserver 8.8.4.4 --gateway 172.24.4.1 \
  --subnet-range 172.24.4.0/24 provider
```

We are using a similar setup as the original example except I'm using a provider network
instead.

After that, I created an instance like this:

```
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDQBFZzxFzB3JZbVv7XZf5nvYiH6jzA1xKJnd6MHXooo84egPiW/ItS2+PNX9g/c1HGYK/2hu9421Qr9Cm3vfQxPKp4CKm5kDRisFqdaMuSXoER9mpOn+r8AdIwXv6z4Ufr/obYQppSmjiixmZUTQTu7dCortadhEyGeOMjZflEHeGVZJQpE32HxpJ0MgK4/AyMEsuq7KK60YrSKYcReikyoSvujaPSv5TvDZNooGVOWxLoOrnEY+mNDGVlUa7cEMQMpv9NJGA4XinMsT1zFTfboeXpVY9fZeOZEOVL+RldaPtvDvlO9mZWekUgyp/0pMm6x4mOcXoQzkucpFGE167b dperiquet@Mozart.local" > dp-pub-key

nova keypair-add --pub-key my_pub_key dp-key1

nova boot vm1 --flavor m1.tiny --image "Cirros 0.3.5 64-bit" --key-name dp-key1 --nic net-name=provider
```

I then went to the security groups of this VM and added

* ingress ipv4, icmp all
* egress ipv4, icmp all
* ingress ipv4, tcp all
* egress ipv4, tcp all

Here's is some output:

```
$ nova list
+--------------------------------------+------+--------+------------+-------------+----------------------+
| ID                                   | Name | Status | Task State | Power State | Networks             |
+--------------------------------------+------+--------+------------+-------------+----------------------+
| eb7ccfed-a94a-4e3a-a552-c600c64104d2 | vm1  | ACTIVE | -          | Running     | provider=172.24.4.17 |
+--------------------------------------+------+--------+------------+-------------+----------------------+
ubuntu@hstack1:~$ ping 172.24.4.17
PING 172.24.4.17 (172.24.4.17) 56(84) bytes of data.
64 bytes from 172.24.4.17: icmp_seq=1 ttl=64 time=1.57 ms
64 bytes from 172.24.4.17: icmp_seq=2 ttl=64 time=0.280 ms
^C
--- 172.24.4.17 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.280/0.926/1.572/0.646 ms
ubuntu@hstack1:~$ ssh -i x.rsa cirros@172.24.4.17
$ ifconfig 
eth0      Link encap:Ethernet  HWaddr FA:16:3E:FA:EE:63  
          inet addr:172.24.4.17  Bcast:172.24.4.255  Mask:255.255.255.0
          inet6 addr: fe80::f816:3eff:fefa:ee63/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:347 errors:0 dropped:0 overruns:0 frame:0
          TX packets:305 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:39311 (38.3 KiB)  TX bytes:35442 (34.6 KiB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

NOTE: in the original example with a public network, I was unable to ping instances
I created by hand (i.e., without using the given heat template) from the host.  The
instances I created by hand were attached to the public network 172.24.4.0/24 and
I tried to ping them and they did not respond (I do not know why).

The article I mention above asks about this symptom and the conclusion seems to be that
this is expected behavior and led me to read an
[article on flat networking](https://wiki.openstack.org/wiki/UnderstandingFlatNetworking)
to help me debug it and an article on
[provider networks](https://docs.openstack.org/mitaka/install-guide-ubuntu/launch-instance-networks-provider.html)
to help me understand what is going on.  Honestly, I wonder if my problem was that I
needed to modify the security group as mentioned above (to allow icmp and ssh).  This
is a mystery I intend to solve another day.

### Analysis of Provider vs. Public

Here is what I think is happening (feedback for inaccuracies is welcome).
More bluntly, this is all one big GUESS.

When you run the comand to "Deploy Compute Kit (Nova and Neutron)", this
yaml is used:

```
tee /tmp/neutron.yaml << EOF
network:
  interface:
    tunnel: docker0
conf:
  neutron:
    DEFAULT:
      l3_ha: False
      max_l3_agents_per_router: 1
      l3_ha_network_type: vxlan
      dhcp_agents_per_network: 1
  plugins:
    ml2_conf:
      ml2_type_flat:
        flat_networks: public          <-- public network specified
    openvswitch_agent:
      agent:
        tunnel_types: vxlan
      ovs:
        bridge_mappings: public:br-ex  <-- map public to br-ex
    linuxbridge_agent:
      linux_bridge:
        bridge_mappings: public:br-ex  <-- map public to br-ex
EOF
```

The word "public" means using a public network which (I'm guessing) implies:

* that this network can get FIPs applied to it
* you cannot reach VMs on that network (I still need to prove or disprove this but
  the article above suggests this is normal)

In the other two lines that mention "public", they map
the physical interface "br-ex" to the public network.  I believe in a full
Openstack deployment, you would have that interface be a physical interface (e.g., eth1)
where you would reserve a block of IPs for instances.  Further, I believe that for every
compute node in an Openstack cluster, you would be running with a similar neutron yaml
that mentions the specific interface on that compute node where you want to attach instances.

When I changed "public" to "provider", this made it so that I can attach intances onto
that network and ping them.  I'm guessing that it also implies that I cannot attach FIPs
to them.

When you run the command to setup neutron, it creates the "br-ex" interface -- i.e.,
do an "ip link" before and after running the neutron setup commands and you will see
that "br-ex" is created after.  Later in the openstack-helm instructions, you assign
an IP address to "br-ex".  If "br-ex" is not present, then neutron commands did not
finish.  These are the commands that add the IP address to "br-ex":

```
# Assign IP address to br-ex
OSH_BR_EX_ADDR="172.24.4.1/24"
OSH_EXT_SUBNET="172.24.4.0/24"
sudo ip addr add ${OSH_BR_EX_ADDR} dev br-ex
sudo ip link set br-ex up
```


## Wiping Your Environment

Sometimes you just want to start over from a clean slate (i.e., clean baremetal server)
without doing an full OS reload.  To do that, follow the
[wipe](https://docs.openstack.org/openstack-helm/latest/install/developer/cleaning-deployment.html)
instructions.

In summary, just run the last chunk of commands and reboot.  I paste the instructions
here to be very specific about what I'm talking about:

```
for NS in openstack ceph nfs libvirt; do
   helm ls --namespace $NS --short | xargs -r -L1 -P2 helm delete --purge
done

sudo systemctl stop kubelet
sudo systemctl disable kubelet

sudo docker ps -aq | xargs -r -L1 -P16 sudo docker rm -f

sudo rm -rf /var/lib/openstack-helm/*

# NOTE(portdirect): These directories are used by nova and libvirt
sudo rm -rf /var/lib/nova/*
sudo rm -rf /var/lib/libvirt/*
sudo rm -rf /etc/libvirt/qemu/*

# NOTE(portdirect): Clean up mounts left behind by kubernetes pods
sudo findmnt --raw | awk '/^\/var\/lib\/kubelet\/pods/ { print $1 }' | xargs -r -L1 -P16 sudo umount -f -l
```

## Comparison with Packstack RDO

I've installed packstack all-in-one several times and openstack-helm reminds me of it
except the installation is easier to follow since it's Kubernetes (something I'm very
familiar/comfortable with troubleshooting).  For the most part, for the all-in-one style
deployments, I want to be able to create several VMs and have them talk to the Internet,
to each other and other machines outside of the all-in-one BM device.  I believe both
Packstack RDO and openstack-helm (with some modifications as mentioned above) accomplish
this.

## Limitations

I have to use openstack cli on the host where I have Kubernetes and openstack-helm because
the service is not exposed outside of the openstack-helm all-in-one host. I wonder if we
can make a nodePort for the keystone server and if we need a nodePort for the keystone-api
as well.

```
$ kk get svc|grep keysto
keystone                      ClusterIP   10.98.196.221    <none>        80/TCP,443/TCP                 7h10m
keystone-api                  ClusterIP   10.108.131.206   <none>        5000/TCP                       7h10m

$ ping keystone.openstack.svc.cluster.local
PING keystone.openstack.svc.cluster.local (10.98.196.221) 56(84) bytes of data.
```

The console display is not working and you get `novncproxy.openstack.svc.cluster.local’s
server IP address could not be found.` when you try to goto the console.

I will try to look into how to solve this since
I like being able to use the console for debugging.  But since the instances are working,
this is not a high priority for me.  But getting to know how to get the DNS to work for
that name might be useful in itself.

## Next Steps

* Setup networking so that machines outside of the all-in-one BM can reach my VMs.
  * Maybe get a block of IP addresses on the same subnet as my management interface (bond0)
    on my BM server and setup openstack on a provider network on that interface.
    * This would involve creating a new bridge calles say "br0", giving it my original
      IP address, and then adding bond0 to the bridge.  Then I can use bond0 as my provider
      interface.
    * Something like [this example from RDO Packstack](https://www.rdoproject.org/networking/neutron-with-existing-external-network/)
  * For servers that reside on the same subnet as my host, just add a route
    like `route add -net 172.24.4.0/24 gw x.x.x.x` where x.x.x.x is the IP of my host
    * This works well because the remote can now reach 172.24.4.0/24 and my VMs
      don't need a route back to that server (since it is on the same subnet).
* Try out the multi-node instructions.
* Get more understanding about what openstack-helm actually is and does.
* Eventually run openstack-helm on a 5 node k8s cluster.
  * not sure if I need to create a provider network on each host
  * do we allocate certain hosts as compute and certain ones as openstack control plane?


## Adding a new Glance Image

I added some images like this:

```
$ glance image-create --container-format=bare --disk-format=qcow2  --name=ubuntu16-mk < /tmp/ubuntu-xenial-minikube-0000000045-20G.qcow2 
...

$ glance image-create --container-format=bare --disk-format=qcow2  --name=ubuntu18 < /tmp/ubuntu-bionic-0000000273.qcow2
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | d1143021de09fced5c1840cdcba1c726     |
| container_format | bare                                 |
| created_at       | 2019-07-29T00:54:57Z                 |
| disk_format      | qcow2                                |
| id               | ee1231a7-e767-4523-bbec-8ba5b7a97ab6 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | ubuntu18                             |
| owner            | 436e38d55f4f41fba1b68a08100ad401     |
| protected        | False                                |
| size             | 702087536                            |
| status           | active                               |
| tags             | []                                   |
| updated_at       | 2019-07-29T00:55:06Z                 |
| virtual_size     | Not available                        |
| visibility       | shared                               |
+------------------+--------------------------------------+
```

Then I created an instance:

```
$ nova boot bionic1 --flavor m1.large --image "ubuntu18" --key-name dp-key1 --nic net-name=provider

$ nova list
+--------------------------------------+---------+--------+------------+-------------+----------------------+
| ID                                   | Name    | Status | Task State | Power State | Networks             |
+--------------------------------------+---------+--------+------------+-------------+----------------------+
| afbf13cd-53f9-4d66-be07-518407792274 | bionic1 | ACTIVE | -          | Running     | provider=172.24.4.12 |
| eb7ccfed-a94a-4e3a-a552-c600c64104d2 | vm1     | ACTIVE | -          | Running     | provider=172.24.4.17 |
+--------------------------------------+---------+--------+------------+-------------+----------------------+

$ ssh -i xx.rsa bonnyci@172.24.4.12
Warning: Permanently added '172.24.4.12' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 18.04.2 LTS (GNU/Linux 4.15.0-54-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

bonnyci@ubuntu:~$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.24.4.1      0.0.0.0         UG    0      0        0 ens3
169.254.169.254 172.24.4.10     255.255.255.255 UGH   0      0        0 ens3
172.24.4.0      0.0.0.0         255.255.255.0   U     0      0        0 ens3

bonnyci@ubuntu:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            3.9G     0  3.9G   0% /dev
tmpfs           798M  524K  798M   1% /run
/dev/vda1        76G  1.6G   71G   3% /
tmpfs           3.9G     0  3.9G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
tmpfs           798M     0  798M   0% /run/user/1000
```


## Editing Flavors in Horizon

Navigate to System->Flavors and just click Edit.

I setup my flavors like this (my VMs don't need a disk bigger than 20G):

```
$ openstack flavor list
+--------------------------------------+-----------+-------+------+-----------+-------+-----------+
| ID                                   | Name      |   RAM | Disk | Ephemeral | VCPUs | Is Public |
+--------------------------------------+-----------+-------+------+-----------+-------+-----------+
| 06167b2b-2a47-48c5-8272-a79ffeb9880e | m1.medium |  4096 |   20 |         0 |     2 | True      |
| 1742692c-365e-4c04-a0d4-884d08d075c2 | m1.small  |  2048 |   20 |         0 |     1 | True      |
| 1984748c-a3c1-4548-b9f4-460e12bf8059 | m1.xlarge | 16384 |   20 |         0 |     8 | True      |
| a3839c72-63a8-46cb-b2f4-15570a221a86 | m1.tiny   |   512 |    1 |         0 |     1 | True      |
| bac82ea2-b43c-4ebe-bfcb-3b1e04fea406 | m1.large  |  8192 |   20 |         0 |     4 | True      |
+--------------------------------------+-----------+-------+------+-----------+-------+-----------+
```

## Looking at the Kubernetes Deployment

The openstack-helm project installs Kubernetes via kubeadm.  I look around to see
what's there.

Here's the Kubernetes and Docker versions (v1.13.4, docker 18.9.2):

```
$ kubectl get node -o wide
NAME      STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
hstack1   Ready    master   10h   v1.13.4   172.17.0.1    <none>        Ubuntu 16.04.6 LTS   4.4.0-157-generic   docker://18.9.2

$ kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-0               Healthy   {"health": "true"} 
```

I see only one instance of etcd (makes sense for a single node k8s):

```
$ kubectl get po -n kube-system
NAME                                       READY   STATUS      RESTARTS   AGE
calico-etcd-q9lfn                          1/1     Running     0          10h
calico-kube-controllers-6b896f964f-c8clp   1/1     Running     0          10h
calico-node-lfvwq                          1/1     Running     1          10h
calico-settings-qjdw4                      0/1     Completed   0          10h
etcd-hstack1                               1/1     Running     0          10h <-- one etcd
ingress-error-pages-578666b5cf-xptcl       1/1     Running     0          10h
ingress-gsbsb                              1/1     Running     0          10h
kube-apiserver-hstack1                     1/1     Running     0          10h
kube-controller-manager-hstack1            1/1     Running     0          10h
kube-dns-5b4bdb8cd5-8gjks                  3/3     Running     0          10h
kube-proxy-vzzcf                           1/1     Running     0          10h
kube-scheduler-hstack1                     1/1     Running     0          10h
osh-dns-redirector-hstack1                 1/1     Running     0          10h
tiller-deploy-5c446c85f9-g78c7             1/1     Running     0          10h
```

The etcd state (i.e., the Kubernetes state) is save on `/var/lib/etcd` on the host.
This tells me that the Kubernetes state should be preserved across reboots.

```
$ kubectl describe po -n kube-system etcd-hstack1
  ...
    Mounts:
      /etc/kubernetes/pki/etcd from etcd-certs (rw)
      /var/lib/etcd from etcd-data (rw)
```

I also see they see they use calico v3.4.0 for networking:

```
$ kubectl describe po -n kube-system calico-node-lfvwq|grep Image:
    Image:         quay.io/stackanetes/kubernetes-entrypoint:v0.3.1
    Image:         calico/ctl:v3.4.0
    Image:         quay.io/calico/cni:v3.4.0
    Image:          quay.io/calico/node:v3.4.0
```

I'm assuming k8s nodes are designated for various functions via labels:

```
$ kubectl describe node     
Name:               hstack1
Roles:              master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    ceph-mds=enabled
                    ceph-mgr=enabled
                    ceph-mon=enabled
                    ceph-osd=enabled
                    ceph-rgw=enabled
                    kubernetes.io/hostname=hstack1
                    linuxbridge=enabled
                    node-role.kubernetes.io/master=
                    openstack-compute-node=enabled   <-- run as compute node
                    openstack-control-plane=enabled  <-- run as control plane
                    openstack-helm-node-class=primary
                    openvswitch=enabled
...
Addresses:
  InternalIP:  172.17.0.1            <-- docker IP (usually, this is the host IP)
  Hostname:    hstack1
...
Capacity:
 cpu:                56              <-- 56 cores
 ephemeral-storage:  960072088Ki     <-- 96G RAM
 hugepages-1Gi:      0
 hugepages-2Mi:      0
 memory:             97567716Ki
 pods:               220
...
PodCIDR:                     192.168.0.0/24
```


Certain pods running using the docker IP:

```
$ kk get po -o wide |grep -v 192.168
NAME                                          READY   STATUS      RESTARTS   AGE     IP                NODE
libvirt-libvirt-default-9zq4v                 1/1     Running     0          7h32m   172.17.0.1        hstack1
neutron-dhcp-agent-default-vb7mm              1/1     Running     0          7h22m   172.17.0.1        hstack1
neutron-l3-agent-default-9sn8x                1/1     Running     0          7h22m   172.17.0.1        hstack1
neutron-metadata-agent-default-4vx5w          1/1     Running     0          7h22m   172.17.0.1        hstack1
neutron-ovs-agent-default-rm8cv               1/1     Running     0          7h22m   172.17.0.1        hstack1
nova-compute-default-flfxx                    1/1     Running     0          7h23m   172.17.0.1        hstack1
nova-novncproxy-fd98699d5-7xxrx               1/1     Running     0          7h23m   172.17.0.1        hstack1
openvswitch-db-r4j2q                          1/1     Running     0          7h35m   172.17.0.1        hstack1
openvswitch-vswitchd-hnm8m                    1/1     Running     0          7h35m   172.17.0.1        hstack1
```

## A Reboot Test

I power-cycled the host running the openstack-helm deployment.  The machine came up ok
but the openstack-helm pods did not look good:

```
$ kk get po |grep -v Running|grep -v Comple
NAME                                          READY   STATUS      RESTARTS   AGE
glance-api-7c988cf5f9-s9b9k                   0/1     Error       0          3d7h
glance-registry-565594b768-wnc57              0/1     Init:0/1    1          3d7h
heat-api-6d74d44b74-6kspd                     0/1     Error       0          3d7h
horizon-6c54fc6c68-sc827                      0/1     Error       0          3d7h
libvirt-libvirt-default-9zq4v                 0/1     Init:0/1    1          3d6h
neutron-dhcp-agent-default-vb7mm              0/1     Init:0/1    1          3d6h
neutron-l3-agent-default-9sn8x                0/1     Init:0/1    1          3d6h
neutron-metadata-agent-default-4vx5w          0/1     Init:0/2    1          3d6h
neutron-ovs-agent-default-rm8cv               0/1     Error       0          3d6h
neutron-server-74d5b8f467-hgc7k               0/1     Error       0          3d6h
nova-api-metadata-66b8d4b5dd-xdm6n            0/1     Error       1          3d6h
nova-api-osapi-84b9f7b646-rxbk9               0/1     Error       0          3d6h
nova-compute-default-flfxx                    0/1     Init:0/3    1          3d6h
nova-conductor-5757498876-qgwh5               0/1     Init:0/1    1          3d6h
nova-consoleauth-55b5755f5b-8j7bz             0/1     Init:0/1    1          3d6h
nova-placement-api-6468cdf988-27p9c           0/1     Error       0          3d6h
nova-scheduler-6ddfb7b5d4-7bp5p               0/1     Init:0/1    1          3d6h
```

I manually ran the `./tools/deployment/developer/nfs/170-setup-gateway.sh` commands
(copy/pasted here so you can see):

```
#!/bin/bash

# Assign IP address to br-ex
OSH_BR_EX_ADDR="172.24.4.1/24"
OSH_EXT_SUBNET="172.24.4.0/24"
sudo ip addr add ${OSH_BR_EX_ADDR} dev br-ex
sudo ip link set br-ex up

# NOTE(portdirect): With Docker >= 1.13.1 the default FORWARD chain policy is
# configured to DROP, for the l3 agent to function as expected and for
# VMs to reach the outside world correctly this needs to be set to ACCEPT.
sudo iptables -P FORWARD ACCEPT

# Setup masquerading on default route dev to public subnet
DEFAULT_ROUTE_DEV="$(sudo ip -4 route list 0/0 | awk '{ print $5; exit }')"
sudo iptables -t nat -A POSTROUTING -o ${DEFAULT_ROUTE_DEV} -s ${OSH_EXT_SUBNET} -j MASQUERADE
```

I then cleaned up the `br-ex-dns-server` container:

```
$ sudo docker ps -a|grep br-ex
729b1d331994        openstackhelm/neutron:ocata   "dnsmasq --keep-in-f…"   3 days ago           Exited (255) 14 minutes ago                         br-ex-dns-server
$ sudo docker rm 729b1d331994
```

Then finished up the rest of the script:

```
# NOTE(portdirect): Setup DNS for public endpoints
sudo docker run -d \
  --name br-ex-dns-server \
  --net host \
  --cap-add=NET_ADMIN \
  --volume /etc/kubernetes/kubelet-resolv.conf:/etc/kubernetes/kubelet-resolv.conf:ro \
  --entrypoint dnsmasq \
  docker.io/openstackhelm/neutron:ocata \
    --keep-in-foreground \
    --no-hosts \
    --bind-interfaces \
    --resolv-file=/etc/kubernetes/kubelet-resolv.conf \
    --address="/svc.cluster.local/${OSH_BR_EX_ADDR%/*}" \
    --listen-address="${OSH_BR_EX_ADDR%/*}"
sleep 1
sudo docker top br-ex-dns-server
```

At this point, all the pods came up:

```
$ kk get po |grep -v Running|grep -v Comple
NAME                                          READY   STATUS      RESTARTS   AGE
```

I then tried a test VM:

```
$ nova boot fedora1 --flavor m1.xlarge --image "fedora19" --key-name dp-key1 --nic net-name=provider
$ nova list
+--------------------------------------+---------+--------+------------+-------------+----------------------+
| ID                                   | Name    | Status | Task State | Power State | Networks             |
+--------------------------------------+---------+--------+------------+-------------+----------------------+
| c60a24cf-9868-4fe8-b630-92f2bdbd7ac9 | fedora1 | ACTIVE | -          | Running     | provider=172.24.4.28 |
+--------------------------------------+---------+--------+------------+-------------+----------------------+
```

This shows that the openstack-helm host can survive a reboot.


## Debugging DNS

I'm having trouble getting the instance console to work in the horizon UI.  This section
mentions how I went about debugging it.

I access the horizon UI from the horizon NodePort at 31000:

```
$ kubectl get svc -n openstack |grep NodePort
horizon-int                   NodePort    10.101.138.0     <none>        80:31000/TCP                   3d8h
```

When I try to access an instance console, I get an error about not being able to resolve
`novncproxy.openstack.svc.cluster.local's server IP address could not be found`.

Looking at any relevant service:

```
$ kubectl get svc -n openstack |grep -i novncproxy                            
nova-novncproxy               ClusterIP   10.107.178.6     <none>        6080/TCP                       3d7h
novncproxy                    ClusterIP   10.107.156.255   <none>        80/TCP,443/TCP                 3d7h
```

One of the `ip netns` can resolve that IP:

```
$ kubectl get svc -n kube-system |grep dns
kube-dns              ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP              3d11h

$ ip netns
qdhcp-446d8b52-e4c9-4d54-a6d9-7830bd141ad8

$ sudo ip netns exec qdhcp-446d8b52-e4c9-4d54-a6d9-7830bd141ad8 cat /etc/resolv.conf
search svc.cluster.local cluster.local
nameserver 10.96.0.10
nameserver 8.8.8.8
nameserver 8.8.4.4
options ndots:5 timeout:1 attempts:1

$ sudo ip netns exec qdhcp-446d8b52-e4c9-4d54-a6d9-7830bd141ad8 ping novncproxy.openstack.svc.cluster.local
PING novncproxy.openstack.svc.cluster.local (10.107.156.255) 56(84) bytes of data.
```

I want to use `ping` to help debug but the `ping` executable is not present on the
pods I tried.  My solution to this is to copy the `ping` executable from the host to
one of the pods and ping from there to ensure I can resolve the
`novncproxy.openstack.svc.cluster.local` name:

```
$ kubectl cp `which ping` openstack/neutron-ovs-agent-default-rm8cv:/tmp
$ kk exec -ti neutron-ovs-agent-default-rm8cv -- /bin/bash
neutron@hstack1:/$ ldd /tmp/ping                                                                                             
	linux-vdso.so.1 =>  (0x00007ffd12f3c000)
	libcap.so.2 => /lib/x86_64-linux-gnu/libcap.so.2 (0x00007f06b305d000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f06b2c93000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f06b3263000)
```

The dynamic libraries resolve so ping should be good to go.

```
neutron@hstack1:/$ cat /etc/resolv.conf 
nameserver 10.96.0.10
search openstack.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
neutron@hstack1:/$ /tmp/ping novncproxy.openstack.svc.cluster.local
ping: icmp open socket: Operation not permitted
neutron@hstack1:/$ sudo /tmp/ping novncproxy.openstack.svc.cluster.local
[sudo] password for neutron: 
Sorry, try again.
[sudo] password for neutron: 
neutron@hstack1:/$ rm /tmp/ping 
```

That didn't work.

I start iterating through a bunch of pods to see if I can get ping to work.
But it seems I get the same issue above where I cannot use ping due to no
priveledges.

```
m1=aPodName
kubectl cp `which ping` openstack/$m1:/tmp/ping99
kk exec -ti $m1 -- ldd /tmp/ping99
kk exec -ti $m1 -- /tmp/ping99 8.8.8.8  ;# this gets error
kk exec -ti $m1 -- rm /tmp/ping99
```

I resort to making my own pod in the openstack namespace using a sample
[tool-container](https://github.com/dperique/tool-container/blob/master/centos-client.yaml)
modified to go into the `openstack` namespace (it's just Centos in a container which contains
ping):

```
apiVersion: v1
kind: Pod
metadata:
  name: dp-client
  namespace: openstack
spec:
  hostname: dp-client
  containers:
  - image: centos/python-35-centos7
    imagePullPolicy: Always
    name: dp-client
    command: [ "sleep", "999999" ]
```

I apply the yaml and try a ping:

```
$ kubectl apply -f x.yaml
pod/dp-client created

$ kk exec -ti dp-client -- /bin/bash

(app-root)bash-4.2$ cat /etc/resolv.conf
nameserver 10.96.0.10
search openstack.svc.cluster.local svc.cluster.local cluster.local
options ndots:5

(app-root)bash-4.2$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=53 time=0.959 ms
^C
--- 8.8.8.8 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.959/0.959/0.959/0.000 ms

(app-root)bash-4.2$ ping novncproxy.openstack.svc.cluster.local
PING novncproxy.openstack.svc.cluster.local (10.107.156.255) 56(84) bytes of data.
^C
--- novncproxy.openstack.svc.cluster.local ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 999ms
```

So I can ping the Internet and resolve the `novncproxy.openstack.svc.cluster.local`
variable.

## Getting the console to work

Continuing on from DNS debugging to finally getting console to work ...

Let's try to make the `novncproxy` a NodePort. I created the first file below by
doing `kubectl -n openstack edit svc novncproxy`.  I created the second one by copy/paste
and changing it to a NodePort type:

```
$ cat novncproxy.yaml
apiVersion: v1
kind: Service
metadata:
  name: novncproxy
  namespace: openstack
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  - name: https
    port: 443
    protocol: TCP
    targetPort: 443
  selector:
    app: ingress-api
  sessionAffinity: None
  type: ClusterIP

$ cat novncproxy-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: novncproxy
  namespace: openstack
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    nodePort: 30080
  - name: https
    port: 443
    protocol: TCP
    nodePort: 30443
  selector:
    app: ingress-api
  sessionAffinity: None
  type: NodePort

$ kk apply -f novncproxy-nodeport.yaml
service/novncproxy configured

$ kk get svc|grep novnc
nova-novncproxy               ClusterIP   10.107.178.6     <none>        6080/TCP                       3d18h
novncproxy                    NodePort    10.107.156.255   <none>        80:30080/TCP,443:30443/TCP     3d18h
```

Now I need to add the `novncproxy.openstack.svc.cluster.local` url to my `/etc/hosts` on the machine running my browser:

```
$ cat /etc/hosts
...
10.198.203.108   novncproxy.openstack.svc.cluster.local
...
```

When I click the console link from instances, I get farther but the console still does not load.
I have to make the link look like this:

```
http://novncproxy.openstack.svc.cluster.local:30080/vnc_auto.html?token=...&title=ubuntu-mk1(...)
```

instead of:

```
http://novncproxy.openstack.svc.cluster.local/vnc_auto.html?token=...&title=ubuntu-mk1(...)
```

Now I see my console.  At this point, I have a console though I need to tweak the url.  I'm ok with that.
