---
layout: post
title: "Go 카카오톡 클론 — 클린 아키텍처 + WebSocket"
date: 2026-07-16
categories: [Go, 학습]
tags: [Go, interface, channel, goroutine, WebSocket, CleanArchitecture]
---

## 개요

1:1 채팅 서버를 완성한 뒤, 카카오톡 같은 실제 서비스를 목표로
클린 아키텍처 기반의 새 프로젝트를 시작했습니다.

---

## Phase 1 — Go 심화 개념

### 1. interface — 다형성

Go의 interface는 `implements` 키워드 없이
메서드만 맞으면 자동으로 구현됩니다.

```go
type Room interface {
    GetID()      string
    GetType()    string
    Join(userID uint)
    Leave(userID uint)
    GetMembers() []uint
    Send(msg Message)
}

// DirectRoom, GroupRoom 둘 다 Room으로 쓸 수 있음
var rooms []Room
rooms = append(rooms, &DirectRoom{})
rooms = append(rooms, &GroupRoom{})
```

### 2. channel — goroutine 간 통신

```go
// 메모리를 공유하지 말고, 통신으로 공유하라
hub.Broadcast <- &BroadcastMsg{RoomID: roomID, Message: msg}
```

### 3. select — 여러 channel 동시 대기

```go
for {
    select {
    case client := <-h.Register:
        // 새 연결 처리
    case client := <-h.Unregister:
        // 연결 종료 처리
    case msg := <-h.Broadcast:
        // 메시지 브로드캐스트
    }
}
```

### 4. 커스텀 에러 타입

```go
type AppError struct {
    Code    int
    Message string
    Err     error
}

// 에러 종류에 따라 HTTP 상태코드 자동 결정
func ErrNotFound(msg string) *AppError {
    return &AppError{Code: 404, Message: msg}
}
```

---

## Phase 2 — 클린 아키텍처

### 레이어 구조

```
Handler   → 요청/응답만 담당
Service   → 비즈니스 로직
Repository → DB 접근만 담당
```

### 핵심 — 의존성 주입

```go
// Repository → Service → Handler 순서로 생성
userRepo := repository.NewUserRepository(db.DB)
userSvc  := service.NewUserService(userRepo)
authHandler := handler.NewAuthHandler(userSvc)
```

Handler는 DB를 모릅니다.
Service는 Repository interface만 압니다.
Repository만 GORM을 씁니다.

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

나중에 DB를 바꿔도 Repository 구현체만 교체하면 됩니다.

---

## WebSocket — Hub 패턴

기존 채팅 서버와의 차이점입니다.

| | 기존 | 카카오 클론 |
|---|---|---|
| 연결 관리 | 전역 Map | Hub struct |
| 동시성 | sync.Mutex | channel |
| 지원 | 1:1 | 1:1 + 그룹 |

```go
// ReadPump · WritePump 분리
// 수신과 송신을 독립적인 goroutine으로 처리
go client.WritePump()
go client.ReadPump()
```

---

## 구현된 API 목록

```
POST   /auth/register
POST   /auth/login
POST   /auth/google
GET    /users/search?q=키워드
PATCH  /users/profile
GET    /chats
POST   /chats/direct
POST   /chats/group
GET    /chats/:roomId/messages
PATCH  /chats/:roomId/read
GET    /friends
POST   /friends/:id
PATCH  /friends/:id/accept
DELETE /friends/:id
GET    /ws  (WebSocket)
```

---

## 다음 — React UI 제작

- 카카오톡 스타일 UI
- 친구 목록 · 채팅방 목록
- 실시간 메시지 (WebSocket 연동)
- 그룹 채팅 생성

## 레포지토리

[kakao-clone](https://github.com/kimtaesung98/kakao-clone)
