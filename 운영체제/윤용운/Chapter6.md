# Chapter6. 동기화 도구들
## 6.1. 배경
- 프로세스가 병렬, 혹은 병행 실행될 때 데이터의 무결성 문제가 발생할 수 있다.
- 여러개의 프로세스가 동일한 자료를 접근하여 조작하는 상황 : race condition
  - 한순간에 하나의 프로세스만이 데이터에 접근할 수 있도록 보장해야함
    - 프로세스 동기화 필요.
      - 동기화 : 프로세스 사이의 동작을 맟추는 것. 혹은 서로 정보를 공유하는 것

## 6.2. 임계구역 문제
![image](https://github.com/VSFe/Tech-Interview/assets/46861704/773dd061-964e-45f2-8e9c-10b9626a0f8f)
- 각 프로세스는 임계구역(critical section)이라 부르는 코드를 포함.
  - 안에서 하나 이상의 다른 프로세스와 공유하는 데이터에 접근/갱신 가능
  - 한 프로세스가 자신의 임계구역에서 수행되는 동안 다른 프로세스들은 임계구역 접근 불가
- 임계구역 문제
  - 프로세스들이 데이터를 협력적으로 공유하기 위해 자신들의 활동을 동기화 할 때 사용할 수 있는 프로토콜을 설계하는 것
- 임계구역 문제 해결책
  - 상호 배제(mutual exclusion) => Mutex
    - 프로세스가 자기 임계구역에서 실행되면, 다른 프로세스의 진입을 금지
  - 진행(progress)
    - 임계구역 안에 있는 프로세스 외에는 다른 프로세스가 임계구역에 진입하는것을 방해하면 안됨
  - 한정된 대기(bounded waiting)
    - 프로세스의 임계구역 진입은 유한시간 내에허용되어야 한다.

## 6.3. Peterson의 해결안
```java
while(true) {
  flag[i] = true; // 프로세스 준비 상태
  turn = j; // 진입할 프로세스
  while (flag[j] && turn == j);
  // critical section
  flag[i] = false;
  // remainder section
}
```
- 두 프로세스(Pi, Pj)가 두개의 데이터 항목(flag[], turn)을 공유하도록 하여 해결
  - flag : 프로세스 준비 상태
  - trun : 임계구역에 진입할 프로세스

- Dijksta's Algorithm
  - n개의 프로세스간의 임계구역 문제 해결 방법

- SW solution 문제점
  - 속도가 느림
  - 구현 복잡
  - Mutex 실행 중 선점될 수 있음
    - 공유데이터 수정 중 인터럽트 억제 => 오버헤드
  - Busy waiting
    - 기다리는 동안 while문을 계속 돌고있음

## 6.4. 동기화를 위한 하드웨어 지원
### 메모리 장벽
- 메모리 모델 : 컴퓨터 아키텍처가 응용 프로그램에게 제공하는 메모리 접근 시 보장되는 사항을 결정한 방식
  - 강한 순서 : 한 프로세서의 메모리 변경 별과가 다른 프로세서에게 즉시 보임
  - 약한 순서 : 한 프로세서의 메모리 변경 결과가 다른 프로세서에 즉시 보이지 않음

- 메모리 장벽 : 메모리의 모든 변경사항을 다른 모든 프로세서로 전파하는 명령어 제공

### 하드웨어 명령어
- atomic instruction : 하드웨어에서 보장하는 실행도중 다른 instruction이 끼어들지 않도록 보장하는 명령. 
- TAS(TestAndSet)
  - test와 set 한번에 수행하는 기계어
  - 기계어 => 인터럽트를 받지 않음(선점되지 않음)
  - 프로세스 3개 이상이면 Bounded waiting 위배
- CAS(CompareAndSwap)
  - 3개의 변수 사용

- 원자적 변수
  - 정수, bool과 같은 기본 데이터 유형에 대한 원자적 연산을 제공
  - 모든 상황에서 race condition 해결 못함
  - 
  
### Mutex Lock(Spin Locks)
```java
acquire() {
  while (!available); // busy waiting
  available = false;
}

release() {
  available = true;
}

while (true) {
  // acquire lock
    critical section
  // release lock
    remainder section
}
```
- `available`이라는 boolean 변수 가짐
- 단일코에에서는 오버헤드가 크지만, 다중코에에서는 오버헤드가 작다
  - 한 스레드가 락을 걸면 다른 스레드는 다른 코어에서 스핀하고 있으면 되기 때문에 context switching이 필요없음

### 세마포어
- mutex와 유사하지만, 프로세스들이 자신들의 행동을 좀 더 정교하게 동기화 가능
- S : 정수 변수. 초기화 제외하고 두개의 표준 원자적 연산 wait() (P), signal() (V) 로만 접근 가능
- 카운팅 세마포어
  - 제한 없는 영영을 가짐
- 이진 세마포어
  - 0, 1만 가능
  - 스핀락과 비슷한 동작
- 세마포어 구현
```java
S = 0;
wait(S) {
  S--;
  while(S <= 0) {
    list.add(S)  // 임의의 큐잉 전략 사용 가능
    sleep();
  }
}

signal(S) {
  S++
  if (S <= 0) {
    process = list.get()  
    wakeup(process);
  }
}
```

## 모니터
- 언어에서 제공하는 도구 (java에선 synchronized)
- 사용자의 실수를 예방 해줌
- 내부에서 상호배제 보장


## 라이브니스
- 프로세스가 실행 주기동안 진행되는 것을 보장하기 위해 충족해야 하는 일련의 속성
- 교착상태
  - 프로세스가 다른 프로세스가 선점하고 있는 자원을 무한히 대기하는 상황
- 우선순위 역전
  1. Low, Middle, High 프로세스가 있음
  2. Low가 세마포어 S를 가지고 있고, H는 S가 필요 -> H 대기
  3. 이때 S가 필요하지 않은 Middle 프로세스가 ready가 되고, Low 프로세스는 Middle프로세스에 의해 선점
  4. H는 계속 대기해야됨 
  - 셋 이상의 우선순위를 가진 운영체제에서 발생
  - 우선순위 상속 프로토콜을 구현하여 해결(Low 프로세스를 High우선순위로 상속해버림)
