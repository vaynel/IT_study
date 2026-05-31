# Deploy 실행시 발생한 에러 
## openvswitch : Ensuring OVS ports are properly setup error
**원인** : br-ex가 브릿인데, br-ex라는 포트도 있어서 꼬인 상황  
**에러로그**
```bash
TASK [openvswitch : Ensuring OVS ports are properly setup] **************************************************************************************************************************************************************************
skipping: [compute01] => (item=['br-ex', 'br-ex']) 
skipping: [compute01]
[ERROR]: Task failed: Module failed: ovs-vsctl: cannot create a port named br-ex because a bridge named br-ex already exists
Origin: /root/openstack/venv/share/kolla-ansible/ansible/roles/openvswitch/tasks/post-config.yml:39:3

37       or (inventory_hostname in groups["compute"] and computes_need_external_bridge | bool )
38
39 - name: Ensuring OVS ports are properly setup column 3

failed: [controller] (item=['br-ex', 'br-ex']) => {"action": "openvswitch_port", "ansible_loop_var": "item", "changed": false, "cmd": "/usr/bin/ovs-vsctl -t 5 add-port br-ex br-ex", "item": ["br-ex", "br-ex"], "msg": "ovs-vsctl: cannot create a port named br-ex because a bridge named br-ex already exists", "rc": 1, "stderr": "ovs-vsctl: cannot create a port named br-ex because a bridge named br-ex already exists\n", "stderr_lines": ["ovs-vsctl: cannot create a port named br-ex because a bridge named br-ex already exists"], "stdout": "", "stdout_lines": []}
[WARNING]: Could not match supplied host pattern, ignoring: enable_openvswitch_True_enable_ovs_dpdk_True
```
### 해결 방법
globals.yml
```bash
# neutron_external_interface
위 주석처리
```
multinode
```bash
각 node neutron_external_interface=물리NIC로 설정
```


## nova-cell : Waiting for nova-compute services to register themselves error
**원인** : nova-cell: from_json “Expecting value …” (JSON 파싱 실패)  
- ansible.builtin.from_json failed: Expecting value ...
- nova_compute_services.stdout | from_json에서 깨짐  



```bash
TASK [nova-cell : Waiting for nova-compute services to register themselves] *********************************************************************************************************************************************************
[ERROR]: Task failed: The filter plugin 'ansible.builtin.from_json' failed: Expecting value: line 1 column 1 (char 0)

Task failed.
Origin: /root/openstack/venv/share/kolla-ansible/ansible/roles/nova-cell/tasks/wait_discover_computes.yml:25:7

23     expected_compute_service_hosts: "{{ virt_compute_service_hosts + ironic_compute_service_hosts }}"
24   block:
25     - name: Waiting for nova-compute services to register themselves column 7

<<< caused by >>>

The filter plugin 'ansible.builtin.from_json' failed: Expecting value: line 1 column 1 (char 0)
Origin: /root/openstack/venv/share/kolla-ansible/ansible/roles/nova-cell/tasks/wait_discover_computes.yml:53:11

51         # could exclude services that are down, the nova-manage cell_v2
52         # discover_hosts does not do this so let's not block on it here.
53         - (nova_compute_services.stdout |
             ^ column 11

fatal: [compute01 -> controller]: FAILED! => {"changed": false, "msg": "Task failed: The filter plugin 'ansible.builtin.from_json' failed: Expecting value: line 1 column 1 (char 0)"}
```

### 해결방법
```bash
# root로 파일 소유권/권한 정리
docker exec -u 0 -it kolla_toolbox bash -lc '
ls -al /etc/kolla/admin-openrc.sh || true
chown root:root /etc/kolla/admin-openrc.sh
chmod 600 /etc/kolla/admin-openrc.sh
ls -al /etc/kolla/admin-openrc.sh
'

# root로 openstack token issue 테스트
docker exec -u 0 -it kolla_toolbox bash -lc '
set -e
source /etc/kolla/admin-openrc.sh
openstack token issue
'
```