# 주안의소리 - 시스템 아키텍처 문서

**System Architecture Document v1.0**  
**프로젝트명**: 주안의소리 (Voice of Juan)  
**문서 작성일**: 2025년 6월 29일  
**담당자**: 시스템 아키텍트  
**승인자**: CTO  
**관련 문서**: PRD v1.0, FRD v1.1

---

## 1. 문서 개요

### 1.1 목적
본 문서는 주안의소리 시각장애인용 오디오북 앱의 전체 시스템 아키텍처를 정의하고, 각 구성 요소 간의 관계와 데이터 흐름을 명시합니다.

### 1.2 범위
- 모바일 애플리케이션 아키텍처 (Flutter)
- 웹 관리자 시스템 아키텍처 (React)
- 백엔드 서비스 아키텍처 (Firebase)
- 보안 및 인증 시스템
- 데이터 저장 및 스트리밍 아키텍처
- 모니터링 및 로깅 시스템

### 1.3 아키텍처 원칙
- **접근성 우선**: 시각장애인을 위한 최적화된 설계
- **보안 강화**: DRM 및 어뷰저 방지 시스템
- **확장성**: 동시 사용자 1,000명까지 지원
- **안정성**: 99.9% 가용성 목표
- **성능**: 스트리밍 시작 5초 이내

---

## 2. 전체 시스템 아키텍처

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Presentation Layer                       │
├─────────────────────────────┬───────────────────────────────────┤
│     Mobile App (Flutter)    │     Web Admin (React)            │
│  - User Authentication      │  - Admin Dashboard               │
│  - Audio Streaming          │  - User Management               │
│  - Bookmark Management      │  - Content Management            │
│  - Accessibility Features   │  - Real-time Monitoring          │
└─────────────────────────────┴───────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │   API Gateway     │
                    │  (Firebase Auth)  │
                    └─────────┬─────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                     Application Layer                           │
├─────────────────────────────┬───────────────────────────────────┤
│    Authentication Service   │     Content Management Service    │
│  - Google OAuth             │  - Book Metadata                 │
│  - Member Number Validation │  - Chapter Management            │
│  - Abuse Detection          │  - Category Management           │
│  - Session Management       │  - Search & Filtering            │
├─────────────────────────────┼───────────────────────────────────┤
│    Streaming Service        │     Analytics Service            │
│  - Audio Streaming          │  - Play Logs                     │
│  - DRM Protection           │  - User Statistics               │
│  - Quality Adaptation       │  - Real-time Monitoring          │
│  - Bookmark Sync            │  - Abuse Detection               │
└─────────────────────────────┴───────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                      Data Layer                                 │
├─────────────────────────────┬───────────────────────────────────┤
│    Cloud Firestore         │     Firebase Storage              │
│  - User Profiles            │  - Audio Files                   │
│  - Book Metadata            │  - Cover Images                  │
│  - Bookmarks                │  - App Assets                    │
│  - Analytics Data           │                                  │
│  - Admin Settings           │     CDN (Cloud CDN)              │
│  - Abuse Reports            │  - Global Distribution           │
└─────────────────────────────┴───────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                   Infrastructure Layer                          │
├─────────────────────────────┬───────────────────────────────────┤
│    Firebase Hosting         │     Cloud Functions              │
│  - Web Admin Hosting        │  - Scheduled Tasks               │
│  - Static Assets            │  - Webhook Handlers              │
│                             │  - Background Processing          │
├─────────────────────────────┼───────────────────────────────────┤
│    Firebase Extensions      │     Monitoring & Logging         │
│  - Analytics                │  - Crashlytics                   │
│  - Performance Monitoring   │  - Cloud Logging                 │
│  - Crashlytics              │  - Error Reporting               │
└─────────────────────────────┴───────────────────────────────────┘
```

### 2.2 Component Interaction Diagram

```
Mobile App ──────────────────┐
                            │
Web Admin ───────────────────┼─────→ Firebase Auth ──────────────┐
                            │                                   │
                            │                                   ▼
                            ▼                          User Authentication
                    API Gateway                              │
                            │                               │
                            ▼                               ▼
              ┌─────────────────────────────┐      Cloud Firestore
              │    Business Logic Layer     │              │
              │                            │              │
              │  ┌─────────────────────────┐ │              │
              │  │  Authentication Svc     │ │◄─────────────┘
              │  └─────────────────────────┘ │
              │  ┌─────────────────────────┐ │
              │  │  Content Management     │ │
              │  └─────────────────────────┘ │
              │  ┌─────────────────────────┐ │
              │  │  Streaming Service      │ │◄──── Firebase Storage
              │  └─────────────────────────┘ │           │
              │  ┌─────────────────────────┐ │           │
              │  │  Analytics Service      │ │           ▼
              │  └─────────────────────────┘ │         CDN
              └─────────────────────────────┘
```

---

## 3. 기술 스택 아키텍처

### 3.1 모바일 애플리케이션 (Flutter)

#### 3.1.1 애플리케이션 구조
```
lib/
├── main.dart
├── core/
│   ├── constants/
│   ├── errors/
│   ├── utils/
│   └── network/
├── features/
│   ├── authentication/
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   ├── content/
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   ├── streaming/
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   └── accessibility/
├── shared/
│   ├── widgets/
│   ├── themes/
│   └── services/
└── injection_container.dart
```

#### 3.1.2 주요 패키지 및 의존성
```yaml
dependencies:
  flutter: ^3.19.0
  
  # State Management
  flutter_riverpod: ^2.4.0
  
  # Network & HTTP
  dio: ^5.4.0
  retrofit: ^4.0.3
  
  # Firebase
  firebase_core: ^2.24.0
  firebase_auth: ^4.15.0
  cloud_firestore: ^4.13.0
  firebase_storage: ^11.5.0
  firebase_analytics: ^10.7.0
  firebase_crashlytics: ^3.4.0
  
  # Audio Streaming
  just_audio: ^0.9.36
  audio_service: ^0.18.12
  
  # Local Storage
  hive: ^2.2.3
  hive_flutter: ^1.1.0
  
  # UI & Accessibility
  flutter_screenutil: ^5.9.0
  auto_size_text: ^3.0.0
  
  # Utilities
  equatable: ^2.0.5
  freezed_annotation: ^2.4.1
  json_annotation: ^4.8.1
  
dev_dependencies:
  flutter_test: ^3.19.0
  mockito: ^5.4.4
  integration_test: ^3.19.0
  build_runner: ^2.4.7
  freezed: ^2.4.6
  json_serializable: ^6.7.1
```

#### 3.1.3 아키텍처 패턴
- **Clean Architecture**: Domain, Data, Presentation 계층 분리
- **Repository Pattern**: 데이터 소스 추상화
- **Dependency Injection**: GetIt 패키지 활용
- **State Management**: Riverpod 사용

### 3.2 웹 관리자 시스템 (React)

#### 3.2.1 프로젝트 구조
```
src/
├── components/
│   ├── common/
│   ├── layout/
│   ├── charts/
│   └── forms/
├── pages/
│   ├── dashboard/
│   ├── users/
│   ├── content/
│   ├── analytics/
│   └── settings/
├── hooks/
├── services/
│   ├── auth.service.ts
│   ├── users.service.ts
│   ├── content.service.ts
│   └── analytics.service.ts
├── types/
├── utils/
├── constants/
└── contexts/
```

#### 3.2.2 주요 패키지 및 의존성
```json
{
  "dependencies": {
    "react": "^18.2.0",
    "typescript": "^5.0.0",
    
    "firebase": "^10.7.0",
    "firebase-admin": "^12.0.0",
    
    "@mui/material": "^5.15.0",
    "@mui/icons-material": "^5.15.0",
    "@mui/x-data-grid": "^6.18.0",
    "@mui/x-charts": "^6.18.0",
    
    "react-router-dom": "^6.20.0",
    "react-query": "^3.39.3",
    "zustand": "^4.4.7",
    
    "recharts": "^2.8.0",
    "date-fns": "^2.30.0",
    "react-hook-form": "^7.48.0",
    "yup": "^1.4.0",
    
    "axios": "^1.6.0",
    "lodash": "^4.17.21"
  },
  "devDependencies": {
    "@testing-library/react": "^14.1.0",
    "@testing-library/jest-dom": "^6.1.0",
    "jest": "^29.7.0",
    "eslint": "^8.55.0",
    "prettier": "^3.1.0"
  }
}
```

### 3.3 백엔드 서비스 (Firebase)

#### 3.3.1 Firebase 서비스 구성
```
Firebase Project
├── Authentication
│   ├── Google OAuth Provider
│   ├── Custom Claims (admin, member)
│   └── Security Rules
├── Firestore Database
│   ├── Collections (8개)
│   ├── Security Rules
│   └── Indexes
├── Storage
│   ├── Audio Files (/audio/)
│   ├── Cover Images (/covers/)
│   └── Security Rules
├── Cloud Functions
│   ├── Auth Triggers
│   ├── Scheduled Functions
│   └── HTTP Triggers
├── Hosting
│   └── Web Admin
└── Extensions
    ├── Analytics
    ├── Crashlytics
    └── Performance Monitoring
```

#### 3.3.2 Cloud Functions 구조
```
functions/
├── src/
│   ├── auth/
│   │   ├── onUserCreate.ts
│   │   ├── onUserDelete.ts
│   │   └── validateMemberNumber.ts
│   ├── content/
│   │   ├── onBookCreate.ts
│   │   ├── generateStreamingUrl.ts
│   │   └── validateAudioFile.ts
│   ├── analytics/
│   │   ├── aggregatePlayLogs.ts
│   │   ├── generateDailyReport.ts
│   │   └── detectAbusiveUser.ts
│   ├── admin/
│   │   ├── approveExternalUser.ts
│   │   ├── suspendUser.ts
│   │   └── unmapMemberNumber.ts
│   └── utils/
│       ├── firestore.ts
│       ├── storage.ts
│       └── notifications.ts
├── package.json
└── tsconfig.json
```

---

## 4. 데이터 아키텍처

### 4.1 데이터베이스 설계 (Firestore)

#### 4.1.1 Collection 구조
```
주안의소리 Database
├── users
├── books
├── bookmarks
├── userStats
├── playLogs
├── adminSettings
├── memberMappingLogs
└── abuseReports
```

#### 4.1.2 데이터 관계도
```
users (1) ←──────→ (1) userStats
  │
  ├── (1:N) ────→ bookmarks
  ├── (1:N) ────→ playLogs
  ├── (1:N) ────→ memberMappingLogs
  └── (1:N) ────→ abuseReports

books (1) ←──────→ (N) bookmarks
  │
  └── (1:N) ────→ playLogs

adminSettings (1) ←─→ (N) memberMappingLogs
```

#### 4.1.3 Firestore 인덱스 설계
```javascript
// Composite Indexes
{
  collectionGroup: "playLogs",
  fields: [
    { fieldPath: "userId", order: "ASCENDING" },
    { fieldPath: "timestamp", order: "DESCENDING" }
  ]
},
{
  collectionGroup: "bookmarks",
  fields: [
    { fieldPath: "userId", order: "ASCENDING" },
    { fieldPath: "lastUpdated", order: "DESCENDING" }
  ]
},
{
  collectionGroup: "users",
  fields: [
    { fieldPath: "userType", order: "ASCENDING" },
    { fieldPath: "isApproved", order: "ASCENDING" },
    { fieldPath: "createdAt", order: "DESCENDING" }
  ]
},
{
  collectionGroup: "memberMappingLogs",
  fields: [
    { fieldPath: "memberNumber", order: "ASCENDING" },
    { fieldPath: "isActive", order: "ASCENDING" },
    { fieldPath: "timestamp", order: "DESCENDING" }
  ]
}
```

### 4.2 데이터 흐름 아키텍처

#### 4.2.1 사용자 인증 플로우
```
User Input ──→ Google OAuth ──→ Firebase Auth ──→ Validation
    │                                               │
    ▼                                               ▼
Member Number ────────────────────────────────→ Firestore
    │                                               │
    ▼                                               ▼
Validation ────────────────────────────────────→ User Creation
    │                                               │
    ▼                                               ▼
Member Mapping ────────────────────────────────→ Session Start
```

#### 4.2.2 스트리밍 데이터 플로우
```
User Request ──→ Authentication ──→ Authorization ──→ Storage Access
     │                                                     │
     ▼                                                     ▼
Play Log ←─────────────────────────────────────────── CDN Delivery
     │                                                     │
     ▼                                                     ▼
Analytics ←───────────────────────────────────────── Audio Stream
     │                                                     │
     ▼                                                     ▼
Real-time ←───────────────────────────────────────── Client Playback
Dashboard
```

---

## 5. 보안 아키텍처

### 5.1 인증 및 권한 관리

#### 5.1.1 Firebase Security Rules
```javascript
// Firestore Security Rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Users 컬렉션 - 본인 데이터만 접근
    match /users/{userId} {
      allow read, write: if request.auth != null && 
                           request.auth.uid == userId;
    }
    
    // Books 컬렉션 - 승인된 사용자만 읽기
    match /books/{bookId} {
      allow read: if request.auth != null && 
                     isApprovedUser(request.auth.uid) &&
                     isActiveUser(request.auth.uid);
      allow write: if hasAdminRole(request.auth.uid);
    }
    
    // Bookmarks 컬렉션 - 사용자 개인 데이터
    match /bookmarks/{bookmarkId} {
      allow read, write: if request.auth != null && 
                           resource.data.userId == request.auth.uid;
    }
    
    // PlayLogs 컬렉션 - 사용자별 로그
    match /playLogs/{logId} {
      allow create: if request.auth != null && 
                      request.resource.data.userId == request.auth.uid;
      allow read: if request.auth != null && 
                    (resource.data.userId == request.auth.uid || 
                     hasAdminRole(request.auth.uid));
    }
    
    // Admin 전용 컬렉션들
    match /adminSettings/{settingId} {
      allow read, write: if hasAdminRole(request.auth.uid);
    }
    
    match /memberMappingLogs/{logId} {
      allow read, write: if hasAdminRole(request.auth.uid);
    }
    
    match /abuseReports/{reportId} {
      allow read, write: if hasAdminRole(request.auth.uid);
    }
    
    // Helper Functions
    function isApprovedUser(uid) {
      return exists(/databases/$(database)/documents/users/$(uid)) &&
             get(/databases/$(database)/documents/users/$(uid)).data.isApproved == true;
    }
    
    function isActiveUser(uid) {
      return exists(/databases/$(database)/documents/users/$(uid)) &&
             get(/databases/$(database)/documents/users/$(uid)).data.isActive == true;
    }
    
    function hasAdminRole(uid) {
      return request.auth.token.admin == true;
    }
  }
}

// Firebase Storage Security Rules
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    // 오디오 파일 - 승인된 사용자만 스트리밍 접근
    match /audio/{allPaths=**} {
      allow read: if request.auth != null && 
                     isApprovedUser(request.auth.uid) &&
                     isActiveUser(request.auth.uid);
      allow write: if hasAdminRole(request.auth.uid);
    }
    
    // 커버 이미지 - 인증된 사용자 읽기
    match /covers/{allPaths=**} {
      allow read: if request.auth != null;
      allow write: if hasAdminRole(request.auth.uid);
    }
  }
}
```

#### 5.1.2 DRM 및 콘텐츠 보호
```
┌─────────────────────────────────────────────────────────────────┐
│                       DRM Protection Layer                      │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐    ┌─────────────────┐    ┌──────────────┐ │
│  │  Session Token  │    │  Stream Token   │    │  Time-based  │ │
│  │  Validation     │    │  Generation     │    │  Expiration  │ │
│  └─────────────────┘    └─────────────────┘    └──────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐    ┌─────────────────┐    ┌──────────────┐ │
│  │  IP Restriction │    │  Device Limit   │    │  Watermarking│ │
│  │  & Geofencing   │    │  (Max 3 devices)│    │  (User ID)   │ │
│  └─────────────────┘    └─────────────────┘    └──────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 어뷰저 감지 및 방지 시스템

#### 5.2.1 1계정-1성도번호 매핑 검증
```typescript
// Cloud Function: 성도번호 중복 검증
export const validateMemberNumberMapping = functions.https.onCall(
  async (data, context) => {
    const { memberNumber, userId } = data;
    
    // 1. 현재 성도번호가 다른 계정에 매핑되어 있는지 확인
    const existingMapping = await admin.firestore()
      .collection('memberMappingLogs')
      .where('memberNumber', '==', memberNumber)
      .where('isActive', '==', true)
      .get();
    
    if (!existingMapping.empty) {
      // 2. 이미 매핑된 성도번호인 경우 차단
      await logAbuseAttempt(userId, 'duplicate_member_number', memberNumber);
      throw new functions.https.HttpsError(
        'already-exists',
        '이미 사용 중인 성도번호입니다.'
      );
    }
    
    // 3. 성도번호 유효성 검증
    const validMemberNumbers = await getValidMemberNumbers();
    if (!validMemberNumbers.includes(memberNumber)) {
      await logAbuseAttempt(userId, 'invalid_member_number', memberNumber);
      throw new functions.https.HttpsError(
        'not-found',
        '등록되지 않은 성도번호입니다.'
      );
    }
    
    return { valid: true };
  }
);
```

#### 5.2.2 실시간 어뷰저 감지
```typescript
// 의심스러운 활동 패턴 감지
export const detectSuspiciousActivity = functions.firestore
  .document('playLogs/{logId}')
  .onCreate(async (snap, context) => {
    const logData = snap.data();
    const userId = logData.userId;
    
    // 1. 짧은 시간 내 과도한 재생 요청
    const recentLogs = await admin.firestore()
      .collection('playLogs')
      .where('userId', '==', userId)
      .where('timestamp', '>', new Date(Date.now() - 5 * 60 * 1000))
      .get();
    
    if (recentLogs.size > 10) {
      await flagUserForReview(userId, 'excessive_requests');
    }
    
    // 2. 비정상적인 IP 패턴
    const userIpHistory = await getUserIpHistory(userId);
    if (hasAnomalousIpPattern(userIpHistory)) {
      await flagUserForReview(userId, 'suspicious_ip_pattern');
    }
    
    // 3. 동시 다중 접속 감지
    const activeSessions = await getActiveUserSessions(userId);
    if (activeSessions.length > 3) {
      await flagUserForReview(userId, 'multiple_concurrent_sessions');
    }
  });
```

---

## 6. 네트워크 및 CDN 아키텍처

### 6.1 CDN 구조

#### 6.1.1 Global CDN Distribution
```
┌─────────────────────────────────────────────────────────────────┐
│                      Global CDN Network                         │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │   Seoul     │  │   Busan     │  │   Tokyo     │  │ Singapore│ │
│  │ Edge Server │  │ Edge Server │  │ Edge Server │  │Edge Server│ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └──────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                     Origin Server (GCS)                         │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              Firebase Storage Bucket                        │ │
│  │  - Audio Files (High Quality)                              │ │
│  │  - Cover Images                                             │ │
│  │  - Transcoded Multiple Bitrates                             │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

#### 6.1.2 Adaptive Streaming 구조
```
Original Audio File
        │
        ▼
┌─────────────────┐
│  Transcoding    │
│   Service       │
└─────────────────┘
        │
        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   64 kbps       │    │   128 kbps      │    │   256 kbps      │
│  (Mobile Data)  │    │  (WiFi/3G)      │    │  (High Quality) │
└─────────────────┘    └─────────────────┘    └─────────────────┘
        │                       │                       │
        └───────────────────────┼───────────────────────┘
                               │
                               ▼
                    ┌─────────────────┐
                    │  Client Player  │
                    │ (Auto-Selection)│
                    └─────────────────┘
```

### 6.2 네트워크 최적화

#### 6.2.1 Connection Management
```typescript
// Network State Management
class NetworkManager {
  private connectionType: ConnectionType;
  private bandwidth: number;
  private latency: number;
  
  async optimizeStreamingQuality(): Promise<StreamingQuality> {
    const networkInfo = await this.getNetworkInfo();
    
    if (networkInfo.type === 'wifi' && networkInfo.bandwidth > 2000) {
      return StreamingQuality.HIGH; // 256 kbps
    } else if (networkInfo.bandwidth > 500) {
      return StreamingQuality.MEDIUM; // 128 kbps
    } else {
      return StreamingQuality.LOW; // 64 kbps
    }
  }
  
  async handleConnectionChange(): Promise<void> {
    const previousQuality = this.currentQuality;
    const newQuality = await this.optimizeStreamingQuality();
    
    if (previousQuality !== newQuality) {
      await this.switchStreamingQuality(newQuality);
      this.notifyQualityChange(newQuality);
    }
  }
}
```

---

## 7. 배포 및 DevOps 아키텍처

### 7.1 CI/CD Pipeline

#### 7.1.1 GitHub Actions Workflow
```yaml
# .github/workflows/main.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test-mobile:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: subosito/flutter-action@v2
      - run: flutter pub get
      - run: flutter test
      - run: flutter analyze
      
  test-web-admin:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run test
      - run: npm run lint
      
  test-functions:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: cd functions && npm ci
      - run: cd functions && npm run test
      
  deploy-staging:
    needs: [test-mobile, test-web-admin, test-functions]
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: firebase deploy --only hosting,functions --project staging
      
  deploy-production:
    needs: [test-mobile, test-web-admin, test-functions]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: firebase deploy --project production
```

### 7.2 환경 분리

#### 7.2.1 Multi-Environment Setup
```
Development Environment
├── Firebase Project: juan-voice-dev
├── Database: Firestore (dev)
├── Storage: Firebase Storage (dev)
└── Hosting: juan-voice-dev.web.app

Staging Environment
├── Firebase Project: juan-voice-staging
├── Database: Firestore (staging)
├── Storage: Firebase Storage (staging)
└── Hosting: juan-voice-staging.web.app

Production Environment
├── Firebase Project: juan-voice-prod
├── Database: Firestore (prod)
├── Storage: Firebase Storage (prod)
└── Hosting: juan-voice.com
```

---

## 8. 모니터링 및 로깅 아키텍처

### 8.1 실시간 모니터링 시스템

#### 8.1.1 Monitoring Dashboard
```
┌─────────────────────────────────────────────────────────────────┐
│                    Real-time Monitoring                         │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │System Health│  │User Activity│  │Content Usage│  │Security  │ │
│  │- Uptime     │  │- Active Users│  │- Play Stats │  │- Abuse   │ │
│  │- Response   │  │- Sessions   │  │- Popular    │  │- Failed  │ │
│  │- Errors     │  │- Geolocation│  │- Duration   │  │  Logins  │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └──────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │Performance  │  │Network      │  │Storage      │  │Alerts    │ │
│  │- Load Time  │  │- Bandwidth  │  │- Usage      │  │- Critical│ │
│  │- Memory     │  │- Latency    │  │- Costs      │  │- Warning │ │
│  │- CPU        │  │- CDN Hits   │  │- Growth     │  │- Info    │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └──────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

#### 8.1.2 Alert System
```typescript
// Cloud Function: Alert System
export const alertSystem = functions.pubsub
  .schedule('every 1 minutes')
  .onRun(async (context) => {
    // 1. System Health Checks
    const healthMetrics = await getSystemHealthMetrics();
    
    if (healthMetrics.errorRate > 0.05) { // 5% 이상 에러율
      await sendAlert({
        type: 'CRITICAL',
        message: `Error rate exceeded threshold: ${healthMetrics.errorRate}`,
        channels: ['slack', 'email']
      });
    }
    
    // 2. Security Monitoring
    const securityEvents = await getRecentSecurityEvents();
    
    if (securityEvents.abuseAttempts > 10) {
      await sendAlert({
        type: 'WARNING',
        message: `High number of abuse attempts: ${securityEvents.abuseAttempts}`,
        channels: ['slack']
      });
    }
    
    // 3. Performance Monitoring
    const performanceMetrics = await getPerformanceMetrics();
    
    if (performanceMetrics.avgResponseTime > 2000) { // 2초 이상
      await sendAlert({
        type: 'WARNING',
        message: `Slow response time: ${performanceMetrics.avgResponseTime}ms`,
        channels: ['slack']
      });
    }
  });
```

### 8.2 로깅 시스템

#### 8.2.1 Structured Logging
```typescript
// 통합 로깅 시스템
class Logger {
  static info(message: string, metadata?: any) {
    console.log(JSON.stringify({
      timestamp: new Date().toISOString(),
      level: 'INFO',
      message,
      metadata,
      service: 'juan-voice',
      version: process.env.APP_VERSION
    }));
  }
  
  static warn(message: string, metadata?: any) {
    console.warn(JSON.stringify({
      timestamp: new Date().toISOString(),
      level: 'WARN',
      message,
      metadata,
      service: 'juan-voice',
      version: process.env.APP_VERSION
    }));
  }
  
  static error(message: string, error?: Error, metadata?: any) {
    console.error(JSON.stringify({
      timestamp: new Date().toISOString(),
      level: 'ERROR',
      message,
      error: error ? {
        name: error.name,
        message: error.message,
        stack: error.stack
      } : undefined,
      metadata,
      service: 'juan-voice',
      version: process.env.APP_VERSION
    }));
  }
}
```

---

## 9. 성능 및 확장성 아키텍처

### 9.1 성능 최적화 전략

#### 9.1.1 Caching Strategy
```
┌─────────────────────────────────────────────────────────────────┐
│                        Caching Layers                           │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │   Browser   │  │     CDN     │  │  Firebase   │  │ Function │ │
│  │   Cache     │  │    Cache    │  │   Cache     │  │  Memory  │ │
│  │             │  │             │  │             │  │  Cache   │ │
│  │ - UI Assets │  │ - Audio     │  │ - Query     │  │ - Config │ │
│  │ - Metadata  │  │ - Images    │  │   Results   │  │ - Session│ │
│  │ - User Data │  │ - Static    │  │ - Computed  │  │ - Temp   │ │
│  │             │  │   Content   │  │   Values    │  │   Data   │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └──────────┘ │
│       1-4h             24h              1h             5m        │
└─────────────────────────────────────────────────────────────────┘
```

#### 9.1.2 Database Optimization
```typescript
// Firestore 쿼리 최적화
class OptimizedQueries {
  // 페이지네이션을 통한 대량 데이터 처리
  async getPaginatedBooks(lastVisible?: DocumentSnapshot, limit = 20) {
    let query = this.firestore
      .collection('books')
      .where('isActive', '==', true)
      .orderBy('createdAt', 'desc')
      .limit(limit);
    
    if (lastVisible) {
      query = query.startAfter(lastVisible);
    }
    
    return await query.get();
  }
  
  // 인덱스 최적화된 복합 쿼리
  async getUserActivitySummary(userId: string, startDate: Date, endDate: Date) {
    return await this.firestore
      .collection('playLogs')
      .where('userId', '==', userId)
      .where('timestamp', '>=', startDate)
      .where('timestamp', '<=', endDate)
      .orderBy('timestamp', 'desc')
      .get();
  }
  
  // 배치 처리를 통한 효율적인 업데이트
  async batchUpdateBookmarks(updates: BookmarkUpdate[]) {
    const batch = this.firestore.batch();
    
    updates.forEach(update => {
      const ref = this.firestore.collection('bookmarks').doc(update.id);
      batch.update(ref, update.data);
    });
    
    return await batch.commit();
  }
}
```

### 9.2 확장성 계획

#### 9.2.1 Horizontal Scaling Strategy
```
Current Architecture (Phase 1)
├── Single Firebase Project
├── Single Region (asia-northeast1)
├── Max 1,000 concurrent users
└── 1TB storage capacity

Scaling Phase 2 (6 months)
├── Multi-region deployment
├── Load balancing
├── Max 5,000 concurrent users
└── 5TB storage capacity

Scaling Phase 3 (1 year)
├── Microservices architecture
├── Dedicated streaming servers
├── Max 20,000 concurrent users
└── 20TB storage capacity
```

---

## 10. 재해 복구 및 백업 전략

### 10.1 백업 시스템

#### 10.1.1 Data Backup Strategy
```typescript
// 자동화된 백업 시스템
export const dailyBackup = functions.pubsub
  .schedule('0 2 * * *') // 매일 오전 2시
  .timeZone('Asia/Seoul')
  .onRun(async (context) => {
    // 1. Firestore 데이터 백업
    await backupFirestoreCollections([
      'users',
      'books',
      'bookmarks',
      'userStats',
      'adminSettings'
    ]);
    
    // 2. Storage 파일 백업 (증분)
    await incrementalStorageBackup();
    
    // 3. 백업 상태 확인 및 알림
    await verifyBackupIntegrity();
    
    return null;
  });

async function backupFirestoreCollections(collections: string[]) {
  for (const collection of collections) {
    await admin.firestore().runTransaction(async (transaction) => {
      // Cloud Storage로 데이터 내보내기
      const backupBucket = admin.storage().bucket('juan-voice-backups');
      const timestamp = new Date().toISOString().split('T')[0];
      
      // 백업 파일 생성 및 업로드
      await createCollectionBackup(collection, `${timestamp}/${collection}.json`);
    });
  }
}
```

### 10.2 재해 복구 계획

#### 10.2.1 Disaster Recovery Plan
```
RTO (Recovery Time Objective): 4시간
RPO (Recovery Point Objective): 1시간

재해 유형별 복구 절차:

1. Firebase Service Outage
   ├── 감지: 자동 모니터링 (5분 이내)
   ├── 알림: 관리팀 즉시 통보
   ├── 조치: Google Cloud Status 확인
   └── 복구: 서비스 복구 시까지 대기

2. Data Corruption
   ├── 감지: 데이터 무결성 검사
   ├── 격리: 손상된 데이터 식별
   ├── 복구: 최신 백업에서 복원
   └── 검증: 데이터 일관성 확인

3. Security Breach
   ├── 감지: 보안 모니터링 시스템
   ├── 격리: 영향받은 계정 즉시 정지
   ├── 분석: 침해 범위 및 원인 조사
   └── 복구: 보안 패치 및 시스템 강화
```

---

## 11. 문서 관리 및 참조

### 11.1 버전 관리
- **문서 버전**: v1.0
- **마지막 업데이트**: 2025년 6월 29일
- **다음 리뷰 예정**: 2025년 7월 29일

### 11.2 관련 문서
- **PRD v1.0**: Product Requirements Document
- **FRD v1.1**: Functional Requirements Document  
- **API 명세서**: RESTful API 및 GraphQL 스키마
- **보안 정책서**: 데이터 보호 및 접근 제어 정책
- **운영 매뉴얼**: 서비스 운영 및 장애 대응 가이드
- **테스트 계획서**: QA 및 성능 테스트 전략

### 11.3 승인 이력
| 날짜 | 버전 | 승인자 | 변경 사항 |
|------|------|--------|-----------|
| 2025-06-29 | v1.0 | CTO | 초기 시스템 아키텍처 문서 작성 |

---

**Document Version**: v1.0  
**Last Updated**: 2025년 6월 29일  
**Architect**: 시스템 아키텍트팀  
**Approved**: CTO