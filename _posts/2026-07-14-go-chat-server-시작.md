---
layout: post
title: "Go로 1:1 채팅 서버 만들기 — 시작"
date: 2026-07-14
categories: [Go, 학습]
tags: [Go, Gin, PostgreSQL, WebSocket, JWT]
---

## 학습 목적

Node.js로 완성한 1:1 채팅 서버를 Go 언어로 재구현했습니다.
같은 DFD 설계를 기반으로, Go의 방식과 Node.js의 방식을 비교하며 학습했습니다.

## 기술 스택

| | Node.js | Go |
|---|---|---|
| Framework | Fastify | Gin |
| ORM | Prisma | GORM |
| 실시간 | socket.io | gorilla/websocket |
| 인증 | JWT + bcrypt | golang-jwt + bcrypt |

## 배운 Go 핵심 개념

### 1. 에러 처리
Node.js는 throw/catch, Go는 에러를 값으로 반환합니다.

```go
// Go 에러 처리
data, err := someFunc()
if err != nil {
    // 에러 처리
}
```

### 2. goroutine
WebSocket 연결마다 goroutine을 하나씩 만들어 독립적으로 처리합니다.

```go
go handleConnection(conn) // 경량 스레드 생성
```

### 3. struct + 태그
Prisma 스키마 대신 struct로 DB 모델을 정의합니다.

```go
type User struct {
    ID    uint   `gorm:"primaryKey" json:"id"`
    Email string `gorm:"uniqueIndex" json:"email"`
}
```

## 전체 진행 단계

- Step 0: Go 환경 + Gin + PostgreSQL
- Step 1: 회원가입 · 로그인 · JWT
- Step 2: 메시지 저장 · 조회 · 읽음처리
- Step 3: WebSocket 실시간 메시지
- Step 4: 온라인 상태 관리
- Step 5: DB CRUD 실습
- Step 6~9: Google OAuth 연동
- Step 10~: Web UI + 배포 예정

## 다음 포스팅

Docker Compose로 전체 서비스 묶기