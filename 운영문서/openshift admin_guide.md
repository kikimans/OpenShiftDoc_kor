# OpenShift 운영자 가이드

이 문서는 2.x 버전의 운영자 가이드 문서 입니다.

## 1. 사용자 자원 관리

### 1.1 Gear 할당량과 사이즈 설정

사용자의 기본 gear 할당량과 사이즈  /etc/openshift/broker.conf 환경 파일에 설정을 한다.
위 파일은 broker 호스트에 존재한다.

```
# cat /etc/openshift/broker.conf
```

* *VALID_GEAR_SIZES* : 사용자에게 지원되는 gear 크기 == openshift 에서는 small, medium, large == 까지 설정이 가능하다.
* *DEFAULT_MAX_GEARS* : 사용자 생성시 사용자에게 할당하도록 지정한다. 사용자가 생성할 수 있는 총 gear 수
* *DEFAULT_GEAR_SIZE* : 사용자 생성시 기본으로 부여되는 기어 크기

== 기본적으로 openshift 에서는 사용자 생성시 할당하는것은 small gear 에 100개의 기어를 생성할 수 있다 ==

``` add) DEFAULT_MAX_DOMAINS - 사용자에게 부여되는 도메인 총 갯수 ```

### 1.2 사용자 생성 기어 숫자 설정

모든 사용자의 설정을 변경 하지 않고 특정 사용자의 기어의 숫자를 변경 하려고 할때 명령어를 제공한다.

```
# oo-admin-ctl-user

# oo-admin-ctl-user -l <username>

user <username> :
	consumed gears:0
    max gears:100
	gear sizes:small

# oo-admin-ctl-user -l <username> --setmaxgears 25

Setting max_gears to 25... Done.

user <username> :
	consumed gears:0
    max gears:25
	gear sizes:small
```

### 1.3 사용자 gear Type 변경

```
# oo-admin-ctl-user -l <username> --addgearsize <medium|large>
```

현재 default는 small gear 타입으로 되어 있지만 사용자의 따라서 small 이외의 다른 기어를 추가하려면 위 명령어 대로 처리 해주면 된다.

결과
```
user <username> :
	consumed gears:0
    max gears:25
	gear sizes:small, medium
```

추가된 기어 타입 삭제

```
# oo-admin-ctl-user -l <username> --removegearsize <small|medium|large>
```
---
## 2. 노드 자원 활용
OpenShift는 효율적인 자원 관리를 위해 호스트 노드간의 gear 이동 및 기어를 더 이상 생성하지 않도록 비활성화도 가능하다.

### 2.1 OpenShift의 엔티티 계층구조

OpenShift 구조 파악을 위해서는 gear 와 cartridge의 의미를 이해하고 있어야 한다.

* gear : 하나 이상의 cartridge 인스턴스를 포함하고 있는것
* node host : 기어를 포함하고 있는 linux 사용자
* District : node host 와 node host에 포함되어 있는 gear

== node host 또는 district는 하나의 gear size를 가질 수 있다 ==

즉 하나의 호스트는 하나의 gear size를 가질 수 밖에 없는것이다.

### 2.2 Districts의 목적
노드의 자원 사용을 관리 할 수 있는 신뢰성에 목적을 두고 있다.

기어는 node host의 리눅스 사용자 ID(UID) 에 따라서 외부포트 범위 및 IP 어드레스 범위를 포함하여 자원을 할당한다.

### 2.3 브로커의 Districts 의 활성화

Districts를 사용하려면 브로커의 Mcollective 플러그인은 District를 사용하도록 구성해야 한다.
```
/etc/openshift/plugins.d/openshift-origin-msg-broker-mcollective.conf
```
위 파일에 대한 환경 설정이 필요 하다.

broker host의 변경
```
DISTRICTS_ENABLED=true
NODE_PROFILE_ENABLED=true
```
위 설정은 기본 설정 변경 내용이구 추가적으로 하나의 환경 변수가 있는지 확인해서 아래와 같이 수정해야 한다.

```
DISTRICTS_REQUIRE_FOR_APP_CREATE=true
```
위 설정은 districts 가 완료 되지 않는 node host에는 app 이 생성이 되는것을 방지 하기 위한 설정이다.

### 2.4 Districts 생성 및 입력

Districts를 생성하기 위해서는 oo-admin-ctl-district를 이용하여 생성할 수 있다.

```
# oo-admin-ctl-district -c create -n small_district -p small
```

```
-c : command 옵션으로
(add-node|remove-node|deactivate-node|activate-node|add-capacity|remove-capacity|create|destroy|publish-uids) 등을 사용할 수 있다.

-n : districts 이름
-p : node profile(gear size or profile)
```
### 2.4.1 브로커에서의 Districts 의 표시
실제로 위의 행위는 mongodb에 districts collections에 새로운 값을 입력 한 것이다.
```
# mongo -u <mongousername> -p <mongodpassword> <mongodatabase name>
```
```
> show collections or db.getCollectionsNames()
```

```
[
	"Account",
	"applications",
	"authorizations",
	"cloud_users",
	"districts",
	"domains",
	"locks",
	"system.indexes",
	"system.users",
	"usage",
	"usage_records"
]
```

```
> db.districts.find()
```

==생성된 정보는 mongodb의 districts collections으로 확인이 가능하다.==

### 2.4.2 Node Host 추가

```
# oo-admin-ctl-district -c add-node -n small_district -i node.example.com
```

```
-i : node host의 hostname
```

### 2.5 District로부터 node 제거

관리적인 측면에서 호스트의 문제나 특이한 상황에서 노드 호스트를 제거해야 하는 상황에서는 아래와 같은 명령어로 처리가 가능하다.

```
oo-admin-ctl-district -c deactivate-node -n small_district -i node.example.com
```
일단 먼저 node host 가 활성화 상태을 비 활성화 상태로 전환 한다.

==추가적으로 oo-admin-move를 통해 기존 node host에 있는 gear 정보를 다른 node host로 옮겨준다.==

node 삭제
```
oo-admin-ctl-district -c remove-node -n small_district -i node.example.com
```

### 2.6 District 제거
```
oo-admin-ctl-district -c remove-capacity -n <district_name> -s 6000
```
먼저 district 에 capacity size를 0 으로 처리 해주어야 한다.

그리고 2.5 에서 설명 한것 처럼 모든 기어와 노드를 제거 합니다.

```
oo-admin-ctl-district -c delete -n <district_name>
```

### 2.7 기어(gear) 용량 관리
Districts 와 node host는 허용 기어수의 대한 다른 2가지 용량 제한을 두고 있다. Districts는 할당된 UID 풀을 고정하고, 상태와 상관없이 기어를 6000개 까지 포함할 수 있다.
Node Host의 용량은 host의 활성 기어의 수를 제한한다.

### 2.7.1 노드 호스트
```
/etc/openshift/resource_limits.conf
```
노드 호스트는 위 파일의 max_active_gear 의 설정 값으로 활성 기어수를 제한 한다.
==기본값은 100== 이지만 관리 측면에서 이 부분은 대부분 수정이 되어야 한다. 하지만 정지 또는 유휴기어를 계산하지 않기 때문에 노드가 비 활성화 기어를 가지는것이 가능하다. 또한 한계에 도달 후에 비활성 기어를 시작하여 한계를 초과 할 수도 있다. 이것이 초과되면 단순히 브로커에서 이 노드의 기어 생성을 하지 않는 것 뿐이다.

사용제한을 두는 가장 안정적인 방법은 아래와 같다.
예를 들어 8GB 호스트에 기어당 512MB 의 메모리를 부여 하였다면

```
max_active_gear = 8GB/512MB = 16gears
```

그러나 실제로는 대부분의 기어는 전체 자원 할당을 사용하지 않을 것이다. 효율적인 운영적인 측면에서는 모든 자원을 사용하면 맞을것 보다 더 많은 기어를 허용하여 시스템의 적어도 일부를 오버 커밋 실험을 통해서 관리자가 시스템의 한계를 증가여 오버 커밋 퍼센트를 결정하여야 한다.

이것들은 메모리 이외의 CPU, 네트워크 대역폭 , 프로세스, 등등의 다른 자원과 카트리지의 및 Application의 유형의 따라 기준을 정해야 한다.

### 2.7.2 District
현재의 제약 때무에 District는 6000개의 gear만 포함 할 수 있다. 그것은 District의 많은 노드 호스트를 추가하는것은 중요하지 하는 이유는 district가 UID pool를 모두 소비하면 추가 적인 gear 생성은 안되기 때문에 자원의 낭비가 발생 될 수 있다.

District는 gear의 이동을 용이 하게 하기 위해 존재 하는것이다.

District당 필요한 노드 계산법

```
N = 6000* 활성노드기어퍼센트 / (100* max_active_gear)
```
예를 들면 활성 노드 기어 퍼센트율을 10%로 잡고 최대 기어 수는 50개로 설정을 한다면 
```
6000*10 / (100*50) = 12
```
District 당 12개의 노드 호스트가 필요한 것이다.

### 2.7.3 District 와 기어의 사용량 확인
기어와 District의 사용량을 볼 수 있는 명령어
```
oo-stats
```

### 2.8 노드 간의 기어 이동
노드간의 기어 이동은 `oo-admin-move` 를 브로커 서버에서 사용하면 된다.

```
oo-admin-move --gear_uuid <uuid> -i <nodehost_name>
```
== 같은 gear size district 간의 이동만 가능하다 ==

---

## 3. 카트리지 추가
브로커는 좀 더 빨리 카트리지 정보를 제공하고 설치 하기 위해 캐쉬 처리를 한다.
예를 들면 MCollective 노드가 가능한 카트리지 목록을 캐싱한다. 새로운 카트리지 설치시 이게 다시 업데이트 되지 않는한 새로운 카트리지는 목록에서 확인이 되지 않아 사용자는 사용을 할 수 없다.

### 3.1 origin cartridge 리스트 업
```
# yum search origin-cartridge
```
```
openshift-origin-cartridge-cron.noarch : Embedded cron support for OpenShift
openshift-origin-cartridge-diy.noarch : DIY cartridge
openshift-origin-cartridge-haproxy.noarch : Provides HA Proxy
openshift-origin-cartridge-jbossews.noarch : Provides JBossEWS2.0 support
openshift-origin-cartridge-jenkins.noarch : Provides jenkins-1.x support
openshift-origin-cartridge-jenkins-client.noarch : Embedded jenkins client
                                                 : support for OpenShift
openshift-origin-cartridge-mock.noarch : Mock cartridge for V2 Cartridge SDK
openshift-origin-cartridge-mock-plugin.noarch : Mock plugin cartridge for V2
                                              : Cartridge SDK
openshift-origin-cartridge-mysql.noarch : Provides embedded mysql support
openshift-origin-cartridge-nodejs.noarch : Provides Node.js support
openshift-origin-cartridge-perl.noarch : Perl cartridge
openshift-origin-cartridge-php.noarch : Php cartridge
openshift-origin-cartridge-postgresql.noarch : Provides embedded PostgreSQL
                                             : support
openshift-origin-cartridge-python.noarch : Python cartridge
openshift-origin-cartridge-ruby.noarch : Ruby cartridge
```

### 3.2 새로운 cartridge 등록
openshift 2.1 버전 부터는 아래의 명령어가 추가 되었다.
```
oo-admin-ctl-cartridge
```
브로커 호스트에서 카트리지를 활성화 및 비 활성화가 가능하다.

### 3.3 Customer 카트리지 설치 방법
Customer 카트리지 설치시 패키지 혹은 단순한 소스로 되어 있다면 각 노드 host에 등록해야 한다.
```
oo-admin-cartridge -a install -s <source_path>
```
카트리지가 설치되면 설치 경로 위치는 ==/var/libexec/openshift/cartridges== 에 설치 된다.

### 3.4 카트리지 삭제
위에서 언급한 oo-admin-ctl-cartridge는 브로커에서 단순히 카트리지를 활성화, 비 활성화를 시키는 거지만 

실제로 삭제를 하려면 아래와 같이 진행 하면 된다.
```
oo-admin-cartridge -a erase -n <cartrdge_name> -v <version> -c <cartridge_version>
```

```
# oo-admin-cartridge --list

(jyes, egovframework, 3.0, 0.0.4.1)
(redhat, php, 5.3, 0.0.8.2)
(redhat, jenkins, 1, 0.0.6)
(redhat, mock-plugin, 0.1, 0.0.1)
(redhat, mock-plugin, 0.2, 0.0.1)
(jyes, cubrid, 9.2, 0.1.0)
(redhat, jenkins-client, 1, 0.0.5)
(redhat, nodejs, 0.10, 0.0.8)
(redhat, postgresql, 8.4, 0.3.6)
(redhat, postgresql, 9.2, 0.3.6)
(redhat, perl, 5.10, 0.0.7.2)
(redhat, jbosseap, 6, 0.0.8)
(redhat, cron, 1.4, 0.0.8)
(redhat, python, 2.6, 0.0.8.2)
(redhat, python, 2.7, 0.0.8.2)
(redhat, mysql, 5.1, 0.2.6)
(redhat, ruby, 1.8, 0.0.10.2)
(redhat, ruby, 1.9, 0.0.10.2)
(redhat, diy, 0.1, 0.0.5)
(redhat, jbossews, 1.0, 0.0.9)
(redhat, jbossews, 2.0, 0.0.9)
(redhat, haproxy, 1.4, 0.0.9.1)

cartridge_name : php
version : 5.3
cartridge_version : 0.0.8.2

```
---

## 4. 관리자 콘솔
openshift는 관리자 콘솔을 제공한다. 하지만 콘솔은 외부로 공개 되지 않고 port 포워딩을 통해서 접근이 가능하다.

```
ssh root@borker host -L 8080:localhost:8080 -N
```
노드 정보와 gear 의 정보를 간략하게 볼 수 있는 정도로 제공한다.











