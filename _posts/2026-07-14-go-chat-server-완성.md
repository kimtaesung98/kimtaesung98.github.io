---
layout: post
title: "Go 1:1 채팅 서버 완성 — Node.js 개발자의 Go 학습 기록"
date: 2026-07-14
categories: [Go, 학습]
tags: [Go, Gin, GORM, WebSocket, JWT, Docker, Google OAuth]
---

## 개요

Node.js로 완성한 1:1 채팅 서버를 Go로 재구현했습니다.

## 전체 기술 스택

| | Node.js | Go |
|---|---|---|
| Framework | Fastify | Gin |
| ORM | Prisma | GORM |
| 실시간 | socket.io | gorilla/websocket |
| 인증 | JWT + bcrypt | golang-jwt + bcrypt |
| 배포 | 수동 | Docker Compose |

## 구현 기능

- 회원가입 · 로그인 (이메일 + Google OAuth)
- 1:1 실시간 메시지 (WebSocket)
- 메시지 저장 · 조회 · 읽음처리
- 온라인 상태 실시간 반영
- 유저 검색 + 채팅 시작

## 배운 Go 핵심 개념

### 1. 에러를 값으로 처리
```go
data, err := someFunc()
if err != nil {
    return err
}
```

### 2. goroutine — 경량 스레드
```go
// WebSocket 연결마다 독립 실행
go handleConnection(conn)
```

### 3. sync.Mutex — 동시성 보호
```go
mu.Lock()
clients[userID] = conn
mu.Unlock()
```

### 4. struct 태그
```go
type User struct {
    ID    uint   `gorm:"primaryKey" json:"id"`
    Email string `gorm:"uniqueIndex" json:"email"`
}
```

### 5. defer — 함수 종료 시 자동 실행
```go
defer func() {
    delete(clients, userID)
    conn.Close()
}()
```

## Docker Compose 구조

```yaml
services:
  db:     # PostgreSQL
  server: # Go 서버
  client: # nginx 정적 파일
```

```bash
docker compose up --build  # 한 번에 전부 실행
```

## 느낀 점

- Go는 에러 처리가 명시적이라 처음엔 귀찮지만
  코드만 봐도 어디서 에러가 날 수 있는지 보임
- goroutine은 Node.js 이벤트 루프보다
  직관적으로 동시성을 다룰 수 있음
- 정적 타입은 IDE 지원이 강력해서
  오히려 개발 속도가 빨라짐

## 레포지토리

- [chat-server-go](https://github.com/kimtaesung98/chat-server-go)
- [chat-client](https://github.com/kimtaesung98/chat-client)