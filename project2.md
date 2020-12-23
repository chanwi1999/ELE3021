> 이번 과제를 위해 작성한 부분은 모두 `project2` 라는 주석으로 표시

# 구현사항
## Pthread 
xv6용 pthread와 유사한 LWP 만들기
- 각 LWP는 주소 공간을 공유하지만 리소스는 다름
- 각 LWP는 독립 실행을 위해 별도의 스택 존재
- pthread의 경우 stack size, detach status, scheduling policy 등을 정할 수 있지만 우리는 간단히 속성만 같은 LWP 만들기가 목표

## MLFQ 와 stride scheduling
- 한 LWP 그룹 내의 LWP는 동일한 스케쥴링 하에 실행
- 그룹 내 전환은 1tick마다 발생
- LWP 간의 전환 발생 시 다른 LWP와 직접 전환해야 함 (LWP와 스케쥴러 사이의 전환은 없음)
- project1에 구현한 것을 재구현

### MLFQ scheduling
- 같은 그룹 내의 LWP는 quantum과 allotment를 공유함
- 스케쥴러가 LWP를 선택해 실행하고자 하면 LWP 그룹 내에서 quantum을 사용함
![image](/uploads/b8adcded34e411e50bda3e29aa903664/image.png)
- prior boosting 주기는 200ticks
- 각 queue는 Round Robin 정책을 따름

### stride scheduling
- 기본 quantum 5 ticks
- 같은 그룹 내의 LWP는 quantum을 공유함
- 스케쥴러가 LWP를 선택해 실행하고자 하면 LWP 그룹 내에서 quantum을 사용함
- LWP가 `set_cpu_share( )` 호출 시 해당 그룹의 LWP는 stride scheduler에 의해 관리
- 요청된 총 CPU양은 80% 초과 불가. 예외 처리 필요함

## noted item
- LWP는 동일한 address space 공유. proc struct를 사용해 LWP 구현 (주소 공간 공유 == 프로세스에 의해 생성된 LWP 간의 동일한 page table 공유)
- xv6의 fork, exec, exit, wait 기능에서 많은 힌트
- ptable용 lock 사용에 주의
- 리눅스 머신에서 Pthread를 이용해 다양한 test case

# The FirstMilstone: Design Basic LWP Operations

## Process 와 Thread
### process
![image](/uploads/ce16d00c3763ec1f749d76e9c244e10a/image.png)
- 자신만의 주소 공간을 가짐
- 실행해야할 작업들을 순차적으로 실행
- 실행 시 메모리 전체를 복사하고 주소 공간 전체를 컨텍스 스위치

### thread
![image](/uploads/64400ef7237d4be2bf0b275fa53a5297/image.png)
- 한 프로세스 내에서 독립적으로 실행되는 명령어 묶음
- 처음 시작 시 main()에서 시작되는 단일 스레드 상태로 작동
- 스레드는 프로세스 내에서 자원 공유, 독립적으로 실행되기 위해 필요한 자원(user stack)만 따로 가짐
- 공유 메모리에 자원을 두고 스레드들이 병렬적으로 작업 실행
- 같은 프로세스 내의 스레드 간의 컨텍스 스위치는 주소 공간 전체를 스위치하지 않음

## POSIX thread
thread를 제어하기 위한 함수들을 모아 놓은 C library
![image](/uploads/c57938c11a7539fae651f4ed05217735/image.png)

> master thread는 `pthread_create()`로 생성시킨 worker thread를 `pthread_join()`을 통해서 끝나길 기다림
> worker thread는 `pthread_create()`로 생성되어 작동한 뒤 `pthread_exit()`를 통해 종료 후 master thread에게 끝난 다는 것을 알림

`int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine)(void *),void *arg)`
새로운 스레드 생성. 성공 시 return 0
- thread: 스레드 id
- attr: 스레드 특성, default NULL
- start_routine: 스레드가 실행할 함수 포인터
- arg: start_routine 스레드 함수의 매개변수

`int pthread_join(pthread_t thread, void **ret_val)` 특정 스레드가 끝날 때까지 기다린 후 진행
기다릴 스레드 id 변수를 갖고 있는 thread, 스레드의 반환 값이 담긴 ret_val. 끝날 때까지 기다림, 종료 시킬 때 리소스 정리. 성공 시 return 0
- thread: 기다릴 스레드 id, `thread_create()`로 생성된 thread id
- ret_val: 스레드 반환 값의 포인터

`void pthread_exit(void *ret_val)` 실행 중인 스레드 종료.
- ret_val: `pthread_join( )` 의 ret_val 주소에 해당 매개변수가 들어가게 됨

## Design basic LWP operations for xv6

### struct proc
- struct proc *master: 스레드의 경우 자신의 master thread를 가르킴, master thread의 경우 0
- int total_th: worker thread 개수
- struct proc *thArr[NTHREAD]: worker thread의 tid를 index로 포인터 가짐
- void *ret: worker thread의 반환 값
- int tid: 스레드 id, worker thread가 아닐 경우 -1, master thread의 thArr[ ]의 index
- int idx: 스레드끼리 round robin하기 위함

### create functions and variables
> 이번 과제에 필요한 함수는 모두 `LWP.c`에 구현

- g_sz_list[NPROC*NTHREAD]: 할당 받은 스택을 해제할 때 시작 주소값 저장. 이후 다른 worker thread가 스택 할당 받을 때 재활용할 수 있게 함

`int Count_tid(struct proc*)` master thread의 thArr[ ]에 새로 생성된 스레드를 넣기 위한 인덱스를 찾아 반환, 이는 해당 스레드의 tid가 됨. return (-1,index)

`int Count_sz_empty(void)` 스레드에게 할당했던 스택을 해제하고 그 주소를 저장하기 위해 g_sz_list의 빈 공간의 가장 작은 index 반환

`int Count_sz_valid(void)` 스레드에게 스택을 할당하고자 할 때 해제되었던 주소를 이용하기 위해 g_sz_list의 값이 존재하는 공간의 가장 작은 index 반환

`struct proc* call_master(void)` 스레드의 master thread를 찾아주기 위함, 자신이 master thread의 경우 자신을 반환

`int Alloc_stack(struct proc*)` 스레드에게 개인적인 스택을 할당, 스레드 생성할 때 호출. return (-1,0)

`void Dealloc_stack(struct proc*)` 스레드에 할당했던 스택을 해제, 스레드 종료할 때 호출

`struct proc* Alloc_LWP(struct proc*)` 스레드 생성할 때 호출, allocproc( )과 코드 진행이 같으며 마지막에 스레드임을 나타내는 부분만 변경. return (0,struct proc*)

`RR_LWP(struct proc*)` 마스터 스레드가 작동할 차례에 하위 스레드가 존재한다면 round robin 방식으로 하위 스레드를 먼저 작동시킴. 하위 스레드의 실행 순서는 MLFQ_Scheduling()과 같은 방식으로 index를 계산하여 작동.

`void thread_clean(struct proc*)` 스레드 종료 이후 자원을 정리하는 용도

### define
- uint thread_t
- NTHREAD 20

# The SecondMilstone - First step: basic LWP operations

## system call
> `sysproc.c`에서 매개변수가 올바른지 체크하고 아래 함수들을 반환

> 아래 함수의 정의는 `LWP.c`에 작성

`int thread_create(thread_t *thread, void * (*start_routine)(void*), void *arg)` 프로세스 내에서 스레드 생성, 각 스레드에 할당된 실행 루틴 시작. 성공 시 return 0
- fork()와 exec() 참고
- 다른 프로세스와 마찬가지로 생성되지만 개인적인 스택을 가질 수 있도록 Alloc_LWP()와 Alloc_stack()를 이용하여 수행

>1. Alloc_LWP( )로 새로운 kstack 할당, pgdir은 master thread와 동일

>2. Alloc_stack( )으로 개인적인 ustack 할당

>3. 인자로 받은 *thread는 새로 생성된 스레드의 tid로 설정

>4. 생성된 스레드의 eip에 start_routine 저장

>5. arg를 start_routine( )의 인자로 주기 위해 ustack에 저장하고 esp에 새로 설정한 sp 저장

>6. 생성된 스레드의 상태를 RUNNABLE로 변경

>7. master thread의 total_th 업데이트

`void thread_exit(void *retval)` 스레드 종료, 스레드 루틴 마지막에 `thread_exit` 함수 호출, 스레드의 결과를 반환
- exit() 참고
- initproc과 wakeup1() 이용을 위해 `proc.c`에 구현
- exit()와 코드 진행이 같으며 마지막에 반환값을 저장하고 마스터 스레드의 total_th를 업데이트

>1. 현재 프로세스가 initproc과 같으면 오류, master thread라면 exit( )를 호출

>2. 열린 파일 모두 닫고 입출력 끔

>3. master thread를 wakeup1( )으로 깨움

>4. 부모가 현재 프로세스인 프로세스를 찾아 부모를 initproc로 설정하고, 상태가 ZOMBIE라면 initproc를 wakeup1( )으로 깨움

>5. 현재 프로세스의 ret은 인자로 받은 retval로 설정하고 상태를 ZOMBIE로 변경

>6. master thread의 total_th 업데이트

`int thread_join(thread_t thread, void **retval)` 지정된 스레드가 종료될 때까지 기다림, 스레드가 이미 종료된 경우 해당 스레드 반환. 성공 시 return 0
- wait() 참고
- wait()과 코드 진행이 같으며 state가 ZOMBIE일때 Dealloc_stack()을 이용해 스레드에 부여한 개인적인 스택 해제

>1. 스레드의 master와 현재 프로세스의 master가 같고, tid가 인자로 받은 thread와 같은 스레드 찾음

>2. 찾은 스레드의 상태가 ZOMBIE라면 인자로 받은 retval에 스레드의 ret를 저장

>3. Dealloc_stack( )을 이용해 할당해주었던 ustack을 해제

>4. 스레드가 사용되지 않도록 정보 초기화

>5. master thread는 sleep( )의 인자로 넣어줌

# The SecondeMilstone - Second step: interaction with other services in xv6
> 따로 지시사항이 없는 경우 pthread에 따름

## xv6와의 상호작용 - 추가하거나 변경한 부분만 작성
- basic operations: system call에 의해 생성된 LWP의 동작
address space 공유, 고유한 context와 stack 가짐, 리소스 공유할 경우 race condition 생성, 위의 함수 호출을 통해 LWP 관리
- `exit()`
LWP가 exit system call 호출하면 모든 LWP가 종료되고 각 LWP가 사용하던 리소스가 정리됨. 해당 커널을 나중에 다시 사용 가능. 호출 실행 후 LWP가 오래 생존 불가

>1. master의 하위 스레드들의 파일을 전부 닫고 이후 master의 파일도 전부 닫으며 입출력 역시 마찬가지로 전부 끔

>2. 하위 스레드의 상태를 모두 ZOMBIE로 변경

>3. master의 부모를 wakeup1( )으로 깨움

>4. 부모가 master와 같고 자신이 master인 프로세스를 찾아 부모를 initproc으로 설정하고 상태가 ZOMBIE이면 initproc을 wakeup1( )으로 깨움

>5. 리소스 정리는 wait( )에서 할 수 있도록 변경

- `exec()`
exec system call하면 모든 LWP의 리소스 정리.

>1. 호출한 프로세스가 master thread라면, 하위 스레드의 리소스를 모두 정리한 후 원래대로 실행

>2. 호출한 프로세스가 worker thread라면, 자신 제외 모든 worker thread를 모두 정리하고 master thread 위치에 자신을 넣고 이후 master thread 리소스 역시 정리

>+) 2번의 경우 정리하는 방식에 대한 아이디어는 `https://github.com/etual118/` 에서 얻었다.
![image](/uploads/e9eccc530ba26357f235e4ce43498fd4/image.png) 

- `sbrk()`
여러 LWP가 동시에 sbrk system call하여 메모리 영역 확장시 겹치기 불가, 요청된 크기와 다른 크기 할당 불가, 확장된 영역은 LWP 간의 공유
> growproc( )에서 master thread의 sz를 증가시키도록 변경

- `wait()`
> 프로세스의 하위 스레드가 존재할 경우 모든 리소스 정리

- `wakeup1()`
> 프로세스가 stride scheduling에 따라야 할 경우 g_min_pass와 해당 프로세스의 pass를 비교하여 해당 프로세스의 pass가 더 작을 경우 g_min_pass로 그 값을 변경

- `fork()`
여러개의 LWP가 동시에 fork system call해도 정상적으로 새로운 프로세스 생성. LWP address space 역시 정상적으로 복사. wait system call은 정상적으로 child 프로세스 기다림. fork 이후 프로세스 간의 parent, child를 기록

- `kill()`
둘 이상의 LWP가 소멸되면 모든 LWP 종료. 각 LWP 리소스 역시 정리.

- `pipe()`
모든 LWP는 pipe 공유. read, write, data를 동기화하고 복제는 불가

- `sleep()`
특정 LWP가 sleep system call하면 요청된 LWP만 요청된 시간 동안 sleep 상태. LWP 종료시 sleep LWP 역시 종료

- other corner cases

## project1에서 구현한 scheduler와의 상호 작용
- 동일한 프로세스에 속하는 모든 프로세스는 동일한 scheduler에 의해 스케쥴링
- stride와 MLFQ scheduler의 스레드는 origin 프로세스의 time slice 공유

### project1 오류 수정
![image](/uploads/49fbfd6cbe267a412fe36e6a1f4c1da2/image.png)
- g_min_pass 찾는 범위 설정 오류를 고침
- 여러 곳에서 g_MLFQ_ticks 초기화 하던 것을 고침
- quantum을 다 쓰지않고 yield()를 호출하는 것을 방지
- ptable을 돌면서 해당 proc의 scheduling을 호출하는 방식 대신 MLFQ_scheduling 해야하는 proc들을 하나의 stride로 보고 먼저 stride_scheduling 호출하는 방식으로 변경

### change/create functions and variables

`void remove_master(struct proc*, struct proc*)` exec()할 때 curproc의 master를 지우기 전에 master의 scheduling에 따라 stable과 FQ 정보 변경. 이후 master 0으로 설정

`struct proc* Stride_Scheduling(void)` 실행시킬 프로세스를 반환할 때 RR_LWP() 함수의 인자로 넣은 채 반환하도록 수정

`int OverTicks(void)` 현재 프로세스의 master thread의 pass와 tick을 증가시키고 project2의 MLFQ scheduling 규정에 맞춰 수정

# Result
## thread system call
![image](/uploads/a6d832a0654f5ae65a4136575dbbabaa/image.png)

## xv6와의 상호작용
![image](/uploads/e3ffa67e4b243db2ae8e3756fdc4ecd6/image.png)

## project1에서 구현한 scheduler와의 상호작용
![image](/uploads/d3a2fbf07093db76f35e983ad46bee99/image.png)
- test_thread2로 확인했을때 stridetest를 panic없이 통과
- project2의 MLFQ 구현 사항에 맞췄음으로 test_master의 결과 분포가 달라짐