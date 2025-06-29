# 주안의소리 - API 명세서

**API Specification Document v1.0**  
**프로젝트명**: 주안의소리 (Voice of Juan)  
**문서 작성일**: 2025년 6월 29일  
**담당자**: Backend Team  
**승인자**: Tech Lead  
**관련 문서**: PRD v1.0, FRD v1.1, System Architecture v1.0

---

## 1. 문서 개요

### 1.1 목적
본 문서는 주안의소리 시각장애인용 오디오북 앱의 모든 API 엔드포인트와 데이터 스키마를 정의합니다.

### 1.2 API 아키텍처
- **Backend**: Firebase (Firestore, Cloud Functions, Storage)
- **Authentication**: Firebase Auth (Google OAuth)
- **Protocol**: HTTPS/REST + WebSocket (실시간 기능)
- **Data Format**: JSON
- **Base URL**: `https://asia-northeast1-juan-voice.cloudfunctions.net/api`

### 1.3 인증 방식
모든 API는 Firebase Auth JWT 토큰을 사용합니다.

```javascript
// Request Header
Authorization: Bearer <firebase_jwt_token>
Content-Type: application/json
```

### 1.4 공통 응답 구조
```json
{
  "success": true,
  "data": { ... },
  "message": "Success message",
  "timestamp": "2025-06-29T10:30:00.000Z"
}

// Error Response
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Error description",
    "details": { ... }
  },
  "timestamp": "2025-06-29T10:30:00.000Z"
}
```

---

## 2. 인증 및 사용자 관리 API

### 2.1 사용자 인증

#### 2.1.1 Google OAuth 로그인
**Firebase Auth SDK 사용 - 클라이언트에서 직접 처리**

```javascript
// Firebase SDK
import { signInWithPopup, GoogleAuthProvider } from 'firebase/auth';

const provider = new GoogleAuthProvider();
const result = await signInWithPopup(auth, provider);
const idToken = await result.user.getIdToken();
```

#### 2.1.2 사용자 프로필 조회
```http
GET /api/v1/auth/profile
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "uid": "string",
    "email": "user@example.com",
    "displayName": "홍길동",
    "userType": "member|external",
    "memberNumber": "12345",
    "isApproved": true,
    "isActive": true,
    "createdAt": "2025-06-29T10:30:00.000Z",
    "lastLoginAt": "2025-06-29T10:30:00.000Z"
  }
}
```

### 2.2 성도번호 인증

#### 2.2.1 성도번호 검증 및 매핑
```http
POST /api/v1/auth/validate-member-number
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

**Request:**
```json
{
  "memberNumber": "12345"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "isValid": true,
    "isAvailable": true,
    "memberInfo": {
      "memberNumber": "12345",
      "name": "홍길동",
      "registrationDate": "2020-01-01"
    }
  }
}
```

**Error Responses:**
```json
// 이미 사용 중인 성도번호
{
  "success": false,
  "error": {
    "code": "MEMBER_NUMBER_ALREADY_MAPPED",
    "message": "이미 사용 중인 성도번호입니다.",
    "details": {
      "memberNumber": "12345",
      "mappedAt": "2025-06-20T10:30:00.000Z"
    }
  }
}

// 유효하지 않은 성도번호
{
  "success": false,
  "error": {
    "code": "INVALID_MEMBER_NUMBER",
    "message": "등록되지 않은 성도번호입니다.",
    "details": {
      "memberNumber": "99999"
    }
  }
}
```

#### 2.2.2 성도번호 매핑 확정
```http
POST /api/v1/auth/map-member-number
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

**Request:**
```json
{
  "memberNumber": "12345"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "userId": "firebase_uid",
    "memberNumber": "12345",
    "mappedAt": "2025-06-29T10:30:00.000Z",
    "userType": "member",
    "isApproved": true
  }
}
```

### 2.3 외부 사용자 승인 요청

#### 2.3.1 승인 요청 제출
```http
POST /api/v1/auth/request-approval
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

**Request:**
```json
{
  "name": "김외부",
  "phone": "010-1234-5678",
  "purpose": "시각장애인 지원을 위한 오디오북 이용"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "requestId": "req_12345",
    "status": "pending",
    "submittedAt": "2025-06-29T10:30:00.000Z",
    "estimatedReviewTime": "1-3 business days"
  }
}
```

#### 2.3.2 승인 상태 확인
```http
GET /api/v1/auth/approval-status
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "status": "pending|approved|rejected",
    "requestedAt": "2025-06-29T10:30:00.000Z",
    "reviewedAt": "2025-06-30T10:30:00.000Z",
    "reviewedBy": "admin_uid",
    "reason": "승인/거부 사유"
  }
}
```

---

## 3. 콘텐츠 관리 API

### 3.1 도서 목록 조회

#### 3.1.1 도서 목록 조회 (페이지네이션)
```http
GET /api/v1/books?page=1&limit=20&category=all&search=검색어
Authorization: Bearer <jwt_token>
```

**Query Parameters:**
- `page`: 페이지 번호 (기본값: 1)
- `limit`: 페이지당 항목 수 (기본값: 20, 최대: 100)
- `category`: 카테고리 필터 (all, fiction, non-fiction, religion, etc.)
- `search`: 검색어 (제목, 저자)
- `sort`: 정렬 기준 (createdAt, title, author, playCount)
- `order`: 정렬 순서 (asc, desc)

**Response:**
```json
{
  "success": true,
  "data": {
    "books": [
      {
        "bookId": "book_12345",
        "title": "오디오북 제목",
        "author": "저자명",
        "description": "도서 설명",
        "coverImageUrl": "https://storage.googleapis.com/covers/book_12345.jpg",
        "category": "fiction",
        "totalDuration": 3600,
        "totalChapters": 12,
        "playCount": 150,
        "averageRating": 4.5,
        "createdAt": "2025-06-29T10:30:00.000Z",
        "isActive": true
      }
    ],
    "pagination": {
      "currentPage": 1,
      "totalPages": 10,
      "totalItems": 200,
      "hasNextPage": true,
      "hasPrevPage": false
    }
  }
}
```

#### 3.1.2 도서 상세 정보 조회
```http
GET /api/v1/books/{bookId}
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "bookId": "book_12345",
    "title": "오디오북 제목",
    "author": "저자명",
    "description": "상세 설명",
    "coverImageUrl": "https://storage.googleapis.com/covers/book_12345.jpg",
    "category": "fiction",
    "totalDuration": 3600,
    "playCount": 150,
    "chapters": [
      {
        "chapterIndex": 1,
        "title": "챕터 1",
        "duration": 300,
        "streamingUrl": "https://storage.googleapis.com/audio/book_12345_ch1.m3u8"
      }
    ],
    "tags": ["소설", "추천도서"],
    "isbn": "978-1234567890",
    "publisher": "출판사명",
    "publishedDate": "2024-01-01",
    "createdAt": "2025-06-29T10:30:00.000Z"
  }
}
```

### 3.2 카테고리 관리

#### 3.2.1 카테고리 목록 조회
```http
GET /api/v1/categories
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "categories": [
      {
        "categoryId": "fiction",
        "name": "소설",
        "description": "소설 및 문학 작품",
        "bookCount": 45,
        "isActive": true
      },
      {
        "categoryId": "religion",
        "name": "종교",
        "description": "종교 및 영성 관련 도서",
        "bookCount": 23,
        "isActive": true
      }
    ]
  }
}
```

---

## 4. 스트리밍 및 재생 API

### 4.1 스트리밍 URL 생성

#### 4.1.1 보안 스트리밍 URL 요청
```http
POST /api/v1/streaming/generate-url
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

**Request:**
```json
{
  "bookId": "book_12345",
  "chapterIndex": 1,
  "quality": "medium"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "streamingUrl": "https://cdn.juan-voice.com/audio/book_12345_ch1_signed.m3u8?token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
    "expiresAt": "2025-06-29T11:30:00.000Z",
    "quality": "medium",
    "bitrate": 128,
    "duration": 300,
    "sessionId": "session_12345"
  }
}
```

#### 4.1.2 스트리밍 세션 갱신
```http
POST /api/v1/streaming/refresh-session
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

**Request:**
```json
{
  "sessionId": "session_12345"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "streamingUrl": "https://cdn.juan-voice.com/audio/book_12345_ch1_signed.m3u8?token=new_token...",
    "expiresAt": "2025-06-29T12:30:00.000Z",
    "sessionId": "session_12345"
  }
}
```

### 4.2 재생 로그

#### 4.2.1 재생 이벤트 로깅
```http
POST /api/v1/playback/log
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

**Request:**
```json
{
  "bookId": "book_12345",
  "chapterIndex": 1,
  "action": "start|pause|resume|complete|skip",
  "positionSeconds": 150,
  "sessionId": "session_12345",
  "deviceInfo": {
    "platform": "android",
    "version": "1.0.0",
    "model": "Samsung Galaxy S21"
  }
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "logId": "log_12345",
    "timestamp": "2025-06-29T10:30:00.000Z"
  }
}
```

---

## 5. 북마크 및 이어듣기 API

### 5.1 북마크 관리

#### 5.1.1 북마크 저장/업데이트
```http
PUT /api/v1/bookmarks
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

**Request:**
```json
{
  "bookId": "book_12345",
  "chapterIndex": 1,
  "positionSeconds": 150,
  "note": "이 부분부터 다시 들어야 함"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "bookmarkId": "bookmark_12345",
    "bookId": "book_12345",
    "chapterIndex": 1,
    "positionSeconds": 150,
    "note": "이 부분부터 다시 들어야 함",
    "lastUpdated": "2025-06-29T10:30:00.000Z"
  }
}
```

#### 5.1.2 사용자 북마크 목록 조회
```http
GET /api/v1/bookmarks?page=1&limit=20
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "bookmarks": [
      {
        "bookmarkId": "bookmark_12345",
        "bookId": "book_12345",
        "bookTitle": "오디오북 제목",
        "bookCoverUrl": "https://storage.googleapis.com/covers/book_12345.jpg",
        "chapterIndex": 1,
        "chapterTitle": "챕터 1",
        "positionSeconds": 150,
        "totalDuration": 300,
        "progressPercentage": 50,
        "note": "이 부분부터 다시 들어야 함",
        "lastUpdated": "2025-06-29T10:30:00.000Z"
      }
    ],
    "pagination": {
      "currentPage": 1,
      "totalPages": 5,
      "totalItems": 100
    }
  }
}
```

#### 5.1.3 북마크 삭제
```http
DELETE /api/v1/bookmarks/{bookmarkId}
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "bookmarkId": "bookmark_12345",
    "deletedAt": "2025-06-29T10:30:00.000Z"
  }
}
```

### 5.2 이어듣기

#### 5.2.1 최근 재생 도서 조회
```http
GET /api/v1/continue-listening
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "recentBooks": [
      {
        "bookId": "book_12345",
        "title": "오디오북 제목",
        "author": "저자명",
        "coverImageUrl": "https://storage.googleapis.com/covers/book_12345.jpg",
        "lastPlayedChapter": 1,
        "lastPlayedPosition": 150,
        "totalDuration": 3600,
        "progressPercentage": 25,
        "lastPlayedAt": "2025-06-29T10:30:00.000Z"
      }
    ]
  }
}
```

---

## 6. 사용자 통계 API

### 6.1 개인 통계

#### 6.1.1 사용자 청취 통계 조회
```http
GET /api/v1/stats/user?period=week&startDate=2025-06-01&endDate=2025-06-29
Authorization: Bearer <jwt_token>
```

**Query Parameters:**
- `period`: day|week|month|year
- `startDate`: YYYY-MM-DD
- `endDate`: YYYY-MM-DD

**Response:**
```json
{
  "success": true,
  "data": {
    "totalListeningTime": 7200,
    "totalSessions": 25,
    "completedBooks": 3,
    "averageSessionDuration": 288,
    "favoriteCategory": "fiction",
    "dailyStats": [
      {
        "date": "2025-06-29",
        "listeningTime": 1800,
        "sessions": 3,
        "booksListened": 2
      }
    ],
    "categoryBreakdown": [
      {
        "category": "fiction",
        "listeningTime": 3600,
        "percentage": 50
      }
    ]
  }
}
```

#### 6.1.2 사용자 성취 조회
```http
GET /api/v1/stats/achievements
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "achievements": [
      {
        "achievementId": "first_book",
        "title": "첫 번째 완독",
        "description": "첫 번째 도서를 완주했습니다",
        "unlockedAt": "2025-06-25T10:30:00.000Z",
        "iconUrl": "https://storage.googleapis.com/achievements/first_book.png"
      }
    ],
    "progress": [
      {
        "achievementId": "reading_streak_7",
        "title": "7일 연속 청취",
        "currentCount": 5,
        "targetCount": 7,
        "progress": 71
      }
    ]
  }
}
```

---

## 7. 관리자 API

### 7.1 사용자 관리

#### 7.1.1 외부 사용자 승인 요청 목록
```http
GET /api/v1/admin/approval-requests?status=pending&page=1&limit=20
Authorization: Bearer <admin_jwt_token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "requests": [
      {
        "requestId": "req_12345",
        "userId": "user_12345",
        "email": "user@example.com",
        "displayName": "김외부",
        "approvalRequest": {
          "name": "김외부",
          "phone": "010-1234-5678",
          "purpose": "시각장애인 지원을 위한 오디오북 이용",
          "requestedAt": "2025-06-29T10:30:00.000Z"
        },
        "status": "pending",
        "userIP": "192.168.1.100",
        "userAgent": "Mozilla/5.0..."
      }
    ],
    "pagination": {
      "currentPage": 1,
      "totalPages": 5,
      "totalItems": 87
    }
  }
}
```

#### 7.1.2 외부 사용자 승인 처리
```http
POST /api/v1/admin/approve-user
Authorization: Bearer <admin_jwt_token>
Content-Type: application/json
```

**Request:**
```json
{
  "userId": "user_12345",
  "action": "approve|reject",
  "reason": "승인/거부 사유"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "userId": "user_12345",
    "action": "approve",
    "processedAt": "2025-06-29T10:30:00.000Z",
    "processedBy": "admin_12345",
    "reason": "승인 사유"
  }
}
```

#### 7.1.3 사용자 계정 정지/해제
```http
POST /api/v1/admin/suspend-user
Authorization: Bearer <admin_jwt_token>
Content-Type: application/json
```

**Request:**
```json
{
  "userId": "user_12345",
  "action": "suspend|activate",
  "reason": "정지/해제 사유",
  "duration": 7
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "userId": "user_12345",
    "action": "suspend",
    "suspendedAt": "2025-06-29T10:30:00.000Z",
    "suspendedBy": "admin_12345",
    "reason": "정지 사유",
    "expiresAt": "2025-07-06T10:30:00.000Z"
  }
}
```

### 7.2 성도번호 매핑 관리

#### 7.2.1 성도번호 매핑 현황 조회
```http
GET /api/v1/admin/member-mappings?page=1&limit=20&search=12345
Authorization: Bearer <admin_jwt_token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "mappings": [
      {
        "memberNumber": "12345",
        "userId": "user_12345",
        "userEmail": "user@example.com",
        "userName": "홍길동",
        "mappedAt": "2025-06-29T10:30:00.000Z",
        "isActive": true,
        "lastLoginAt": "2025-06-29T10:30:00.000Z",
        "lastLoginIP": "192.168.1.100"
      }
    ],
    "summary": {
      "totalMemberNumbers": 500,
      "mappedCount": 234,
      "unmappedCount": 266,
      "suspendedCount": 5
    }
  }
}
```

#### 7.2.2 성도번호 매핑 강제 해제
```http
POST /api/v1/admin/unmap-member-number
Authorization: Bearer <admin_jwt_token>
Content-Type: application/json
```

**Request:**
```json
{
  "memberNumber": "12345",
  "reason": "어뷰저 계정으로 인한 강제 해제"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "memberNumber": "12345",
    "previousUserId": "user_12345",
    "unmappedAt": "2025-06-29T10:30:00.000Z",
    "unmappedBy": "admin_12345",
    "reason": "어뷰저 계정으로 인한 강제 해제"
  }
}
```

### 7.3 콘텐츠 관리

#### 7.3.1 도서 업로드
```http
POST /api/v1/admin/books
Authorization: Bearer <admin_jwt_token>
Content-Type: multipart/form-data
```

**Request:**
```
title: 오디오북 제목
author: 저자명
description: 도서 설명
category: fiction
coverImage: [file]
audioFiles: [file array]
chapters: [JSON array of chapter metadata]
```

**Response:**
```json
{
  "success": true,
  "data": {
    "bookId": "book_12345",
    "uploadStatus": "processing",
    "estimatedProcessingTime": "10-15 minutes",
    "chapters": [
      {
        "chapterIndex": 1,
        "uploadStatus": "processing",
        "fileSize": 15728640
      }
    ]
  }
}
```

#### 7.3.2 도서 활성화/비활성화
```http
PUT /api/v1/admin/books/{bookId}/status
Authorization: Bearer <admin_jwt_token>
Content-Type: application/json
```

**Request:**
```json
{
  "isActive": true,
  "reason": "콘텐츠 검토 완료"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "bookId": "book_12345",
    "isActive": true,
    "updatedAt": "2025-06-29T10:30:00.000Z",
    "updatedBy": "admin_12345"
  }
}
```

### 7.4 실시간 모니터링

#### 7.4.1 실시간 사용자 현황
```http
GET /api/v1/admin/real-time/users
Authorization: Bearer <admin_jwt_token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "activeUsers": 156,
    "onlineUsers": [
      {
        "userId": "user_12345",
        "email": "user@example.com",
        "displayName": "홍길동",
        "currentBook": "book_12345",
        "currentChapter": 1,
        "sessionStartTime": "2025-06-29T10:00:00.000Z",
        "ipAddress": "192.168.1.100",
        "location": "Seoul, South Korea",
        "deviceInfo": {
          "platform": "android",
          "model": "Samsung Galaxy S21"
        }
      }
    ],
    "statistics": {
      "totalSessions": 187,
      "averageSessionDuration": 1800,
      "peakConcurrentUsers": 234,
      "newUsersToday": 12
    }
  }
}
```

#### 7.4.2 어뷰저 의심 계정 조회
```http
GET /api/v1/admin/abuse-reports?status=pending&page=1&limit=20
Authorization: Bearer <admin_jwt_token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "reports": [
      {
        "reportId": "report_12345",
        "reportedUserId": "user_12345",
        "reportType": "invalid_member_number",
        "description": "성도번호 중복 사용 시도",
        "severity": "high",
        "createdAt": "2025-06-29T10:30:00.000Z",
        "reportedBy": "system",
        "userInfo": {
          "email": "suspicious@example.com",
          "memberNumber": "12345",
          "ipHistory": ["192.168.1.100", "10.0.0.1"],
          "registrationIP": "192.168.1.100"
        },
        "evidences": [
          {
            "type": "duplicate_mapping_attempt",
            "timestamp": "2025-06-29T10:25:00.000Z",
            "details": {
              "memberNumber": "12345",
              "existingUserId": "user_67890"
            }
          }
        ]
      }
    ]
  }
}
```

---

## 8. 통계 및 분석 API

### 8.1 서비스 통계

#### 8.1.1 대시보드 통계 조회
```http
GET /api/v1/admin/statistics/dashboard?period=week
Authorization: Bearer <admin_jwt_token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "summary": {
      "totalUsers": 1234,
      "activeUsers": 567,
      "totalBooks": 89,
      "totalPlayTime": 123456789,
      "newUsersThisWeek": 45
    },
    "userGrowth": [
      {
        "date": "2025-06-29",
        "newUsers": 12,
        "totalUsers": 1234,
        "activeUsers": 567
      }
    ],
    "contentUsage": [
      {
        "bookId": "book_12345",
        "title": "인기 도서",
        "playCount": 234,
        "totalPlayTime": 45678,
        "completionRate": 0.78
      }
    ],
    "categoryPopularity": [
      {
        "category": "fiction",
        "playCount": 1234,
        "percentage": 45.6
      }
    ]
  }
}
```

#### 8.1.2 상세 분석 리포트
```http
GET /api/v1/admin/statistics/detailed?startDate=2025-06-01&endDate=2025-06-29&metrics=users,content,performance
Authorization: Bearer <admin_jwt_token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "reportPeriod": {
      "startDate": "2025-06-01",
      "endDate": "2025-06-29"
    },
    "userMetrics": {
      "newRegistrations": 123,
      "approvalRate": 0.89,
      "retentionRate": {
        "day1": 0.85,
        "day7": 0.65,
        "day30": 0.45
      },
      "churnRate": 0.05
    },
    "contentMetrics": {
      "totalPlayTime": 987654321,
      "averageSessionDuration": 1800,
      "completionRates": {
        "overall": 0.67,
        "byCategory": {
          "fiction": 0.72,
          "religion": 0.81
        }
      }
    },
    "performanceMetrics": {
      "averageLoadTime": 2.3,
      "errorRate": 0.02,
      "streamingQuality": {
        "bufferingEvents": 234,
        "qualityDrops": 45
      }
    }
  }
}
```

---

## 9. WebSocket API (실시간 기능)

### 9.1 실시간 연결

#### 9.1.1 WebSocket 연결
```javascript
// WebSocket 연결
const ws = new WebSocket('wss://asia-northeast1-juan-voice.cloudfunctions.net/api/ws');

// 인증
ws.send(JSON.stringify({
  type: 'auth',
  token: firebase_jwt_token
}));
```

#### 9.1.2 실시간 이벤트 구독
```javascript
// 관리자 실시간 모니터링 구독
ws.send(JSON.stringify({
  type: 'subscribe',
  channel: 'admin.realtime'
}));

// 개인 알림 구독
ws.send(JSON.stringify({
  type: 'subscribe',
  channel: `user.${userId}.notifications`
}));
```

### 9.2 실시간 이벤트

#### 9.2.1 사용자 상태 변경 이벤트
```json
{
  "type": "user_status_change",
  "data": {
    "userId": "user_12345",
    "status": "online|offline|playing",
    "timestamp": "2025-06-29T10:30:00.000Z",
    "currentBook": "book_12345",
    "sessionDuration": 1800
  }
}
```

#### 9.2.2 어뷰저 감지 알림
```json
{
  "type": "abuse_detected",
  "data": {
    "reportId": "report_12345",
    "userId": "user_12345",
    "reportType": "suspicious_activity",
    "severity": "high",
    "autoActions": ["account_flagged"],
    "timestamp": "2025-06-29T10:30:00.000Z"
  }
}
```

#### 9.2.3 승인 요청 알림
```json
{
  "type": "approval_request",
  "data": {
    "requestId": "req_12345",
    "userId": "user_12345",
    "userEmail": "user@example.com",
    "requestInfo": {
      "name": "김외부",
      "phone": "010-1234-5678",
      "purpose": "시각장애인 지원을 위한 오디오북 이용"
    },
    "timestamp": "2025-06-29T10:30:00.000Z"
  }
}
```

---

## 10. 에러 코드 및 처리

### 10.1 공통 에러 코드

| 코드 | HTTP Status | 설명 |
|------|-------------|------|
| `UNAUTHORIZED` | 401 | 인증되지 않은 요청 |
| `FORBIDDEN` | 403 | 권한이 없는 요청 |
| `NOT_FOUND` | 404 | 리소스를 찾을 수 없음 |
| `VALIDATION_ERROR` | 400 | 요청 데이터 검증 실패 |
| `INTERNAL_ERROR` | 500 | 서버 내부 오류 |
| `RATE_LIMIT_EXCEEDED` | 429 | 요청 제한 초과 |
| `SERVICE_UNAVAILABLE` | 503 | 서비스 일시 중단 |

### 10.2 인증 관련 에러

| 코드 | 설명 |
|------|------|
| `INVALID_TOKEN` | 유효하지 않은 JWT 토큰 |
| `TOKEN_EXPIRED` | 만료된 토큰 |
| `MEMBER_NUMBER_ALREADY_MAPPED` | 이미 사용 중인 성도번호 |
| `INVALID_MEMBER_NUMBER` | 유효하지 않은 성도번호 |
| `USER_NOT_APPROVED` | 승인되지 않은 사용자 |
| `ACCOUNT_SUSPENDED` | 정지된 계정 |

### 10.3 콘텐츠 관련 에러

| 코드 | 설명 |
|------|------|
| `BOOK_NOT_FOUND` | 도서를 찾을 수 없음 |
| `CHAPTER_NOT_FOUND` | 챕터를 찾을 수 없음 |
| `STREAMING_UNAVAILABLE` | 스트리밍 서비스 이용 불가 |
| `INVALID_AUDIO_FORMAT` | 지원하지 않는 오디오 형식 |
| `FILE_SIZE_EXCEEDED` | 파일 크기 제한 초과 |

### 10.4 에러 응답 예시

```json
{
  "success": false,
  "error": {
    "code": "MEMBER_NUMBER_ALREADY_MAPPED",
    "message": "이미 사용 중인 성도번호입니다.",
    "details": {
      "memberNumber": "12345",
      "mappedUserId": "user_67890",
      "mappedAt": "2025-06-20T10:30:00.000Z"
    },
    "suggestions": [
      "다른 성도번호를 사용해 주세요.",
      "본인의 성도번호가 맞다면 관리자에게 문의해 주세요."
    ]
  },
  "timestamp": "2025-06-29T10:30:00.000Z",
  "requestId": "req_abcd1234"
}
```

---

## 11. SDK 및 클라이언트 라이브러리

### 11.1 Flutter SDK 사용 예시

```dart
// Juan Voice API Client
class JuanVoiceApiClient {
  final FirebaseAuth _auth = FirebaseAuth.instance;
  final Dio _dio = Dio();

  JuanVoiceApiClient() {
    _dio.interceptors.add(InterceptorsWrapper(
      onRequest: (options, handler) async {
        final user = _auth.currentUser;
        if (user != null) {
          final token = await user.getIdToken();
          options.headers['Authorization'] = 'Bearer $token';
        }
        handler.next(options);
      },
    ));
  }

  // 성도번호 검증
  Future<MemberValidationResponse> validateMemberNumber(String memberNumber) async {
    final response = await _dio.post(
      '/api/v1/auth/validate-member-number',
      data: {'memberNumber': memberNumber},
    );
    return MemberValidationResponse.fromJson(response.data);
  }

  // 도서 목록 조회
  Future<BookListResponse> getBooks({
    int page = 1,
    int limit = 20,
    String? category,
    String? search,
  }) async {
    final response = await _dio.get(
      '/api/v1/books',
      queryParameters: {
        'page': page,
        'limit': limit,
        if (category != null) 'category': category,
        if (search != null) 'search': search,
      },
    );
    return BookListResponse.fromJson(response.data);
  }

  // 스트리밍 URL 생성
  Future<StreamingUrlResponse> generateStreamingUrl({
    required String bookId,
    required int chapterIndex,
    String quality = 'medium',
  }) async {
    final response = await _dio.post(
      '/api/v1/streaming/generate-url',
      data: {
        'bookId': bookId,
        'chapterIndex': chapterIndex,
        'quality': quality,
      },
    );
    return StreamingUrlResponse.fromJson(response.data);
  }

  // 북마크 저장
  Future<BookmarkResponse> saveBookmark({
    required String bookId,
    required int chapterIndex,
    required int positionSeconds,
    String? note,
  }) async {
    final response = await _dio.put(
      '/api/v1/bookmarks',
      data: {
        'bookId': bookId,
        'chapterIndex': chapterIndex,
        'positionSeconds': positionSeconds,
        if (note != null) 'note': note,
      },
    );
    return BookmarkResponse.fromJson(response.data);
  }
}
```

### 11.2 React Admin SDK 사용 예시

```typescript
// Juan Voice Admin API Client
class JuanVoiceAdminClient {
  private auth: Auth;
  private apiClient: AxiosInstance;

  constructor() {
    this.auth = getAuth();
    this.apiClient = axios.create({
      baseURL: 'https://asia-northeast1-juan-voice.cloudfunctions.net/api',
    });

    this.apiClient.interceptors.request.use(async (config) => {
      const user = this.auth.currentUser;
      if (user) {
        const token = await user.getIdToken();
        config.headers.Authorization = `Bearer ${token}`;
      }
      return config;
    });
  }

  // 승인 요청 목록 조회
  async getApprovalRequests(params: {
    status?: string;
    page?: number;
    limit?: number;
  }): Promise<ApprovalRequestsResponse> {
    const response = await this.apiClient.get('/v1/admin/approval-requests', {
      params,
    });
    return response.data;
  }

  // 사용자 승인 처리
  async approveUser(userId: string, action: 'approve' | 'reject', reason: string) {
    const response = await this.apiClient.post('/v1/admin/approve-user', {
      userId,
      action,
      reason,
    });
    return response.data;
  }

  // 실시간 사용자 현황
  async getRealTimeUsers(): Promise<RealTimeUsersResponse> {
    const response = await this.apiClient.get('/v1/admin/real-time/users');
    return response.data;
  }

  // 성도번호 매핑 해제
  async unmapMemberNumber(memberNumber: string, reason: string) {
    const response = await this.apiClient.post('/v1/admin/unmap-member-number', {
      memberNumber,
      reason,
    });
    return response.data;
  }
}
```

---

## 12. 성능 및 제한사항

### 12.1 Rate Limiting

#### 12.1.1 API 요청 제한
| 엔드포인트 | 제한 | 기간 |
|------------|------|------|
| `/api/v1/auth/*` | 10 requests | 1분 |
| `/api/v1/books` | 100 requests | 1분 |
| `/api/v1/streaming/*` | 50 requests | 1분 |
| `/api/v1/bookmarks` | 200 requests | 1분 |
| `/api/v1/admin/*` | 500 requests | 1분 |

#### 12.1.2 스트리밍 제한
- 동시 스트리밍 세션: 계정당 최대 3개
- 스트리밍 URL 유효 시간: 1시간
- 일일 스트리밍 시간: 계정당 최대 12시간

### 12.2 파일 크기 제한

| 파일 타입 | 최대 크기 |
|-----------|-----------|
| 오디오 파일 | 500MB |
| 커버 이미지 | 5MB |
| 프로필 이미지 | 2MB |

### 12.3 응답 시간 목표

| API 카테고리 | 목표 응답 시간 |
|--------------|----------------|
| 인증 API | < 500ms |
| 콘텐츠 조회 | < 1s |
| 스트리밍 URL 생성 | < 2s |
| 파일 업로드 | < 30s |
| 관리자 API | < 3s |

---

## 13. 버전 관리 및 변경 이력

### 13.1 API 버전 정책
- **Semantic Versioning**: v1.0.0 형식 사용
- **하위 호환성**: 마이너 버전에서는 하위 호환성 유지
- **Deprecation**: 최소 6개월 전 공지
- **Support**: 최신 버전 + 이전 1개 버전 지원

### 13.2 변경 이력

| 버전 | 날짜 | 변경 사항 |
|------|------|-----------|
| v1.0.0 | 2025-06-29 | 초기 API 명세서 작성 |

### 13.3 관련 문서
- **PRD v1.0**: Product Requirements Document
- **FRD v1.1**: Functional Requirements Document
- **System Architecture v1.0**: 시스템 아키텍처 문서
- **Security Policy**: 보안 정책 및 가이드라인

---

**Document Version**: v1.0  
**Last Updated**: 2025년 6월 29일  
**API Base URL**: `https://asia-northeast1-juan-voice.cloudfunctions.net/api`  
**Support**: dev-team@juan-voice.com