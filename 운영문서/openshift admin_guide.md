# OpenShift 운영자 가이드

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









