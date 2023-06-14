# Chapter 6. 프로세스 동기화 도구

## Why?

- race condition으로부터 데이터를 보호하기 위해 동기화와 조정이 필요함
    - 실행 결과가 접근 순서에 의해 변형될 수 있는 상황을 경쟁 상태(race condition)이라고 한다
    - **병행 수행중**인 **비동기적** 프로세스가 **공유 자원**에 **동시 접근**할 경우 문제가 발생할 수 있음
    - ex ) Thread unsafe

## 임계영역 문제 (Critical-Section Problem)

- 임계 영역(Critical Section)
    - 하나의 프로세스(스레드)만 데이터에 접근하고 갱신할 수 있는 부분
- 문제를 해결하기 위한 요구 조건
    - 상호 배제(mutual exclusion)
        - 오직 하나의 프로세스만 임계구역에서 진입할 수 있다
    - 진행(progress)
        - 임계구역에서 실행되는 프로세스가 없을 때, 다른 프로세스들이 임계구역에 접근하는 것이 연기되지 않는다
    - 한정된 대기(bounded waiting)
        - 어느 프로세스의 임계구역 진입 요청은 언젠가 허용되어야 한다(허용까지 다른 프로세스의 진입이 유한해야 한다)

## Peterson’s Solution

- 현대 컴퓨터 구조에서는 올바르게 실행된다는 보장은 없다
    - 컴파일러가 종속성이 없는 읽기 및 쓰기 작업은 재정렬할 수 있기 때문

![피터슨 솔루션](https://github.com/insukL/CS-STUDY/assets/66675919/4a8cf100-e542-4177-936e-045c66a1c0ed)

- 동시에 flag와 turn에 접근해도 turn이 결국 1가지 상태로 저장된다 ⇒ mutual exclusion 만족
- 임계 영역에 다시 접근해도 turn을 상대 프로세스에게 지정하면서 상대 프로세스가 임계 영역을 나갈 때까지 대기하게 된다 ⇒ progress, bounded waiting 만족
- 소프트웨어적인 방법으로 다익스트라 알고리즘이 추가적으로 있다
- 단점
    - 구현이 복잡하다
    - 속도 느리다
    - Busy waiting

## 하드웨어’s Solution

- 메모리 장벽
    - 메모리 접근 시 보장되는 사항을 결정한 방식을 **메모리 모델**이라고 한다
        - 메모리 가시성 : 한 프로세서의 메모리 변경이 다른 프로세스에게 보이는지
            - Java의 volatile
                - 값을 캐시에서 읽는 것이 아니라 매번 메인 메모리를 참조해온다
                - 값을 저장할 때도 매번 메인 메모리에 쓴다
                - 하나의 스레드에서 쓰고, 나머지가 읽는다는 전제
        - 강한 순서 : 한 프로세스 메모리 변경 결과가 다른 모든 프로세서에 즉시 보임
        - 약한 순서 : 한 프로세서의 메모리 변경 결과가 다른 프로세서에 즉시 보이지 않음
    - 메모리의 변경 사항을 다른 프로세서가 확인할 수 있도록 보장하는 명령어를 메모리 장벽이라고 한다
        - 변경 사항을 무조건 전파 ⇒ 변경 사항을 보장 ⇒ 해당 메모리에 명령어의 접근 순서 보장
- 하드웨어 명령어
    - 인터럽트 되지 않는(원자적으로 실행되는) 기계어
    - 매우 낮은 수준으로 다른 동기화 도구들을 구성하기 위한 기초로 사용됨
        - TAS(test_and_set)
            - 변경 이후 이전 값을 반환
            - 원자적 명령어
            
            ```c
            //함수가 한 번에 수행됨을 하드웨어적으로 보장
            boolean test_and_set (boolean *target) {
            	boolean temp = *target;  // 이전 값 기록
            	*target = true;          // target 값을 true로 설정
            	return temp;             // 기존 값 반환
            }
            ```
            
            - 명령어를 활용한 상호 배제
            
            ```c
            do {
            	while (test_and_set(&lock)); // 명령어가 원자적으로 실행, 하나만 진입 보장
            
            	/* critical section */
            
            	lock = false;                // 나오면서 해제
            
            	/* remainder section */
            
            } while (true);
            ```
            
        - CAS(compare_and_swap)
            - 비교 후 이전 값 반환
            - 원자적 명령어
            
            ```c
            int compare_and_swap (int *value, int expected, int new_value) {
            	int temp = *value;
            	
            	if (*value == expected) {
            		*value = new_value;
            	}
            
            	return temp;
            }
            ```
            
            - 명령어를 활용한 상호 배제
            
            ```c
            while (true) {
            	while (compare_and_swap(&lock, 0, 1) != 0);
            
            	/* critical section */
            
            	lock = 0;
            
            	/* remainder section */
            }
            ```
            
            - bounded waiting을 만족 시키려면 대기 중인 프로세스 리스트를 순회하도록 구성해야 함
        - 원자적 변수
            - CAS를 통해 값을 변경
                - 모든 Race Condition을 해결하진 못함
                    - 값 변경은 원자적으로 보장되지만, 참조하는 경우 문제가 발생할 수 있음
                    - ex) 소비자와 생산자 문제 ⇒ count가 0임에도 소비하는 경우가 발생
            - Java의 Atomic Type
                - CAS를 이용하여 동시성 보장

## OS Supported Solution

- Spin Lock (Mutex Lock)
    - 락의 가용 여부를 표시하는 변수를 가짐
    - 변수에 접근하는 함수는 OS에 의해 원자성 보장(CAS와 같은 연산)
    
    ```c
    acquire() {
    	while (!available)
    		;
    	available = false;
    }
    
    release() {
    	available = true;
    }
    ```
    
    - 프로세스는 가용 전까지 acquire를 계속 호출하며 **busy waiting**을 해야하는데, 락을 사용 가능할 때까지 프로세스가 회전하여 이러한 Mutex Lock의 형태를 **Spin Lock**이라고 한다
    - 멀티 코어 환경에서 Context Switching이 필요없어(busy waiting) 오버헤드가 적음
        - 다중 코어 컴퓨팅 시스템에서 널리 이용됨
- Semaphore
    - 정수 변수(S) 사용 - 초기화, wait(P), signal(V) 연산만 접근 가능
    - 동기화가 필요한 데이터마다 생성
    - Binary semaphore
        - S가 0과 1 두 종류만 갖는 경우
        - 상호 배제나 프로세스 동기화 목적으로 사용
    - Counting semaphore
        - S가 0 이상의 정수 값을 가질 수 있는 경우
        - Producer-Consumer 문제 등을 해결하기 위해 사용
    
    ```c
    init : S = 1
    wait(S) {
    	S--;                      // CAS
    	if( S < 0 ) {             // S가 음수가 되면 절댓값은 대기하는 프로세스 수
    		대기 큐에 프로세스 삽입  // PCB의 연결 필드에 의해 대기 큐 간단히 구현 가능
    		호출 프로세스 sleep
    	}
    }
    
    signal(S) {
    	S++;                     // CAS
    	if( S <= 0 ) {
    		대기 큐에서 프로세스 1개 꺼냄
    		꺼낸 프로세스 실행
    	}
    }
    ```
    

## High-level Mechanism

- Monitor
    - 언어에서 제공하는 상호배제 도구
        - Java에서의 synchronized 키워드
            - condition variable을 하나 가짐
            - wait, notify(signal), nootifyAll(broadcast)
        - 사용자에 의한 실수 예방
        - 내부에서는 상호배제가 보장
    - 구조
        - Mutual exclusion : 모니터 내에는 항상 하나의 프로세스만 진입 가능
        - Infromation hiding : 공유 데이터는 모니터 내의 프로세스만 접근 가능
        - entry queue : critical section 진입을 기다리는 큐
        - waiting queue(condition queue) : 조건이 충족되길 기다리는 큐
            
            ![모니터](https://github.com/insukL/CS-STUDY/assets/66675919/e2f16cdf-6e0c-4789-bfc2-b0afa2c7c5f1)
            
        
    
    ## 우선 순위 역전
    
    - 우선 순위가 높은 프로세스(H)의 대기가 우선 순위가 낮은 프로세스(L)에 의존하는 경우
        - 우선 순위가 다른 프로세스(M)이 자원을 선점하는 경우 L은 대기하게 되고 H는 우선 순위가 높음에도 L을 대기하기 때문에 무기한 대기가 발생
        - 이런 경우 L의 우선 순위를 H의 우선 순위로 임시로 승격하여 문제를 해결할 수 있음(우선 순위 상속)
