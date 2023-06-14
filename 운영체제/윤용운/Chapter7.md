# Chapter7. 동기화 예제
## 7.1. 고전적인 동기화 문제들
- 세마포어, mutex 등으로 해결 가능한 문제들
### 유한 버퍼 문제
- 유한한 개수의 물건을 임시로 보관하는 버퍼
- 생산자가 버퍼에 물건을 넣을때 버퍼가 가득 찼거나, 소비자가 버퍼에서 물건을 꺼낼 때 비어있다면?
- 다음과 같은 자료구조를 공유한다.
```java
int n;
semaphore mutex = 1 // 이진 세마포어
semaphore empty = n // 비어있는 버퍼의 개수 트래킹
semaphore full = 0; // 차 있는 버퍼의 개수 트래킹
```

생산자와 소비자 프로세스는 다음과 같이 버퍼에 동기화한다.
```java
// 생산자 프로세스
while(true) {
  ...
  // 아이템 생산
  ...
  wait(empty);  // 버퍼에 빈 공간 생길 때 까지 대기
  wait(mutex);  // critical section에 진입할 수 있을 때 까지 대기
  ...
  // 아이템을 버퍼에 추가
  ...
  signal(mutex);  // critical section에 빠져나왔다고 알림
  signal(full);   // 버퍼에 아이템이 있다고 알림
  ...
}

// 소비자 프로세스
while(true) {
  wait(full);  // 버퍼에 아이템이 생길 때 까지 대기
  wait(mutex);  // critical section에 진입할 수 있을 때 까지 대기
  ...
  // 아이템을 버퍼에서 가져옴
  ...
  signal(mutex);  // critical section에 빠져나왔다고 알림
  signal(empty);   // 버퍼에 아이템이 있다고 알림
  ...
  // 아이템 소비
  ...
}
```
### Readers-Writers 문제
- 하나의 데이터베이스가 병행 프로세스간 공유되고, writer와 reader가 동시에 접근하면 발생함(reader 2개는 상관없음)
- writer가 쓰기 작업 동안 해당 데이터에 배타적 접근 권한 부여하여야 한다.
- 여러가지 변형이 존재
  1. writer가 허가를 못받았을 때 reader들이 안기다려도 된다.
  2. writer가 접근하게 되면 새로운 reader들은 읽기 불가
  - 문제의 해결안이 존재. 하지만 기아현상 발생 가능

- 1번 문제 해결 방법 -> writer가 기아현상 발생 가능
```java
int read_count = 0;      // 현재 몇개의 프로세스가 데이터를 읽고 있는지
semaphore rw_mutex = 1;  // writer를 위한 상호배제 세마포어. reader, writer가 모두 공유
semaphore mutex = 1;     // read_count 갱신 시 상호배제를 보장하는 세마포어.

// writer
while (true) {
  wait(rw_mutex);
  ...
  // writing
  ...
  signal(rw_mutex);
}

// reader
while (true) {
  wait(mutex);
  read_count++;
  if (read_count == 1) {  // reader없었기 때문에 writer가 writing 할 가능성이 있음.
    wait(rw_mutex);
  }
  signal(mutex);
  ...
  // reading
  ...
  wait(mutex);
  read_count--;
  if (read_count == 0) {  // reader가 아무도 없기 때문에 writer가 writing 가능
    signal(rw_mutex);
  }
  signal(mutex);
}
```
- 몇몇 운영체제에서는 reader-writer 락을 제공함.
- 이런 상황에서 유용함
  - 어떤 프로세스가 read할지 write 할지 명확히 식별 가능
  - writer보다 reader가 많은 경우
    - reader-writer락을 설정할 때 드는 오버헤드가 세마포어나 mutex락을 설정할 때 보다 크지만, 동시에 여러 reader가 병행하게 읽게 하는 방법 등을 사용하여 오버헤드 상쇄 가능

### 식사하는 철학자들 문제
![image](https://github.com/YunYongWoon/CS-STUDY/assets/46861704/72cf0bcb-5431-4725-abf3-da002e82128f)

- 철학자들은 생각과 식사를 한다. 생각할때는 다른 철학자들과 상호작용을 하지 않는다
- 식사시에는 양 옆의 젓가락을 사용해야한다. 한번에 한개의 젓가락을 들 수 있다. 식사를 끝마치면 다시 젓가락을 내려놓고, 생각을 한다.
- 교착상태와 기아현상을 방지해야 하는 경우를 단순화 한 예시.
- semaphore 사용방법
  - 각 젓가락을 세마포어로 만듬.
  - 문제점 : 동시에 모든 철학자가 배고파져서 왼쪽 젓가락을 들게 된다면? -> 데드락.
  - 해결방법
    - 4명의 철학자만 테이블에 앉게 함.
    - 한 철학자가 젓가락 두개를 모두 집을 수 있을 때만 젓가락을 집도록 허용
    - 비대칭 해결안. 홀수 번호는 왼쪽 젓가락을 우선, 짝수 번호는 오른쪽 젓가락을 먼저 집는 방법 등.
  - 교착상태는 해결하지만, 기아현상의 가능성이 남아 있음.
- motitor 해결 방법.
  - 철학자의 상태를 나타내는 자료구조 도입
    - `enum {THINK, HUNGRY, EAT} state[5]`
  - 철학자는 양쪽 철학자가 식사하지 않을 때만 EAT 상태로 진입 가능
  - 젓가락을 못 얻었을 때 잠시 대기할 수 있게 해줄 변수 추가
    - `condition self[5]`
  - 해당 해결방법 역시 기아현상 가능성 존재
```java
monitor DiningPhilosophers {
  enum {THINKING, HUNGRY, EATING} state[5];
  condition self[5];
  
  void pickup(int i) {
    state[i] = HUNGRY;
    test(i);
    if (state[i] != EATING) {
        self[i].wait();
    }
  }

  void putdown(int i) {
    state[i] = THINKING;
    test((i + 4) % 5)
    test((i + 1) % 5)
  }
  
  void test(int i) {
    if ((state[(i + 4) % 5] != EATING) && (state[i] == HUNGRY) && (state[(i + 1) % 5] != EATING)) {
      state[i] = EATING;
      self[i].signal();
    }
  }
  
  initialization_code() {
    for (int i = 0; i < 5; ++i) {
      state[i] = THINKING;
    }
  }
}
```

## 7.4. Java에서의 동기화
### Monitor
- BoundedBuffer 클래스는 생산자-소비자 문제의 해결안을 구현하며, insert(), remove() 메소드 호출
```java
public class Boundedbufer{
  private static final int BUFFER_SIZE = 5;

  private int count, In, out;
  private E[] buffer;

  public BoundedBuffer() {
    count = 0;
    in = 0;
    out = 0;
    buffer = (E[]) new Object(BUFFER_SIZE)
  }

  // 생산자가 해당 메소드 호출
  public synchronized void insert(E item) {
    while (count == BUFFER_SIZE) {
      try {
        wait();
      } catch (InterruptedException e) {}
    }
    buffer[in] = item;
    in = (in + 1) % BUFFER_SIZE;
    count++;

    notify();
  }

  // 소비자가 해당 메소드 호출
  public synchronized E reomve() {
    E item;
    while (count == 0) {
      try {
        wait();
      } catch (InterruptedException e) {}
    }
    item = buffer[out];
    out = (out + 1) % BUFFER_SIZE;
    count--;

    notify();

    return item;
  }
}
```
- Java의 모든 객체는 하나의 락과 연결되어 있고, `synchronized` 명령문으로 선언된 메소드를 호출하려면 해당 객체와 연결된 락을 소유해야 한다. (블록 동기화(메소드 안에서 synchronized 선언) 또한 가능)
- 객체의 락에는 진입집합과 대기집합이 존재한다.
  ![image](https://github.com/YunYongWoon/CS-STUDY/assets/46861704/010893d8-6a4a-48a3-9c60-30fed158fdef)  
  - wait()
    - 스레드가 객체 락 해제
    - 스레드 상태가 봉쇄됨으로 설정
    - 스레드는 객체의 대기 집합에 넣어짐
  - notify()
    - 대기 집합의 임의의 스레드 T 선택
    - 스레드 T를 대기 집합에서 진입 집합으로 이동
    - T의 상태를 봉쇄됨에서 실행 가능으로 설정

### 재진입 락
- API에서 사용가능한 가장 간단한 락인 `ReentrantLock`
- `lock()` 메소드를 호출하여 `ReentrantLock`을 획득 (`unlock()`을 호출하여 반환)
### 세마포어
- 카운팅 세마포어도 제공
- 생성자 : `Semaphore(int value);`
  - value : 세마포의 초기 값

```java
Semaphore sem = new Semaphore(1);

try {
  sem.acquire();
  // critical section
} catch (InterruptedException e) {
  // 락을 획득하려는 스레드가 인터럽트되면 발생
} finally {
  sem.release();
}
```
### 조건 변수
- `ReentrantLock` 객체 생성 후 `newCondition()` 메소드를 호출하여 `Condition` 객체를 획득
- `wait()`, `notify()` 메서드와 유사한 기능 제공