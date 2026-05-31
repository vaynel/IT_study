# NIC 1개 환경에서 OpenStack External Network 구성 정리
## 개요

물리 NIC가 **1개뿐인 서버**에서 OpenStack(Kolla + Neutron + Open vSwitch)을 구성할 경우,  
외부 네트워크(External Network, Floating IP)는 **물리 NIC가 아닌 OVS 브리지(br-ex)** 를 통해 종단된다.

이 문서는 **NIC 1개 환경에서 외부 네트워크를 안정적으로 구성하는 정석 구조**를 정리한다. 


## 핵심 개념 요약

- 물리 NIC는 **외부 네트워크로 나가는 물리적 uplink**
- 실제 IP / Floating IP 트래픽의 종단점은 **OVS internal bridge (`br-ex`)**
- 물리 NIC는 **절대 down 하면 안 되며**, IP를 가지지 않는다
- Floating IP는 **물리 NIC가 아니라 Neutron External Network에서 할당**된다
  
## 전체 네트워크 구조

[ VM ]  
|  
[ qrouter namespace ]  
|  
[ br-ex (OVS internal bridge) ] ← External Network 종단  
|  
[ enp1s0 (물리 NIC, OVS slave) ] ← 단순 uplink  
|  
[ 외부 스위치 / 공유기 / 인터넷 ]  

## 인터페이스 역할 정리

| 구성 요소 | 역할 |
|---------|------|
| enp1s0 | 물리 NIC (uplink 전용, IP 없음) |
| br-ex | External Network 종단 (IP/Floating IP 트래픽 처리) |
| qrouter | Floating IP NAT / 라우팅 담당 |
| VM | Floating IP를 통해 외부와 통신 |

---

## 왜 물리 NIC를 br-ex로 "옮기는가?"

NIC가 1개인 환경에서는 다음 요구사항이 동시에 존재한다.

- OpenStack 관리 트래픽
- API 트래픽
- External Network(Floating IP)
- 내부 Tenant Network

이를 하나의 NIC로 처리하기 위해:

- **물리 NIC(enp1s0)를 OVS 브리지(br-ex)에 slave로 연결**
- **IP 및 라우팅은 br-ex에서 처리**

이 방식이 **OVS + Kolla 기준 유일하게 안정적인 구조**이다.

---

##  Openstack 설치 방법 
1. 물리 NIC로 설치 진행
2. ssh 연결 종료 발생 (물리 NIC가 가상 NIC로 변경됨)
3. 서버 PC에 가서 네트워크 설정 변경

## 상세 방법
### 첫번째 deploy 실행
```bash
neutron-external-network = 물리 NIC
```

### Open vSwitch 구성

```bash
# 필자의 물리 NIC = enp1s0

# IP는 br-ex에만
ip addr flush dev enp1s0
ip addr add 192.168.35.153/24 dev br-ex
ip route replace default via 192.168.35.1

# OVS 구성
ovs-vsctl del-br br-ex
ovs-vsctl add-br br-ex
ovs-vsctl add-port br-ex enp1s0
ip link set br-ex up
```


### 두번째 deploy 실행
global.yaml
```bash
neutron_bridge_name: br-ex
tunnel_interface: br-ex
neutron-external-network = br-ex
```

multinode
```bash
controller network_interface=물리NIC 
compute network_interface=br-ex neutron_external_interface=물리NIC
```

---
