> 과제를 위해 수정하거나 새로 구현한 부분은 `project3`로 주석 표시

# OverView
## File system
- 데이터를 쉽게 저장하고 검색할 수 있도록 함으로써 storage에 대한 효율적이고 편리한 접근 제공
- 파일 시스템이 어떻게 사용자를 바라보고 알고리즘을 만드는지
- 어떻게 물리 저장매체 secondary 장치에 logical file system을 매핑하는지

## File
- 저장매체의 물리적 속성으로 부터 추출하여 논리 저장 단위인 파일 정의
- 운영체제에 의해 물리 장치에 매핑(대부분 비휘발성)

## inode
- 각 파일은 i number(== inode number)라고 불리는 정수 번호로 식별되는 inode와 연관
- 각 inode는 파일 데이터의 속성 및 디스크 블록 위치 저장
- inode 번호는 장치의 알려진 위치에 있는 inode table index
- inode 번호로 커널의 파일 시스템 드라이버는 파일 위치를 포함한 inode 컨텐츠에 접근 하여 파일 접근

## sync
- unix OS에서 파일 쓰기는 쓰여진 데이터의 내구성을 보장하지 않음
- 파일 쓰기 작업 중 buffer cache에 데이터 기록
- 동기화 작업으로 buffer cache의 모든 데이터가 storage에서 내구성 유지

# The Milestone 1
- 파일의 최대 크기 확장
![image](/uploads/774430cd5b430559a739d27111547fff/image.png)

## tips
- FSSIZE 40000
- struct inode
- struct dinode
- some defined constants
- `static uint bmap(struct inode *ip, uint bn)`
- `static void itrunc(struct inode *ip)`
![image](/uploads/a7a804145a8b8ea76b4a70838f86a0d9/image.png)

## define
- FSSIZE 40000: file size 확장
- NDIRECT 10: doubly indirect와 triple indirect 새로 정의 함에 따라 변경
- NDINDIRECT (NINDIRECT * NINDIRECT): doubly indirect
- NTINDIRECT (NINDIRECT * NINDIRECT * NINDIRECT): triple indirect
- MAXFILE (NDIRECT + NINDIRECT + NDINDIRECT + NTINDIRECT): doubly indirect와 triple indirect 새로 정의 함에 따라 변경

## Change variables and functions
- inode와 dinode의 addrs[NDIRECT+3]

`static uint bmap(struct inode *ip, uint bn)` bn으로 새로 할당할 data block이 어디에 해당하는지 확인. 기존의 함수 로직에 따름
![image](/uploads/c18d2e8da4963c3d3d5677bb0abc1ea6/image.png)


`static void itrunc(struct inode *ip)` 단계별로 unlink하며 가장 끝부분부터 차례로 끊어줌. 기존의 함수 로직에 따름
![image](/uploads/34b6b09b48dcec54427aba3ef3bb007b/image.png)

# The Milestone 2
- pread, pwrite system call 구현

## tips
https://linux.die.net/man/2/pread

## System call
: 기존의 write, read system call과 유사하며 반환 값으로 새로 생성한 pfilewrite()와 pfileread()를 반환

`int pwrite(int fd, void* addr, int n, int off)` 파일에서 원하는 offset에서 부터 write 가능. lseek + write와 같은 역할

`int pread(int fd, void* addr, int n, int off)` 파일에서 원하는 offset에서 부터 read 가능. lseek + read와 같은 역할

## Create fucntions
> `file.c` 와 `fs.c`에 구현

`int pfilewrite(struct file*, char*, int, int)`: filewrite()와 코드가 동일하며 단, writei 대신 새로 생성한 pwritei로 log에 기록. pwritei 호출 시 offset을 f->off가 아닌 인자로 받은 off 사용. 오류가 났을 때 구분하기 위해서 panic("pwrite")

`int pfileread(struct file*, char*, int, int)`: fileread()와 코드가 동일하며 readi 호출 시 offset을 f->off가 아닌 인자로 받은 off 사용. 오류가 났을 때 구분하기 위해서 panic("pread")

`int pwritei(struct inode*, char*, uint, uint)`: writei()와 같은 역할이지만 buffer cache를 읽어 log에 작성하기 전에 inode를 업데이트 할 수 있도록 코드 순서 변경

# The Milestone 3
- sync system call 구현

## tips
- struct log
- `void begin_op(void)` transaction의 시작 부분
- `void end_op(void)` begin_op와 함께 짝을 이뤄 transaction의 마지막 부분이 됨. 이 둘 사이의 진행하는 작업들이 하나의 transaction으로 commit 호출 전까지는 buffer cache에 존재

## system call
`int sync(void)` 호출 시 buffer cache의 내용을 disk에 기록. 일종의 동기화 작업

`int get_log_num(void)`: log.lh.n 반환

## Create functions
> `log.c`에 구현

`int bufferwrite(struct file*, char*, int)`: write system call에서 반환 값으로 쓰인 write() 함수 대신 사용하고자 함. buffer cache에 기록만 하며, log space(or buffer cache)가 가득 찰 경우 내부에서 disk에 작성할 수 있음. filewrite, begin_op, end_op를 참고하여 구현
![image](/uploads/5296abac2b2c703efd0687ebd99726e7/image.png)

`int sync(void)` 호출 시 end_op에서 do_commit == 1 일 때 수행하는 작업을 log.outstanding > 0 조건 하에 그대로 수행하도록 구현. return -1/1

`int get_log_num(void)` return log.lh.n

# Caution and Result
![image](/uploads/d3fef156e8130fb570f87ec79ef7baad/image.png)
- 초반에 bmap과 iturn을 구현할 때 기존의 indirect와 비슷한 논리로 코드를 작성했는데 여기서 doubly, triple indirect로 범위가 넓어진 것에 대해 고려하지 못했다
- 즉, 단계가 깊어짐에 따라 내부로 더 들어가는 작업이 필요한데 그저 숫자만 바뀐 indirect와 다를 바 없는 상태였다
- 이후 단계적으로 접근하도록 코드를 수정했으며 filesize가 구현한 범위를 커질 경우 종종 발생한다

## Milestone1 result
![image](/uploads/0b6205cb3278ecb84d7516b8ca898284/image.png)

## Milestone2 result
![image](/uploads/42fbecfd0b7f81f60aa3152cf0fb255a/image.png)
- thread 기반으로 test code

## Milestone3 result
![image](/uploads/0a3d16a59750f81794f8964b91e75c5b/image.png)
- 구현한 bufferwrite함수를 write system call의 반환 값으로 두었을 때 qemu가 제대로 열리지 않는 모습을 확인했다
- bufferwrite함수가 buffer cache에 기록만 하는 과정을 구현했다고 생각했는데 완전히 마무리 되지 않은 듯 하다

![image](/uploads/93c64a474637a1c261267e0f539ce7c1/image.png)
- 구현한 bufferwrite함수에서 buffer cache에 내용을 모두 기록하고 그 작업이 끝나면 sync( )를 호출하도록 코드를 변경하면 qumu가 열리고 원하는 결과가 나온다
- 그러나 이는 기존의 filewrite와 동일한 기능이기 때문에 제대로 구현하지 못한 것으로 보인다
- 혹시나 있을 오류 발생을 방지하기 위해 write system call의 반환 값은 원래의 filewrite함수로 두었다