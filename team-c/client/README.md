# CloudClub Chat Client

React와 Next.js로 구현된 실시간 채팅 클라이언트입니다.

## 기능

- 🔐 사용자 로그인/로그아웃
- 💬 실시간 메시징 (WebSocket)
- 🏠 다중 채팅방 지원
- 📱 반응형 디자인 (모바일 지원)
- 🔔 토스트 알림
- 🎨 모던한 UI/UX

## 기술 스택

- **Framework**: Next.js 15
- **Language**: TypeScript
- **Styling**: Tailwind CSS
- **Icons**: Heroicons
- **State Management**: React Hooks
- **Real-time**: WebSocket

## 시작하기

### 1. 의존성 설치

```bash
npm install
```

### 2. 개발 서버 실행

```bash
npm run dev
```

브라우저에서 [http://localhost:3000](http://localhost:3000)으로 접속합니다.

### 3. 빌드

```bash
npm run build
npm start
```

## 프로젝트 구조

```
client/
├── src/
│   ├── app/
│   │   └── page.tsx              # 메인 페이지
│   ├── components/
│   │   ├── chat/                 # 채팅 관련 컴포넌트
│   │   │   ├── ChatHeader.tsx
│   │   │   ├── LoginModal.tsx
│   │   │   ├── CreateRoomModal.tsx
│   │   │   ├── Sidebar.tsx
│   │   │   ├── MessageList.tsx
│   │   │   └── MessageInput.tsx
│   │   └── ui/                   # 공통 UI 컴포넌트
│   │       ├── Modal.tsx
│   │       └── Toast.tsx
│   ├── hooks/                    # 커스텀 훅
│   │   ├── useChat.ts
│   │   ├── useWebSocket.ts
│   │   └── useToast.ts
│   ├── types/                    # 타입 정의
│   │   └── index.ts
│   └── styles/
│       └── globals.css           # 전역 스타일
├── package.json
└── README.md
```

## 주요 컴포넌트

### useChat 훅

채팅 로직을 관리하는 메인 훅입니다.

- 사용자 상태 관리
- 채팅방 관리
- 메시지 전송/수신
- WebSocket 연결

### useWebSocket 훅

WebSocket 연결을 관리합니다.

- 자동 재연결
- 연결 상태 추적
- 메시지 송수신

### useToast 훅

토스트 알림을 관리합니다.

- 다양한 알림 타입 (success, error, warning, info)
- 자동 사라짐
- 진행 바 표시

## 서버 연결

이 클라이언트는 Go 서버와 연결됩니다:

- **WebSocket**: `ws://localhost:8080/ws`
- **REST API**: `http://localhost:8080/api/*`

## 개발 시 주의사항

1. 서버가 실행 중이어야 정상적으로 작동합니다
2. WebSocket 연결이 실패하면 자동으로 재연결을 시도합니다
3. 모든 컴포넌트는 TypeScript로 타입 안전성을 보장합니다

## 스크립트

- `npm run dev`: 개발 서버 실행
- `npm run build`: 프로덕션 빌드
- `npm start`: 프로덕션 서버 실행
- `npm run lint`: ESLint 검사
