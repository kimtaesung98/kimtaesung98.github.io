---
layout: post
title: "Go 카카오 클론 — 실시간 채팅 완성 (WebSocket · goroutine · channel)"
date: 2026-07-20
categories: [Go, 학습]
tags: [Go, WebSocket, goroutine, channel, React, 동시성]
---

## 개요

카카오톡 클론 프로젝트에서 실시간 채팅을 완성했습니다.
Go의 goroutine · channel을 활용한 WebSocket Hub 패턴과
동시성 처리 방식을 정리합니다.

---

## 핵심 개념 — 동시성이란

### 물리적 현실

CPU 코어 4개 = 진짜 동시에 4가지만 처리 가능.
하지만 동시 접속자는 수천 명.

**해결책 = 대기 시간 활용**

```
유저A 회원가입 요청
  → bcrypt 해싱 (0.1초)
  → DB 저장 요청
  → DB 응답 대기 중... (0.05초) ← CPU가 노는 시간
  → 이 시간에 유저B 요청 처리
```

### Node.js vs Go

```
Node.js — 직원 1명이 빠르게 번갈아 처리
          (동시처럼 보이지만 사실 순차)

Go      — 요청마다 goroutine(직원) 1명씩 배정
          (진짜 동시 처리)
```

goroutine은 메모리 2KB로 매우 가볍습니다.
수만 개를 만들어도 문제없습니다.

---

## FPS 게임으로 이해하는 동시성

채팅 서버와 FPS 게임 서버는 같은 구조입니다.

| FPS | 채팅 서버 | 역할 |
|---|---|---|
| UDP/TCP 소켓 | WebSocket | 연결 유지 |
| 플레이어 스레드 | goroutine | 독립 처리 |
| 틱레이트 | 브로드캐스트 | 상태 동기화 |
| 게임 방 | 채팅 룸 | 그룹 관리 |
| 서버 간 통신 | Redis Pub/Sub | 수평 확장 |

**FPS 총 발사 흐름:**
```
발사 이벤트 → 서버 전송 → goroutine 수신
→ 피격 판정 → channel로 Hub 전달
→ 같은 방 플레이어 전체에게 브로드캐스트
```

**채팅 메시지 흐름:**
```
메시지 입력 → 서버 전송 → goroutine 수신
→ DB 저장 → channel로 Hub 전달
→ 같은 룸 유저 전체에게 브로드캐스트
```

완전히 같은 흐름입니다.

---

## WebSocket Hub 패턴

### 기존 채팅 서버와의 차이

| | 기존 채팅 서버 | 카카오 클론 |
|---|---|---|
| 연결 관리 | 전역 Map | Hub struct |
| 동시성 | sync.Mutex | channel |
| 지원 | 1:1 | 1:1 + 그룹 |

### Hub 구조

```go
type Hub struct {
    rooms      map[string]map[uint]*Client
    Register   chan *Client   // 새 연결
    Unregister chan *Client   // 연결 종료
    Broadcast  chan *BroadcastMsg // 메시지
    messageSvc *service.MessageService
}
```

### select — 여러 channel 동시 대기

```go
func (h *Hub) Run() {
    for {
        select {
        case client := <-h.Register:
            // 새 연결 등록
        case client := <-h.Unregister:
            // 연결 종료 처리
        case msg := <-h.Broadcast:
            // 룸 전체에 메시지 전달
        }
    }
}
```

`select`는 여러 channel을 동시에 기다립니다.
먼저 도착한 것부터 처리합니다.
Node.js의 `Promise.race()`와 가장 유사합니다.

### ReadPump · WritePump 분리

```go
// 연결마다 goroutine 2개 생성
go client.WritePump()  // 서버 → 클라이언트
go client.ReadPump()   // 클라이언트 → 서버
```

수신과 송신을 독립적인 goroutine으로 처리합니다.
서로 블로킹하지 않습니다.

### channel = 우체통

```go
// 유저A goroutine이 메시지를 channel에 넣음
c.Hub.Broadcast <- &BroadcastMsg{...}

// Hub goroutine이 channel에서 꺼내서 전달
case msg := <-h.Broadcast:
    // 룸 전체에 전달
```

channel은 goroutine 간 안전한 데이터 전달 통로입니다.
Mutex 없이도 race condition이 없습니다.

---

## 클린 아키텍처 — 레이어 분리

```
Handler   → 요청/응답만 담당
Service   → 비즈니스 로직
Repository → DB 접근만 담당 (interface)
```

### 의존성 주입

```go
// 아래에서 위로 생성
userRepo    := repository.NewUserRepository(db.DB)
userSvc     := service.NewUserService(userRepo)
authHandler := handler.NewAuthHandler(userSvc)
```

Handler는 DB를 모릅니다.
Service는 Repository interface만 압니다.
나중에 DB를 바꿔도 Repository만 수정하면 됩니다.

### Repository interface

```go
type UserRepository interface {
    Create(user *model.User) error
    FindByID(id uint) (*model.User, error)
    FindByEmail(email string) (*model.User, error)
    Update(user *model.User) error
    Search(keyword string) ([]model.User, error)
}
```

interface 덕분에 테스트할 때
가짜(Mock) 구현체로 교체할 수 있습니다.

---

## 서버 확장 — 다수 유저 처리

### 현재 구조 (1대)

```
Go 서버 1대
└── Hub (메모리 map)
    └── 모든 유저 연결
```

### Redis 추가 (수만 명)

```
서버1 유저A → 메시지 → Redis 발행
                          ↓
서버2        ← Redis 구독 ← 유저B에게 전달
```

Redis가 서버들 사이의 우체국 역할을 합니다.

### 진행 방향

```
현재: 메모리 Hub (1대)
Phase 3: Redis Pub/Sub 추가
그 이후: Kafka + 마이크로서비스
```

---

## 구현된 전체 API

```
# 인증
POST /auth/register
POST /auth/login
POST /auth/google

# 유저
GET  /users/search?q=키워드
PATCH /users/profile

# 채팅
GET  /chats
POST /chats/direct
POST /chats/group
GET  /chats/:roomId/messages
PATCH /chats/:roomId/read

# 친구
GET    /friends
POST   /friends/:id
PATCH  /friends/:id/accept
DELETE /friends/:id

# WebSocket
GET /ws?token=JWT
```

---

## 동작 확인

```
✓ 회원가입 · 로그인
✓ 유저 검색
✓ 1:1 채팅방 생성
✓ 실시간 메시지 (WebSocket)
✓ 두 유저 간 실시간 통신
```

---

## 다음 목표

- 친구 기능 · 그룹 채팅 UI 완성
- Phase 3 — Redis + 클라우드 배포
- CI/CD — GitHub Actions 자동 배포

## 레포지토리

- [kakao-clone (Go 서버)](https://github.com/kimtaesung98/kakao-clone)
- [kakao-clone-client (React)](https://github.com/kimtaesung98/kakao-clone-client)
