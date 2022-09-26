# Chapter 8. 카프카 버전 업그레이드와 확장

## 카프카 버전 업그레이드를 위한 준비
* 카프카 버전 업그레이드를 작업하기 전에 현재 사용하고 있는 버전을 확인해야 한다.
  * kafka-topics.sh 명령어를 이용하여 확인할 수 있다.
  * 카프카 폴더의 libs 하위 디렉토리에서 kafka의 jar 파일을 확인하여 버전을 확인할 수도 있다.
```text
./kafka-topics.sh --version
```
* 메이저 버전 업그레이드는 릴리즈 노트를 확인하여 호환성 이슈는 없는지 꼼꼼하게 확인해야 한다.
* 마이너 버전 업그레이드는 비교적 용이하게 업그레이드 할 수 있다.
* 카프카의 버전 업그레이드는 크게 다운타임을 가질 수 있는 경우와 다운타임을 가질 수 없는 경우로 나뉜다.
  * 대부분의 경우는 다운타임을 가질 수 없는 경우다.

### 최신 버전의 카프카 다운로드와 설치
* kafka의 /config/server.properties 설정 파일에는 아래의 옵션이 있다.
  * inter.broker.protocol.version=2.1
  * log.message.format.version=2.1
* 위 두 설정은 2.6 버전의 카프카가 실행되어도 브로커 간의 내부 통신은 2.1 버전 기반으로 통신하며, 메시지 포맷도 2.1을 유지한다는 의미다.
* 위 설정이 없다면 이미 실행중인 이전 버전의 브로커들과 통신이 불가능하다.

### 브로커 버전 업그레이드
* 브로커 버전 업그레이드는 한 대씩 순차적으로 진행한다.
* 브로커 버전을 업그레이드하기 위해 종료하면 종료된 브로커가 갖고 있던 파티션의 리더들이 다른 브로커로 변경된다.
  * 이때 클라이언트는 일시적으로 리더를 찾지 못하는 에러가 발생하거나 타임아웃 등이 발생한다.
  * 클라이언트는 내부적으로 재시도 로직이 있으므로 이후에 새로운 리더로 변경된 브로커를 바라보게 된다.
* 새로운 버전으로 업그레이드하더라도 이전에 inter.broker.protocol.version과 log.message.format.version 옵션을 설정했기 때문에 다른 버전의 브로커들과 정상적으로 통신이 가능하다.
* 나머지 브로커도 같은 방식으로 업그레이드를 진행한다.

### 브로커 설정 변경
* 업그레이드된 브로커들은 inter.broker.protocol.version과 log.message.format.version 옵션으로 인해 이전 버전으로 통신하도록 설정된 상태다.
* 위 설정을 제거하여 브로커들이 2.6 버전으로 통신할 수 있도록 변경한다.
* 설정을 변경한 후, 브로커를 한 대씩 재시작한다.

### 업그레이드 작업 시 주의사항
* 카프카의 사용량이 적은 시간대에 업그레이드 하는 것이 좋다.
  * 카프카 사용량이 적은 시간대여야 리플리케이션일 일치시키는 내부 동작이 신속하게 이뤄질 수 있다.
* 프로듀서의 ack=1 옵션을 사용하는 경우 카프카의 롤링 재시작으로 인해 일부 메시지가 손실될 수 있다.

## 카프카 확장
* 카프카는 폭발적으로 사용량이 증가하는 경우를 고려해 안전하고 손쉽게 확장할 수 있도록 디자인됐다.

### 브로커 부하 분산
* 전체 브로커들에게 토픽의 파티션을 고르게 부하 분산하기 위해서는 새로 추가된 브로커를 비롯해 모든 브로커에게 균등하게 파티션을 분산시켜야 한다.
* kafka-reassign-partitions.sh를 이용하면 파티션을 이동시킬 수 있다.
* 카프카 클러스터에 브로커를 추가한다고 해서 카프카의 로드가 자동으로 분산되지는 않는다.
  * 신규 브로커가 추가된 이후에 생성하는 토픽들은 신규 브로커를 비롯해 파티션들을 분산 배치한다.
* 브로커 간의 부하 분산 및 밸런스를 맞추려면 관리자는 기존 파티션들이 모든 브로커에 고르게 분산되도록 수동으로 분산 작업을 진행해야 한다.

### 분산 배치 작업 시 주의사항
* 분산 배치 작업을 수행할 때는 버전 업그레이드와 마찬가지로 카프카의 사용량이 낮은 시간에 진행하는 것이 좋다.
* 카프카에서 파티션이 재배치되는 과정은 파티션이 단순하게 이동하는 것이 아니라 브로커 내부적으로 리플리케이션하는 동작이 일어나기 때문이다.
* 분산 배치 작업 시, 파티션이 리플리케이션이 되는 것이기 때문에 브로커에 큰 부하를 주게 되며, 리플리케이션으로 인한 네트워크 트래픽 사용량도 급증하게 된다.
* 재배치하는 토픽의 메시지들을 모든 컨슈머가 모두 컨슘했고, 앞으로 재처리할 일이 없다면, 최근 메시지를 제외한 나머지 메시지들은 임시로 해당 토픽의 보관 주기를 짧게 변경하여 삭제하여 재배치로 발생하는 부하를 줄일 수도 있다.
* 파티션 재배치 작업 시 여러 개의 토픽을 동시에 진행하지 않고, 하나의 토픽만 진행하며 부하를 최소화하는 방법도 좋다.