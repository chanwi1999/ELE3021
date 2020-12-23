# The First Milestone

## First step: MLFQ
- 3개의 allotment가 다른 queue가 있다.
- 각 레벨의 queue 내부 프로세스는 quantum에 맞춰 Round-Robin에 따른다.

## Second step: Combine the stride scheduling algorithm with MLFQ
- cpu time은 최소 20% 이상 MLFQ scheduling을 사용하며, 최대 80%까지 stride scheduling을 사용할 수 있다. (80% 초과 요청시 예외 처리)


# Design

## MLFQ
![image](/uploads/fc1a91b6ffbe90cc59f3c175898d6737/image.png)
- xv6기준 눈금 1개는 약 10ms이다.
- 눈금 100개(즉 1000ms)의 cpu time이 소모되었다면 우선 순위를 높여서 진행한다. (priority boost)
- 각 queue가 사용할 수 있는 cpu time은 high priority부터 눈금 5개, 10개로 한다.
- 각 queue 내부 프로세스는 RR scheduling에 따르며 그 주기는 high priority부터 눈금 1개, 2개, 4개로 한다.
- 항상 최소 20% 이상의 cpu를 사용한다.

## Stride scheduling
- 프로세스가 cpu 지분을 요청하면 system call로 cpu 지분을 설정한다.
- 새로 생성된 프로세스는 MLFQ에 진입하며, `set_cpu_share( )` 함수가 호출될 경우 stride scheduling으로 전환한다.
- stride queue의 총 stride cpu 지분은 80%를 넘길 수 없으며, 그럴 경우 MLFQ로 수행한다.

+) stride를 갖는 프로세스들을 따로 queue에 저장하는 아이디어는 `https://github.com/etual118/` 에서 얻었다.

## Required system calls
`int sys_yield( )` 다음 프로세스에게 cpu 양보, 전환

`int sys_getlev( )` 현재 프로세스가 MLFQ의 어떤 레벨의 queue에 있는지. return MLFQ_level

`int sys_set_cpu_share( )` wrapper

`int set_cpu_share( int )` 지분 요청 후 stride queue에 삽입. cpu 전체 기준 80% 초과 요청 시 stride queue에 들어가지 못함. return (-1/0)

## Create functions and variables
- 'scheduling.c' 에 알고리즘을 위해 필요한 함수와 변수를 정의한다.

### Functions

`int Push_MLFQ(struct proc*, int)` 원하는 레벨의 MLFQ에 해당 프로세스를 새로 삽입. 프로세스가 새로 생성되거나  prior boost가 발생 시 사용. return (-1,0) 

`int Pop_MLFQ(struct proc*)` MLFQ에서 해당 프로세스를 지움. 프로세스가 끝나거나 stride queue으로 이동하거나 prior boost 발생 시 사용. return (-1,0)

`int Push_Stride(struct proc*, int)` 해당 프로세스를 stride scheduling으로 진행하기 위해 stride queue에 삽입. 해당 프로세스가 삽입 된 stride queue의 index를 sched로 설정. return (-1,0)

`int PriorityBoost( )` 주기마다 MLFQ를 prior boost. return (-1,0)

`CountingShareStride(int)` stride queue에서 유효한 프로세스들의 지분과 새로 요청한 지분을 모두 합산하여 지분 계산. 그에 따른 MLFQ의 지분도 변경. 계산 과정에서 가장 적은 pass를 가진 프로세스를 저장. 프로세스를 새로 생성하거나 종료했을 때 마다 호출.

`int OverTicks(struct proc* p)` 해당 프로세스가 cpu를 점령했던 시간이 MLFQ 각 레벨의 기준을 넘어가면 다음 레벨로 넘기거나 총 cpu 점령 시간이 100ticks가 되면 prior boost 호출. Round-Robin

`struct proc* MLFQ_Scheduling( )` MLFQ 에서 작동시킬 다음 프로세스를 선택해서 반환. queue 내에서도 보다 공정하기 위해서 idx 변수를 이용해 간단히 계산.

`struct proc* Stride_Scheduling( )` stride queue에서 pass가 작은 프로세스를 반환. 만약 MLFQ의 pass가 가장 작다면 MLFQ scheduling( )을 호출해 다음 프로세스를 반환. MLFQ에 실행시킬 프로세스가 없다면 다시 stride queue에서 선택 후 반환.

`int Add_ticks( )` trap이 발생하여 `yield( )`를 호출하기 전에 pass를 올림. MLFQ의 경우MLFQ_ticks도 올림. 다만 컴파일 하는 과정에서도 프로세스가 생성되고 그에 따라 MLFQ에 삽입되기 때문에 MLFQ_ticks이 상승함. 따라서 순수하게 MLFQ만이 cpu를 사용한 시간이라고 보기 어려움.

### Global variables
- int g_MLFQ_ticks: ( MLFQ에서 ) cpu를 사용한 시간, 100이 넘으면 boosting
- int g_total_share_stride: stride queue의 총 지분
- int g_min_pass: stride queue에서 가장 작은 pass
- int g_min_index: stride queue에서 min_pass를 가진 프로세스의 index

### define
- TICKETS 1000000: stride계산용
- MAX 999999999: min_pass 초기화 밎 overflow 기준

## queue struct
- int total_proc: 현재 queue에 속한 프로세스의 개수
- int idx: pArr에서 가장 최근에 실행시킨 프로세스의 index
- proc* pArr[NPROC]: 현재 queue에 속한 프로세스 배열

## stride struct
- int stride
- int pass
- int share
- int valid: 종료된 프로세스의 경우 0
- proc* sProc: stride scheduling으로 실행해야 하는 프로세스

## proc struct
- int MLFQ_level: stride queue의 경우 3, 종료된 프로세스의 경우 -1
- int sched: MLFQ의 경우 0, 종료된 프로세스의 경우 -1
- int ticks: round robin 알고리즘에 이용

# Algorithm
![image](/uploads/10567ea7369d3fcae9a36ad4e328642a/image.png)
![image](/uploads/bf7bc0dbb5cc6fec8d712fdfc1d51d91/image.png)
![image](/uploads/2b2abe95d58a53ee5b9f08946b86eb74/image.png)

- 새로 생성된 프로세스는 `allocproc( )` 을 통해 MLFQ queue level 0에 넣는다.
(sched=0)
- 첫번째 프로세스의 경우 `userinit( )` 에서 MLFQ queue와 stride queue를 초기화한다.
- `set_cpu_share( )`가 호출되면 MLFQ queue에서 지우고 stride queue에 해당 프로세스를 넣는다.
(MLFQ_levle=3 / sched=stride index)
- `scheduler( )` 함수가 ptable을 순차적으로 돌며 프로세스의 sched 변수를 확인하여 알맞은 scheduling을 호출하여 실행한다.
(sched=0; MLFQ_scheduling / else stride scheduling)
- 실행이 끝난 프로세스는 `Pop_MLFQ( )` 또는 stride valid 변수를 통해 더이상 선택되지 않도록 한다.
(sched=-1 / valid = 0)

# Result
## basic test code
![image](/uploads/8cff8c69416d4d7df09556208270c99d/image.png)

- lev[1]이 lev[0] 보다 값이 크지만 예상한 것만큼 큰 차이를 보이지 않았다. 가끔 lev[0]의 값이 더 큰 경우도 발생했다. 그에 비해 lev[2]는 예상한 것처럼 변함없이 가장 큰 값을 보여주었다.
- MLFQ_ticks이 온전히 MLFQ만의 cpu 점령 시간을 의미하지 않게 되면서 프로세스의 ticks 자체도 길어진 듯 하다.  trap의 발생으로 'yield( )'를 호출 해야할 때 먼저 quantum을 다 썼는지 확인하는 방식을 취했다.

![image](/uploads/792b893b52433442c2d021f2a1ca4179/image.png)
- test_stride는 예상대로 정상 작동했다.


### quantum 체크한 후에 `yield( )` 호출
![image](/uploads/38b4a45c6bd51b5a302122190326e00e/image.png)
![image](/uploads/a600743548e7531b8b1dbcfd733a2275/image.png)


### quantum 체크하지 않고 `yield( )` 호출
![image](/uploads/6b031da6f930b5dc9c460c13c69c0f56/image.png)
- trap의 발생으로 'yield( )'를 호출 해야할 때 quantum을 다 썼는지 먼저 확인하지 않았더니 위와 같은 결과가 나왔다. lev[1] 이 큰 경우가 자주 발생함을 볼 수 있다.
- test_master를 실행시켰을 때 부터 MLFQ_ticks을 상승시킬 수 있었다면 조금 더 바랐던 결과에 가까웠을 것 같다.