# grpc-chat-app with Kafka

## 🏗️ 시스템 아키텍처

### 전체 구조도

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   gRPC Client   │    │   gRPC Client   │    │   gRPC Client   │
│     (Alice)     │    │      (Bob)      │    │    (Charlie)    │
└─────────┬───────┘    └─────────┬───────┘    └─────────┬───────┘
          │                      │                      │
          │ gRPC CreateStream    │ gRPC BroadcastMessage│ gRPC CreateStream
          │ & BroadcastMessage   │                      │ & BroadcastMessage
          ▼                      ▼                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                    gRPC Chat Server                             │
│                    (Message Gateway)                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │  Connection     │  │  Connection     │  │  Connection     │ │
│  │   Pool          │  │   Pool          │  │   Pool          │ │
│  │                 │  │                 │  │                 │ │
│  │ - Kafka Producer│  │ - User Sessions │  │ - Message       │ │
│  │ - User Sessions │  │ - gRPC Streams  │  │   Broadcasting  │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└─────────────────┬───────────────────────────────────┬─────────┘
                  │                                   │
                  │ Publish Messages                  │ Receive Processed
                  │ & User Events                     │ Messages
                  ▼                                   ▲
┌─────────────────────────────────────────────────────────────────┐
│                       Kafka Cluster                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   kafka-01      │  │   kafka-02      │  │   kafka-03      │ │
│  │   (Broker 1)    │  │   (Broker 2)    │  │   (Broker 3)    │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                 │
│  Topics:                                                        │
│  - chatting (2 partitions, 2 replicas)                        │
│  - user-connections (3 partitions, 2 replicas)                │
└─────────────────┬───────────────────────────────────┬─────────┘
                  │                                   │
                  │ Consume Messages                  │ Consume User
                  │ & Process                         │ Events
                  ▼                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Message Processor                             │
│                   (Kafka Consumer)                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ Chat Message    │  │ User Connection │  │ Server Registry │ │
│  │ Handler         │  │ Handler         │  │ & Cleanup       │ │
│  │                 │  │                 │  │                 │ │
│  │ - Parse Messages│  │ - Track Users   │  │ - Monitor       │ │
│  │ - Route to      │  │ - Update Server │  │   Servers       │ │
│  │   Servers       │  │   Registry      │  │ - Health Check  │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 주요 컴포넌트

#### 1. gRPC Chat Server (Message Gateway)

- **역할**: 클라이언트 연결 관리 및 메시지 게이트웨이
- **기능**:
  - 클라이언트 gRPC 연결 관리
  - 메시지를 Kafka로 발행 (Producer 역할)
  - 사용자 연결/해제 이벤트를 Kafka로 전송
  - 다중 서버 인스턴스 지원 (수평 확장 가능)

#### 2. Kafka Cluster

- **역할**: 분산 메시지 브로커
- **구성**: 3개 브로커, Zookeeper 포함
- **토픽**:
  - `chatting`: 채팅 메시지 (2 파티션, 2 복제본)
  - `user-connections`: 사용자 연결 이벤트 (3 파티션, 2 복제본)

#### 3. Message Processor (Kafka Consumer)

- **역할**: 메시지 처리 및 라우팅
- **기능**:
  - Kafka에서 메시지 소비
  - 서버 레지스트리 관리
  - 메시지 라우팅 및 분산 처리
  - 비활성 서버 정리

## 🚀 시작하기

### 사전 준비사항

- Go 1.19+
- Docker & Docker Compose
- Make (선택사항, 편의를 위해)

### 설치 및 설정

1. **의존성 설치**

```bash
go mod tidy
```

2. **Kafka 클러스터 시작**

```bash
make kafka-up
# 또는
docker-compose up -d
```

3. **Protobuf 코드 생성** (필요한 경우)

```bash
make proto
# 또는
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    proto/chat.proto
```

### 실행 방법

#### 방법 1: Make 사용 (권장)

```bash
# 1. Kafka 클러스터 시작
make kafka-up

# 2. gRPC 서버 시작 (터미널 1)
make server

# 3. Message Processor 시작 (터미널 2)
make consumer

# 4. 클라이언트 실행 (터미널 3, 4, ...)
make client USER=alice
make client USER=bob
```

#### 방법 2: 직접 실행

```bash
# 1. Kafka 시작
docker-compose up -d

# 2. gRPC 서버 실행
go run main.go

# 3. Consumer 실행 (새 터미널)
go run consumer.go consumer

# 4. 클라이언트 실행 (새 터미널들)
go run client.go alice
go run client.go bob
```

## 📊 모니터링

### Kafka UI

- **URL**: http://localhost:8080
- **기능**: 토픽, 메시지, 컨슈머 그룹 모니터링

### 로그 확인

- **서버 로그**: 연결 관리, Kafka 발행 상태
- **Consumer 로그**: 메시지 처리, 서버 레지스트리 상태
- **클라이언트 로그**: 연결 상태, 메시지 송수신

## 🔧 상세 기능 설명

### 1. 메시지 흐름

#### 메시지 전송 과정

1. **클라이언트**: `BroadcastMessage` gRPC 호출
2. **gRPC 서버**: 메시지를 JSON으로 직렬화하여 Kafka `chatting` 토픽에 발행
3. **Message Processor**: Kafka에서 메시지 소비
4. **Message Processor**: 연결된 모든 서버로 메시지 라우팅 (현재는 로그로 시뮬레이션)
5. **gRPC 서버**: 로컬 연결된 클라이언트들에게 메시지 브로드캐스트

#### 사용자 연결 관리

1. **클라이언트 연결**: `CreateStream` gRPC 호출
2. **gRPC 서버**: 연결 정보를 `user-connections` 토픽에 발행
3. **Message Processor**: 사용자 연결 이벤트 처리 및 서버 레지스트리 업데이트
4. **연결 해제**: 연결 종료 시 해제 이벤트 발행

### 2. 핵심 데이터 구조

#### ChatMessage (Kafka 메시지)

```go
type ChatMessage struct {
    ID        string    `json:"id"`        // 메시지 고유 ID
    Content   string    `json:"content"`   // 메시지 내용
    Timestamp time.Time `json:"timestamp"` // 전송 시간
    UserID    string    `json:"user_id"`   // 발신자 ID
}
```

#### UserConnection (연결 이벤트)

```go
type UserConnection struct {
    UserID    string    `json:"user_id"`    // 사용자 ID
    ServerID  string    `json:"server_id"`  // 서버 ID
    Connected bool      `json:"connected"`  // 연결 상태
    Timestamp time.Time `json:"timestamp"`  // 이벤트 시간
}
```

### 3. Kafka 설정

#### Producer 설정

- **RequiredAcks**: `WaitForAll` (모든 복제본 확인)
- **Retry**: 최대 5회
- **Return Successes**: `true`

#### Consumer 설정

- **Consumer Group**: `chatting-processor-group`
- **Offset**: `OffsetNewest` (최신 메시지부터)
- **Rebalance Strategy**: `RoundRobin`

## 🏗️ 확장 고려사항

### 수평 확장

1. **gRPC 서버**: 여러 인스턴스 실행 가능 (로드 밸런서 필요)
2. **Message Processor**: Consumer Group으로 분산 처리
3. **Kafka**: 파티션 수 증가로 처리량 향상

### 추가 기능 구현 예정

- [ ] 실제 서버 간 gRPC 통신
- [ ] 메시지 지속성 (데이터베이스)
- [ ] 사용자 인증 및 권한
- [ ] 채팅방/채널 기능
- [ ] 메시지 암호화
- [ ] 웹 클라이언트 (WebSocket)

## 🧪 테스트

### 단위 테스트

```bash
make test
# 또는
go test ./...
```

### Postman으로 gRPC 테스트

이 애플리케이션은 **gRPC Reflection**이 활성화되어 있어 Postman에서 쉽게 테스트할 수 있습니다.

#### 빠른 시작

1. **서버 실행**:

   ```bash
   make kafka-up
   make server
   ```

2. **Postman 설정**:

   - 새 gRPC 요청 생성
   - 서버 URL: `localhost:8081`
   - "Use Server Reflection" 활성화
   - `chat.Broadcast` 서비스 확인

3. **테스트 예제**:
   ```json
   // BroadcastMessage 테스트
   {
     "id": "postman-user",
     "content": "Hello from Postman!",
     "timestamp": null
   }
   ```

📋 **상세한 Postman 테스트 가이드**: [postman-guide.md](./postman-guide.md)

### 통합 테스트

```bash
# 전체 시스템 테스트
make kafka-up
sleep 15
make server &
make consumer &
sleep 5
make client USER=testuser1 &
make client USER=testuser2 &
wait
```

## 🐛 문제 해결

### 일반적인 문제

#### Kafka 연결 실패

```bash
# Kafka 상태 확인
docker-compose ps
docker-compose logs kafka-01

# 재시작
make kafka-down
make kafka-up
```

#### 포트 충돌

- gRPC 서버: 8081 (기본값)
- Kafka UI: 8080
- Kafka 브로커: 9092, 9093, 9094

#### 메모리 부족

```bash
# Docker 메모리 할당 확인 및 증가
docker system info
```

### 로그 분석

#### 성공적인 메시지 흐름

```
# gRPC 서버 로그
User alice connected to server grpc-server-1234567890
Message sent to Kafka - Topic: chatting, Partition: 1, Offset: 5

# Consumer 로그
Received chat message: Topic=chatting, Partition=1, Offset=5
Processing chat message: ID=alice, Content=Hello World!, From=alice
Message broadcast completed for message ID: alice

# 클라이언트 로그
Connected to server as user: alice
Message sent: Hello World!
[15:04:05] alice: Hello World!
```

## 📚 참고 자료

- [Kafka를 이용한 chatting 프로그램 개발 - joinc.co.kr](https://joinc.co.kr/w/man/12/Kafka/chatting)
- [gRPC Documentation](https://grpc.io/docs/)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Sarama - Go Kafka Client](https://github.com/IBM/sarama)

## 📄 라이선스

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
