# File Descriptor와 I/O 멀티플렉싱

## 문제의 시작: 서버가 여러 클라이언트를 어떻게 처리할까?

현실의 서버는 수천, 수만 명의 클라이언트를 동시에 처리해야 한다. 어떻게 이게 가능할까?

## File Descriptor란?

### 리눅스의 철학: "모든 것은 파일이다"

리눅스/유닉스 시스템에서는 **모든 것을 파일처럼 다룬다**는 철학이 있다:

- 일반 파일: `/home/user/document.txt`
- 디렉토리: `/home/user/`
- 장치: `/dev/sda` (하드디스크)
- **소켓**: 네트워크 연결도 파일처럼!

### File Descriptor(FD)는 뭘까?

**File Descriptor는 열린 파일을 가리키는 정수 번호**다. 운영체제가 관리하는 파일 테이블의 인덱스라고 생각하면 된다.

```
프로세스
├─ FD 0: 표준 입력 (stdin)
├─ FD 1: 표준 출력 (stdout)
├─ FD 2: 표준 에러 (stderr)
├─ FD 3: /home/user/file.txt
├─ FD 4: 소켓 (클라이언트 A와의 연결)
├─ FD 5: 소켓 (클라이언트 B와의 연결)
└─ FD 6: 소켓 (클라이언트 C와의 연결)
```

### 소켓도 File Descriptor

```java
// 서버 소켓 생성 (FD 3 받음)
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.bind(new InetSocketAddress(8080));

// 클라이언트 연결 수락 (FD 4 받음)
SocketChannel clientChannel = serverChannel.accept();

// 이제 read/write로 네트워크 통신!
ByteBuffer buffer = ByteBuffer.allocate(1024);
clientChannel.read(buffer);   // 소켓에서 데이터 읽기

ByteBuffer data = ByteBuffer.wrap("response".getBytes());
clientChannel.write(data);    // 소켓으로 데이터 쓰기
```

**핵심 깨달음**: 네트워크 통신이 파일 읽기/쓰기와 동일한 API를 사용한다는 것!

## Blocking I/O의 문제점

### 가장 단순한 서버

```java
ServerSocket serverSocket = new ServerSocket(8080);

while (true) {
    Socket clientSocket = serverSocket.accept();  // 🔴 여기서 대기

    InputStream input = clientSocket.getInputStream();
    byte[] buffer = new byte[1024];
    int bytesRead = input.read(buffer);  // 🔴 여기서도 대기

    byte[] response = process(buffer);

    OutputStream output = clientSocket.getOutputStream();
    output.write(response);

    clientSocket.close();
}
```

**문제점**: 한 번에 한 클라이언트만 처리 가능!

- 첫 번째 클라이언트가 느리면, 뒤의 모든 클라이언트가 기다려야 함
- CPU는 놀고 있는데 클라이언트는 못 받음

### 해결책 1: 멀티 프로세스/스레드

```java
while (true) {
    Socket clientSocket = serverSocket.accept();
    // 새 프로세스/스레드 생성
    Thread thread = new Thread(() -> handleClient(clientSocket));
    thread.start();
}
```

**문제점**:

- 클라이언트 1만 명 = 스레드 1만 개?
- 메모리 부족, 컨텍스트 스위칭 오버헤드
- C10K 문제 (connection 10,000 problem) 발생

## I/O 멀티플렉싱의 등장

하나의 프로세스(또는 스레드)가 여러 개의 I/O를 동시에 감시하고 처리하는 기술

**핵심 아이디어**: "여러 개의 FD를 한 스레드에서 모니터링하자!"

### select() - 1세대

```java
// 감시할 FD 집합 (Selector가 fd_set 역할)
Selector selector = Selector.open();

// FD_SET과 동일 - 소켓들을 감시 목록에 추가
socket1.configureBlocking(false);
SelectionKey key1 = socket1.register(selector, SelectionKey.OP_READ);

socket2.configureBlocking(false);
SelectionKey key2 = socket2.register(selector, SelectionKey.OP_READ);

socket3.configureBlocking(false);
SelectionKey key3 = socket3.register(selector, SelectionKey.OP_READ);

// select() 호출 - 🔴 대기
selector.select();

// 어떤 FD에 데이터가 왔는지 확인
Set<SelectionKey> selectedKeys = selector.selectedKeys();
for (SelectionKey key : selectedKeys) {
    if (key.isReadable()) {
        SocketChannel channel = (SocketChannel) key.channel();
        // socket1, socket2, socket3 중 하나에 데이터 도착!
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        channel.read(buffer);
    }
}
selectedKeys.clear();
```

**동작 방식**:

```
Thread 1
  └─ select() 호출
      ├─ FD 4 (클라이언트 A) 감시중...
      ├─ FD 5 (클라이언트 B) 감시중...
      └─ FD 6 (클라이언트 C) 감시중...

      ⏰ FD 5에 데이터 도착!

      → select() 리턴
      → FD 5 처리
      → 다시 select() 호출
```

**장점**: 하나의 스레드로 여러 연결 처리

**단점**:

- FD 개수 제한 (보통 1024개)
- 매번 FD 집합을 복사해야 함
- O(n) 복잡도 - 모든 FD를 순회하며 확인

### poll() - 1.5세대

```java
// pollfd 배열 대신 Selector 사용
Selector selector = Selector.open();

// fds[0].fd = socket1, events = POLLIN
socket1.configureBlocking(false);
socket1.register(selector, SelectionKey.OP_READ);

// fds[1].fd = socket2, events = POLLIN
socket2.configureBlocking(false);
socket2.register(selector, SelectionKey.OP_READ);

// fds[2].fd = socket3, events = POLLIN
socket3.configureBlocking(false);
socket3.register(selector, SelectionKey.OP_READ);

// poll() 호출 - 🔴 대기
selector.select();

// 어떤 FD에 이벤트가 발생했는지 확인
Set<SelectionKey> selectedKeys = selector.selectedKeys();
Iterator<SelectionKey> iter = selectedKeys.iterator();

while (iter.hasNext()) {
    SelectionKey key = iter.next();
    if (key.isReadable()) {
        // key.channel()에 데이터 도착!
        SocketChannel channel = (SocketChannel) key.channel();
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        channel.read(buffer);
    }
    iter.remove();
}
```

**select 대비 개선점**:

- FD 개수 제한 없음
- 비트마스크 대신 구조체 배열 사용 (더 명확함)

**여전한 문제**:

- O(n) 복잡도 유지
- 매번 전체 FD 배열을 커널에 복사

## epoll() - 2세대 (Linux 전용)

**혁명적 개선**: 상태를 커널에 유지!

```java
// 1. epoll 인스턴스 생성 (Linux에서는 내부적으로 epoll 사용)
Selector selector = Selector.open();

// 2. 감시할 FD 등록 (한 번만!)
socket1.configureBlocking(false);
SelectionKey key1 = socket1.register(selector, SelectionKey.OP_READ);

socket2.configureBlocking(false);
SelectionKey key2 = socket2.register(selector, SelectionKey.OP_READ);

socket3.configureBlocking(false);
SelectionKey key3 = socket3.register(selector, SelectionKey.OP_READ);

// 3. 이벤트 대기
while (true) {
    int n = selector.select();  // epoll_wait와 동일

    // 4. 이벤트 발생한 FD만 리턴됨!
    Set<SelectionKey> selectedKeys = selector.selectedKeys();
    Iterator<SelectionKey> iter = selectedKeys.iterator();

    while (iter.hasNext()) {
        SelectionKey key = iter.next();
        SocketChannel channel = (SocketChannel) key.channel();

        if (key.isReadable()) {
            // channel 처리
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            channel.read(buffer);
            handle(buffer);
        }

        iter.remove();  // 처리한 키는 제거해야 함
    }
}
```

**왜 빠른가?**

```
select/poll:
  유저공간 → [FD 리스트 전체 복사] → 커널
  커널 → [모든 FD 순회 검사] → 유저공간

epoll:
  유저공간 → [등록: 한 번만] → 커널
  커널 → [이벤트 발생한 FD만] → 유저공간
```

**핵심 차이점**:

- **O(1) 복잡도**: 활성 연결 수에만 비례
- **커널이 상태 관리**: FD 목록을 매번 복사 안 함
- **Edge-triggered 지원**: 더 효율적인 이벤트 처리

### 성능 비교

```
연결 1,000개 중 10개 활성화:
- select/poll: 1,000개 모두 검사
- epoll: 10개만 처리

연결 10,000개:
- select: 불가능 (FD 한계)
- poll: 매우 느림
- epoll: 빠름
```

## io_uring - 3세대 (최신)

Linux 5.1부터 도입된 **비동기 I/O의 새로운 패러다임**.

### 기존 방식의 한계

```
유저공간 ↔ 커널공간 전환 (시스템 콜) = 비용 발생

read() 호출  → 커널 진입 → 커널 처리 → 유저 복귀
write() 호출 → 커널 진입 → 커널 처리 → 유저 복귀
```

### io_uring의 혁신

**공유 메모리 링 버퍼**로 유저-커널 통신:

```
유저공간                     커널공간
  [제출 큐(SQ)] ←─────────→ [처리]
     ↓                        ↓
  [완료 큐(CQ)] ←─────────→ [결과]
```

**작동 방식**:

1. 유저: SQ에 작업 여러 개 한꺼번에 제출
2. 커널: 백그라운드에서 처리
3. 유저: CQ에서 완료된 작업 확인

**장점**:

- 시스템 콜 최소화 (배치 처리)
- 진정한 비동기 I/O
- 더 높은 처리량

**단점**:

- 비교적 최신 기술 (Linux 5.1+)
- 복잡한 API
- 아직 생태계가 성숙 중

## 기술 진화의 스토리

```
문제: 한 번에 한 클라이언트만 처리
  ↓
해결책 1: 멀티 프로세스/스레드
  문제: 리소스 낭비, C10K 문제
  ↓
해결책 2: select()
  "여러 FD를 한 스레드가 감시"
  문제: FD 개수 제한, O(n) 복잡도
  ↓
해결책 3: poll()
  "FD 제한 제거"
  문제: 여전히 O(n), 매번 전체 복사
  ↓
해결책 4: epoll()
  "커널이 상태 관리, O(1) 복잡도"
  문제: 여전히 시스템 콜 오버헤드
  ↓
해결책 5: io_uring
  "공유 메모리로 배치 처리"
```

## 정리

고성능 서버의 핵심은 **"어떻게 기다리지 않고 일할 것인가?"**다.

- **Blocking I/O**: 한 명씩 처리하며 기다림
- **멀티스레딩**: 여러 명이서 각자 기다림
- **I/O 멀티플렉싱**: 한 명이 여러 개를 동시에 지켜봄
- **io_uring**: 일을 맡겨두고 나중에 결과만 확인

### 참고

Claude
