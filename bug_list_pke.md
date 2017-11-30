## 1.安装neutron, openstack network agent list没有全部服务
在control和computer安装好了各项neutron时,要检查服务是否安装成功
```
 openstack network agent list
+--------------------------------------+----------------+------------+-------------------+-------+-------+------------------------+
| ID                                   | Agent Type     | Host       | Availability Zone | Alive | State | Binary                 |
+--------------------------------------+----------------+------------+-------------------+-------+-------+------------------------+
| 7b4fe3cc-a603-4a40-86c2-8b4c91b53eac | DHCP agent     | controller | nova              | :-)   | UP    | neutron-dhcp-agent     |
| e59964e5-63c3-4f89-a1ef-a27101ead395 | Metadata agent | controller | None              | :-)   | UP    | neutron-metadata-agent |
+--------------------------------------+----------------+------------+-------------------+-------+-------+------------------------+

```
可以看到有Dhcp和neutron-metadata-agent,还有三个服务没有在list上,说明配置可能有问题.

- 对着官网配置重新检查一遍
- 看log文件
最好看log文件,以我的linuxbridge为例
```
sudo cat /var/log/neutron/neutron-linuxbridge-agent.log


2017-10-17 21:22:11.776 5477 ERROR neutron.plugins.ml2.drivers.linuxbridge.agent.linuxbridge_neutron_agent [-] Parsing physical_interface_mappings failed: Invalid mapping: 'enp2s0'. Agent terminated!
2017-10-17 21:22:13.777 5530 INFO neutron.common.config [-] Logging enabled!
2017-10-17 21:22:13.778 5530 INFO neutron.common.config [-] /usr/bin/neutron-linuxbridge-agent version 11.0.0
2017-10-17 21:22:13.778 5530 ERROR neutron.plugins.ml2.drivers.linuxbridge.agent.linuxbridge_neutron_agent [-] Parsing physical_interface_mappings failed: Invalid mapping: 'enp2s0'. Agent terminated!
2017-10-17 21:22:15.788 5586 INFO neutron.common.config [-] Logging enabled!
2017-10-17 21:22:15.789 5586 INFO neutron.common.config [-] /usr/bin/neutron-linuxbridge-agent version 11.0.0
2017-10-17 21:22:15.789 5586 ERROR neutron.plugins.ml2.drivers.linuxbridge.agent.linuxbridge_neutron_agent [-] Parsing physical_interface_mappings failed: Invalid mapping: 'enp2s0'. Agent terminated!
2017-10-17 21:22:17.787 5639 INFO neutron.common.config [-] Logging enabled!
2017-10-17 21:22:17.788 5639 INFO neutron.common.config [-] /usr/bin/neutron-linuxbridge-agent version 11.0.0
2017-10-17 21:22:17.788 5639 ERROR neutron.plugins.ml2.drivers.linuxbridge.agent.linuxbridge_neutron_agent [-] Parsing physical_interface_mappings failed: Invalid mapping: 'enp2s0'. Agent terminated!

```
可以看到就是/etc/neutron/plugins/ml2/linuxbridge_agent.ini  physical_interface_mappings配置出错了,这里有个双网卡问题,这里是添provide 网卡的网络名称.
最最后如果全部正确的化,可以看到五个服务
```
openstack network agent list
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host       | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| 1d54bcce-86f0-4793-8ed4-7faecfa41abf | Linux bridge agent | controller | None              | :-)   | UP    | neutron-linuxbridge-agent |
| 4cf02dee-5402-49f3-930d-a2782321ff4a | Linux bridge agent | compute1   | None              | :-)   | UP    | neutron-linuxbridge-agent |
| 5500ac88-e41d-491a-9997-766ed2070b21 | L3 agent           | controller | nova              | :-)   | UP    | neutron-l3-agent          |
| 7b4fe3cc-a603-4a40-86c2-8b4c91b53eac | DHCP agent         | controller | nova              | :-)   | UP    | neutron-dhcp-agent        |
| e59964e5-63c3-4f89-a1ef-a27101ead395 | Metadata agent     | controller | None              | :-)   | UP    | neutron-metadata-agent    |
+--------------------------------------+--------------------+------------+-------------------+-------+---
```
## 2.安装horizon后,浏览器没有输入http://controller/horizon.无反应.
查看Apache2的日志
```
cat /var/log/apache2/error.log

 ...
  [client 10.0.0.106:53440] Timeout when reading response headers from daemon process 'horizon': /usr/share/openstack-dashboard/openstack_dashboard/wsgi/django.wsgi
```
修改编辑配置文件 /etc/apache2/apache2.conf，修改Timeout参数值，原来为300，现在设置为600
还是不行
```
cat /var/log/apache2/error.log

[wsgi:error] [pid 24783:tid 140477315581696] [client 10.0.0.106:53630] Timeout when reading response headers from daemon process 'horizon': /usr/share/openstack-dashboard/openstack_dashboard/wsgi/django.wsgi
[Wed Oct 18 17:44:43.498813 2017] [mpm_event:notice] [pid 24044:tid 140477675505536] AH00493: SIGUSR1 received.  Doing graceful restart
[Wed Oct 18 17:44:46.510271 2017] [wsgi:error] [pid 24674:tid 140477407901440] [client 10.0.0.106:53736] Truncated or oversized response headers received from daemon process 'horizon': /usr/share/openstack-dashboard/openstack_dashboard/wsgi/django.wsgi
[Wed Oct 18 17:44:46.517037 2017] [wsgi:warn] [pid 24044:tid 140477675505536] mod_wsgi: Compiled for Python/2.7.11.
[Wed Oct 18 17:44:46.517067 2017] [wsgi:warn] [pid 24044:tid 140477675505536] mod_wsgi: Runtime using Python/2.7.12.
[Wed Oct 18 17:44:46.520838 2017] [mpm_event:notice] [pid 24044:tid 140477675505536] AH00489: Apache/2.4.18 (Ubuntu) mod_wsgi/4.3.0 Python/2.7.12 configured -- resuming normal operations
[Wed Oct 18 17:44:46.520867 2017] [core:notice] [pid 24044:tid 140477675505536] AH00094: Command line: '/usr/sbin/apache2'

```
在文件/etc/apache2/conf-available/openstack-dashboard.conf中添加一句话：
                                          WSGIApplicationGroup %{GLOBAL}
   完美,在浏览器可以访问了                                     
3.建立实例
建立实例中出现实例一直是build状态.\
```
~$ openstack server list
+--------------------------------------+-------------------+--------+----------+--------+---------+
| ID                                   | Name              | Status | Networks | Image  | Flavor  |
+--------------------------------------+-------------------+--------+----------+--------+---------+
| ded7efc2-f25e-4530-bb6d-4ee4f9c7acca | provider-instance | BUILD  |          | cirros | m1.nano |
+--------------------------------------+-------------------+--------+----------+--------+---------+

```
检查了好几遍,发现是Nova中cell0链接到sqlite库,没有链接到sql
```
sudo nova-manage cell_v2 list_cells

+-------+--------------------------------------+------------------------------------+-------------------------------------------+
|  Name |                 UUID                 |           Transport URL            |            Database Connection            |
+-------+--------------------------------------+------------------------------------+-------------------------------------------+
|  cell | 0cf5658c-ba91-4672-85c2-0aeb2c88ade0 | rabbit://openstack:****@controller |     sqlite://var/lib/nova/nova.sqlite     |
| cell0 | 00000000-0000-0000-0000-000000000000 |               none:/               |  sqlite://var/lib/nova/nova.sqlite_cell0  |
| cell1 | 48a5ca68-77e8-4dd5-9f66-34f257f7667c | rabbit://openstack:****@controller | mysql+pymysql://nova:****@controller/nova |
+-------+--------------------------------------+------------------------------------+-------------------------------------------+
```
在compute结点查看/var/log/nova/nova-computer/
```
2017-11-13 10:36:45.819 29314 INFO nova.scheduler.client.report [req-d59224d1-4241-4fa3-8ddb-9873137b82dd 0247c48fb53049b8a0e063a22adea8ca 05d157ab676842a196575787a60a78fe - default default] Deleted allocation for instance ded7efc2-f25e-4530-bb6d-4ee4f9c7acca
2017-11-13 10:36:45.820 29314 WARNING nova.compute.manager [req-d59224d1-4241-4fa3-8ddb-9873137b82dd 0247c48fb53049b8a0e063a22adea8ca 05d157ab676842a196575787a60a78fe - default default] 2 consecutive build failures: InstanceActionNotFound_Remote: Action for request_id req-d59224d1-4241-4fa3-8ddb-9873137b82dd on instance ded7efc2-f25e-4530-bb6d-4ee4f9c7acca not found
Traceback (most recent call last):

  File "/usr/lib/python2.7/dist-packages/nova/conductor/manager.py", line 123, in _object_dispatch
    return getattr(target, method)(*args, **kwargs)

  File "/usr/lib/python2.7/dist-packages/oslo_versionedobjects/base.py", line 184, in wrapper
    result = fn(cls, context, *args, **kwargs)

  File "/usr/lib/python2.7/dist-packages/nova/objects/instance_action.py", line 169, in event_start
    db_event = db.action_event_start(context, values)

  File "/usr/lib/python2.7/dist-packages/nova/db/api.py", line 1957, in action_event_start
    return IMPL.action_event_start(context, values)

  File "/usr/lib/python2.7/dist-packages/nova/db/sqlalchemy/api.py", line 250, in wrapped
    return f(context, *args, **kwargs)

  File "/usr/lib/python2.7/dist-packages/nova/db/sqlalchemy/api.py", line 6155, in action_event_start
    instance_uuid=values['instance_uuid'])

InstanceActionNotFound: Action for request_id req-d59224d1-4241-4fa3-8ddb-9873137b82dd on instance ded7efc2-f25e-4530-bb6d-4ee4f9c7acca not found
2017-11-13 10:37:02.074 29314 INFO nova.compute.resource_tracker [req-cb24ea9b-0255-4c30-a7c7-bcfc916571b2 - - - - -] Final resource view: name=compute1 phys_ram=11894MB used_ram=512MB phys_disk=905GB used_disk=0GB total_vcpus=8 used_vcpus=0 pci_stats=[]

```
后续安装了neutron,发现删除不了cell0,最后直接重新安装nova在controller的所有服务
```
sudo nova-manage cell_v2 list_cells
[sudo] password for jingroup: 
+-------+--------------------------------------+------------------------------------+-------------------------------------------------+
|  Name |                 UUID                 |           Transport URL            |               Database Connection               |
+-------+--------------------------------------+------------------------------------+-------------------------------------------------+
| cell0 | 00000000-0000-0000-0000-000000000000 |               none:/               | mysql+pymysql://nova:****@controller/nova_cell0 |
| cell1 | a687b854-10a5-4bde-b937-b49c34634100 | rabbit://openstack:****@controller |    mysql+pymysql://nova:****@controller/nova    |
+-------+--------------------------------------+------------------------------------+-------------------------------------------------+

```
4.Nova发现Nova-computer出错
```
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting compute nodes from cell 'cell1': a687b854-10a5-4bde-b937-b49c34634100
An error has occurred:
Traceback (most recent call last):
  File "/usr/lib/python2.7/dist-packages/nova/cmd/manage.py", line 1699, in main
    ret = fn(*fn_args, **fn_kwargs)
  File "/usr/lib/python2.7/dist-packages/nova/cmd/manage.py", line 1485, in discover_hosts
    hosts = host_mapping_obj.discover_hosts(ctxt, cell_uuid, status_fn)
  File "/usr/lib/python2.7/dist-packages/nova/objects/host_mapping.py", line 223, in discover_hosts
    cctxt, 1)
  File "/usr/lib/python2.7/dist-packages/oslo_versionedobjects/base.py", line 184, in wrapper
    result = fn(cls, context, *args, **kwargs)
  File "/usr/lib/python2.7/dist-packages/nova/objects/compute_node.py", line 397, in get_all_by_not_mapped
    context, mapped_less_than)
  File "/usr/lib/python2.7/dist-packages/nova/db/api.py", line 273, in compute_node_get_all_mapped_less_than
    mapped_less_than)
  File "/usr/lib/python2.7/dist-packages/nova/db/sqlalchemy/api.py", line 265, in wrapped
    return f(context, *args, **kwargs)
  File "/usr/lib/python2.7/dist-packages/nova/db/sqlalchemy/api.py", line 725, in compute_node_get_all_mapped_less_than
    {'mapped': mapped_less_than})
  File "/usr/lib/python2.7/dist-packages/nova/db/sqlalchemy/api.py", line 664, in _compute_node_fetchall
    results = conn.execute(select).fetchall()
  File "/usr/lib/python2.7/dist-packages/sqlalchemy/engine/base.py", line 945, in execute
    return meth(self, multiparams, params)
  File "/usr/lib/python2.7/dist-packages/sqlalchemy/sql/elements.py", line 263, in _execute_on_connection
    return connection._execute_clauseelement(self, multiparams, params)
  File "/usr/lib/python2.7/dist-packages/sqlalchemy/engine/base.py", line 1053, in _execute_clauseelement
    compiled_sql, distilled_params
  File "/usr/lib/python2.7/dist-packages/sqlalchemy/engine/base.py", line 1189, in _execute_context
    context)
  File "/usr/lib/python2.7/dist-packages/sqlalchemy/engine/base.py", line 1398, in _handle_dbapi_exception
    util.raise_from_cause(newraise, exc_info)
  File "/usr/lib/python2.7/dist-packages/sqlalchemy/util/compat.py", line 203, in raise_from_cause
    reraise(type(exception), exception, tb=exc_tb, cause=cause)
  File "/usr/lib/python2.7/dist-packages/sqlalchemy/engine/base.py", line 1182, in _execute_context
    context)
  File "/usr/lib/python2.7/dist-packages/sqlalchemy/engine/default.py", line 470, in do_execute
    cursor.execute(statement, parameters)
  File "/usr/lib/python2.7/dist-packages/pymysql/cursors.py", line 166, in execute
    result = self._query(query)
  File "/usr/lib/python2.7/dist-packages/pymysql/cursors.py", line 322, in _query
    conn.query(q)
  File "/usr/lib/python2.7/dist-packages/pymysql/connections.py", line 856, in query
    self._affected_rows = self._read_query_result(unbuffered=unbuffered)
  File "/usr/lib/python2.7/dist-packages/pymysql/connections.py", line 1057, in _read_query_result
    result.read()
  File "/usr/lib/python2.7/dist-packages/pymysql/connections.py", line 1340, in read
    first_packet = self.connection._read_packet()
  File "/usr/lib/python2.7/dist-packages/pymysql/connections.py", line 1014, in _read_packet
    packet.check_error()
  File "/usr/lib/python2.7/dist-packages/pymysql/connections.py", line 393, in check_error
    err.raise_mysql_exception(self._data)
  File "/usr/lib/python2.7/dist-packages/pymysql/err.py", line 107, in raise_mysql_exception
    raise errorclass(errno, errval)
ProgrammingError: (pymysql.err.ProgrammingError) (1146, u"Table 'nova.compute_nodes' doesn't exist") [SQL: u'SELECT cn.created_at, cn.updated_at, cn.deleted_at, cn.deleted, cn.id, cn.service_id, cn.host, cn.uuid, cn.vcpus, cn.memory_mb, cn.local_gb, cn.vcpus_used, cn.memory_mb_used, cn.local_gb_used, cn.hypervisor_type, cn.hypervisor_version, cn.hypervisor_hostname, cn.free_ram_mb, cn.free_disk_gb, cn.current_workload, cn.running_vms, cn.cpu_info, cn.disk_available_least, cn.host_ip, cn.supported_instances, cn.metrics, cn.pci_stats, cn.extra_resources, cn.stats, cn.numa_topology, cn.ram_allocation_ratio, cn.cpu_allocation_ratio, cn.disk_allocation_ratio, cn.mapped \nFROM compute_nodes AS cn \nWHERE cn.deleted = %(deleted_1)s AND cn.mapped < %(mapped_1)s ORDER BY cn.id ASC'] [parameters: {u'mapped_1': 1, u'deleted_1': 0}]
```
是我controller结点的Nova数据没有同步,执行一下命令:
```
 su -s /bin/sh -c "nova-manage api_db sync" nova
root@controller:/home/jingroup# su -s /bin/sh -c "nova-manage db  sync" nova
/usr/lib/python2.7/dist-packages/pymysql/cursors.py:166: Warning: (1831, u'Duplicate index `block_device_mapping_instance_uuid_virtual_name_device_name_idx`. This is deprecated and will be disallowed in a future release.')
  result = self._query(query)
/usr/lib/python2.7/dist-packages/pymysql/cursors.py:166: Warning: (1831, u'Duplicate index `uniq_instances0uuid`. This is deprecated and will be disallowed in a future release.')
  result = self._query(query)
```
5.执行 nova-status upgrade check 出错
```
 nova-status upgrade check 
Error:
Traceback (most recent call last):
  File "/usr/lib/python2.7/dist-packages/nova/cmd/status.py", line 459, in main
    ret = fn(*fn_args, **fn_kwargs)
  File "/usr/lib/python2.7/dist-packages/nova/cmd/status.py", line 389, in check
    result = func(self)
  File "/usr/lib/python2.7/dist-packages/nova/cmd/status.py", line 133, in _check_cellsv2
    meta.bind = db_session.get_api_engine()
  File "/usr/lib/python2.7/dist-packages/nova/db/sqlalchemy/api.py", line 152, in get_api_engine
    return api_context_manager.get_legacy_facade().get_engine()
  File "/usr/lib/python2.7/dist-packages/oslo_db/sqlalchemy/enginefacade.py", line 791, in get_legacy_facade
    return self._factory.get_legacy_facade()
  File "/usr/lib/python2.7/dist-packages/oslo_db/sqlalchemy/enginefacade.py", line 342, in get_legacy_facade
    self._start()
  File "/usr/lib/python2.7/dist-packages/oslo_db/sqlalchemy/enginefacade.py", line 484, in _start
    engine_args, maker_args)
  File "/usr/lib/python2.7/dist-packages/oslo_db/sqlalchemy/enginefacade.py", line 506, in _setup_for_connection
    "No sql_connection parameter is established")
CantStartEngineError: No sql_connection parameter is established

```
需要root权限访问sql
```
sudo nova-status upgrade check 
[sudo] password for jingroup: 
+---------------------------+
| Upgrade Check Results     |
+---------------------------+
| Check: Cells v2           |
| Result: Success           |
| Details: None             |
+---------------------------+
| Check: Placement API      |
| Result: Success           |
| Details: None             |
+---------------------------+
| Check: Resource Providers |
| Result: Success           |
| Details: None             |
+---------------------------+

```
5.创建实例显示error
```
openstack  server list
+--------------------------------------+-------------------+--------+----------+--------+---------+
| ID                                   | Name              | Status | Networks | Image  | Flavor  |
+--------------------------------------+-------------------+--------+----------+--------+---------+
| 0ffe8f18-f99d-44f6-8aba-489d71bc2b56 | provider-instance | ERROR  |          | cirros | m1.nano |
+--------------------------------------+-------------------+--------+----------+--------+---------+

```
查看错误:
```
2015-08-25 09:42:53.476 49821 ERROR nova.scheduler.utils [req-75a79cbb-9f58-41ea-95ae-744436d34d9a ed9a79c01e2b423c8ba72afb862a6a3f b7be09574fd549acbf094ae25da91377 - - -] [instance: 54626e50-e17a-4bfc-9f0e-a6ed581582be] Error from last host: compute1 (node compute1): [u'Traceback (most recent call last):\n', u'  File "/usr/lib/python2.7/dist-packages/nova/compute/manager.py", line 2219, in _do_build_and_run_instance\n    filter_properties)\n', u'  File "/usr/lib/python2.7/dist-packages/nova/compute/manager.py", line 2362, in _build_and_run_instance\n    instance_uuid=instance.uuid, reason=six.text_type(e))\n', u'RescheduledException: Build of instance 54626e50-e17a-4bfc-9f0e-a6ed581582be was re-scheduled: Unexpected vif_type=binding_failed\n']
2015-08-25 09:42:53.479 49821 INFO oslo_messaging._drivers.impl_rabbit [req-75a79cbb-9f58-41ea-95ae-744436d34d9a ed9a79c01e2b423c8ba72afb862a6a3f b7be09574fd549acbf094ae25da91377 - - -] Connecting to AMQP server on controller:5672
2015-08-25 09:42:53.497 49821 INFO oslo_messaging._drivers.impl_rabbit [req-75a79cbb-9f58-41ea-95ae-744436d34d9a ed9a79c01e2b423c8ba72afb862a6a3f b7be09574fd549acbf094ae25da91377 - - -] Connected to AMQP server on controller:5672
2015-08-25 09:42:53.543 49821 WARNING nova.scheduler.utils [req-75a79cbb-9f58-41ea-95ae-744436d34d9a ed9a79c01e2b423c8ba72afb862a6a3f b7be09574fd549acbf094ae25da91377 - - -] Failed to compute_task_build_instances: No valid host was found. There are not enough hosts available.
```
大概是和上面的一样,就是No valid host was found. There are not enough hosts available.
修改/etc/nova/nova.conf
```
#enabled_filters = RetryFilter,AvailabilityZoneFilter,ComputeFilter,ComputeCapabilitiesFilter,
ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter

enabled_filters = AllHostsFilter

```
然后直接重启nova的所有服务,然后在创建实例就可以啦
```
openstack  server list
+--------------------------------------+--------------------+--------+---------------------+--------+---------+
| ID                                   | Name               | Status | Networks            | Image  | Flavor  |
+--------------------------------------+--------------------+--------+---------------------+--------+---------+
| 0ffe8f18-f99d-44f6-8aba-489d71bc2b56 | provider-instance  | ERROR  |                     | cirros | m1.nano |
| 74e69876-0855-4554-8db0-1d72cb7088e3 | provider-instance1 | ACTIVE | provider=10.0.0.154 | cirros | m1.nano |
+--------------------------------------+--------------------+--------+---------------------+--------+---------+

```
6.创建Ubuntu实例时,用ssh登入实例出错
```
ssh ubuntu@10.0.0.160
Permission denied (publickey).
```
一直是Permission denied .解决方法就是从新生成key
```
ssh-keygen -q -N ""
$ openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
```
但是好像出现新问题:
```
 ssh ubuntu@10.0.0.160
The authenticity of host '10.0.0.160 (10.0.0.160)' can't be established.
ECDSA key fingerprint is SHA256:mONiGPVvYRhk2AUnoJ+jlg5pJgsiOrgarMheTrcbm38.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.0.0.160' (ECDSA) to the list of known hosts.
sign_and_send_pubkey: signing failed: agent refused operation
Permission denied (publickey).
```
这个就是在controller结点执行以下命令
```
ssh-add
```
之后就可以成功登入Ubuntu的实例了.
总结:
断断续续的花费了两个月的时间,终于把openstack安装成功了,终于可以开通实例了.
最好的安转资料还是官网的install guide,其他的中文博客只能作为参考(大多数的博客记录的不是很,很多是想我一样的初学者边学边记录).
出错了,第一是不要烦躁,慢慢的分析其原因,如果确实困扰的比较久,就先放一放.
二.bug的第一准确信息是在log文件中,仔细分析log文件中的error和warring.一般是这里的信息比较精准的定位你的错误.可以网上搜搜bug解决方法(建议放狗搜).
三.很多问题是相应的服务的配置文件不对,多多检查配置文件.

参考资料:
https://docs.openstack.org/install-guide/launch-instance-provider.html
http://www.aboutyun.com/home.php?mod=space&uid=1310&do=blog&quickforward=1&id=3126