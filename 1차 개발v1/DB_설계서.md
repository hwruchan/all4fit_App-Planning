# 모두의 핏 앱 DB 설계서

## 1. 문서 개요

### 1.1 프로젝트 정보
- **프로젝트명**: 모두의 핏 (All4Fit) 앱
- **데이터베이스**: MySQL 8.0
- **버전**: 2.0
- **작성일**: 2025년 10월
- **기반 문서**: IA 명세서, 기능 정의서, API 명세서

### 1.2 설계 원칙
- 1차 개발 범위의 핵심 기능만 구현
- 정규화를 통한 데이터 중복 최소화
- 확장성을 고려한 구조 설계
- 성능 최적화를 위한 인덱스 설계
- 데이터 무결성 보장
- API 명세서와 일치하는 데이터 구조

## 2. 데이터베이스 ERD

### 2.1 전체 ERD 구조
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│     USERS       │    │   INSTRUCTORS   │    │FACILITY_OPERATORS│
│─────────────────│    │─────────────────│    │─────────────────│
│ user_id (PK)    │◄───┤ user_id (FK)    │    │ user_id (FK)    │
│ email           │    │ instructor_id   │    │ operator_id     │
│ phone           │    │ business_number │    │ business_number │
│ password_hash   │    │ certification   │    │ representative  │
│ name            │    │ experience      │    │ operating_hours │
│ nickname        │    │ specializations │    │ status          │
│ birth_date      │    │ hourly_rate     │    └─────────────────┘
│ gender          │    │ status          │             │
│ user_type       │    └─────────────────┘             │
│ status          │             │                      │
└─────────────────┘             │                      │
         │                      │                      │
         │                      ▼                      ▼
         │            ┌─────────────────┐    ┌─────────────────┐
         │            │INSTRUCTOR_FACIL.│    │   FACILITIES    │
         │            │─────────────────│    │─────────────────│
         │            │instructor_id(FK)│    │ facility_id(PK) │
         │            │facility_name    │    │ operator_id(FK) │
         │            │facility_address │    │ name            │
         │            │latitude         │    │ description     │
         │            │longitude        │    │ address         │
         │            └─────────────────┘    │ latitude        │
         │                                   │ longitude       │
         │                                   │ facility_type   │
         │                                   │ average_rating  │
         │                                   │ status          │
         │                                   └─────────────────┘
         │                                            │
         │                                            │
         ▼                                            ▼
┌─────────────────┐                         ┌─────────────────┐
│ USER_PROFILES   │                         │ FACILITY_IMAGES │
│─────────────────│                         │─────────────────│
│ profile_id(PK)  │                         │ image_id(PK)    │
│ user_id (FK)    │                         │ facility_id(FK) │
│ bio             │                         │ image_url       │
│ profile_image   │                         │ alt_text        │
│ preferences     │                         │ sort_order      │
└─────────────────┘                         └─────────────────┘

┌─────────────────┐    ┌─────────────────┐
│     SPORTS      │    │ FACILITY_SPORTS │
│─────────────────│    │─────────────────│
│ sport_id (PK)   │◄───┤ sport_id (FK)   │
│ name            │    │ facility_id(FK) │
│ description     │    │ price_per_hour  │
│ icon_url        │    │ description     │
│ category        │    │ is_available    │
│ is_active       │    └─────────────────┘
└─────────────────┘
```

### 2.2 주요 관계
- **USERS** ↔ **INSTRUCTORS**: 1:1 (지도자)
- **USERS** ↔ **FACILITY_OPERATORS**: 1:1 (시설운영자)
- **FACILITY_OPERATORS** → **FACILITIES**: 1:N (시설운영자가 등록한 시설)
- **INSTRUCTORS** → **INSTRUCTOR_FACILITIES**: 1:N (지도자가 사용하는 시설)
- **FACILITIES** ↔ **SPORTS**: N:M (FACILITY_SPORTS)
- **USERS** → **USER_PROFILES**: 1:1
- **USERS** → **USER_LOCATIONS**: 1:N
- **FACILITIES** → **FACILITY_IMAGES**: 1:N


## 3. 테이블 상세 설계

### 3.1 사용자 관련 테이블

#### 3.1.1 USERS (사용자)
| 컬럼명 | 데이터타입 | 제약조건 | 설명 |
|--------|------------|----------|------|
| user_id | INT | PK, AUTO_INCREMENT | 사용자 고유 ID |
| email | VARCHAR(255) | UNIQUE, NOT NULL | 이메일 주소 |
| phone | VARCHAR(20) | UNIQUE | 전화번호 |
| password_hash | VARCHAR(255) | | 암호화된 비밀번호 (소셜로그인시 NULL) |
| name | VARCHAR(100) | NOT NULL | 실명 |
| nickname | VARCHAR(50) | UNIQUE, NOT NULL | 닉네임 |
| birth_date | DATE | | 생년월일 |
| gender | ENUM('M', 'F', 'OTHER') | | 성별 |
| address | VARCHAR(500) | | 주소 |
| user_type | ENUM('USER', 'INSTRUCTOR', 'OPERATOR', 'ADMIN') | DEFAULT 'USER' | 사용자 유형 |
| status | ENUM('ACTIVE', 'INACTIVE', 'SUSPENDED', 'DELETED', 'PENDING') | DEFAULT 'PENDING' | 계정 상태 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 생성일시 |
| updated_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 수정일시 |
| last_login_at | TIMESTAMP | | 마지막 로그인 시간 |

**인덱스:**
- PRIMARY KEY (user_id)
- UNIQUE KEY uk_email (email)
- UNIQUE KEY uk_phone (phone)
- UNIQUE KEY uk_nickname (nickname)
- KEY idx_user_type (user_type)
- KEY idx_status (status)

#### 3.1.2 USER_PROFILES (사용자 프로필)
| 컬럼명 | 데이터타입 | 제약조건 | 설명 |
|--------|------------|----------|------|
| profile_id | INT | PK, AUTO_INCREMENT | 프로필 고유 ID |
| user_id | INT | FK, NOT NULL | 사용자 ID |
| bio | TEXT | | 자기소개 |
| profile_image_url | VARCHAR(500) | | 프로필 이미지 URL |
| preferences | JSON | | 사용자 설정 (JSON) |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 생성일시 |
| updated_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 수정일시 |

**인덱스:**
- PRIMARY KEY (profile_id)
- UNIQUE KEY uk_user_id (user_id)
- FOREIGN KEY fk_user_profiles_user_id (user_id) REFERENCES USERS(user_id)

#### 3.1.3 USER_LOCATIONS (사용자 위치)
| 컬럼명 | 데이터타입 | 제약조건 | 설명 |
|--------|------------|----------|------|
| location_id | INT | PK, AUTO_INCREMENT | 위치 고유 ID |
| user_id | INT | FK, NOT NULL | 사용자 ID |
| address | VARCHAR(500) | NOT NULL | 주소 |
| latitude | DECIMAL(10, 8) | | 위도 |
| longitude | DECIMAL(11, 8) | | 경도 |
| is_default | BOOLEAN | DEFAULT FALSE | 기본 위치 여부 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 생성일시 |
| updated_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 수정일시 |

**인덱스:**
- PRIMARY KEY (location_id)
- KEY idx_user_id (user_id)
- KEY idx_is_default (is_default)
- FOREIGN KEY fk_user_locations_user_id (user_id) REFERENCES USERS(user_id)

### 3.2 지도자 관련 테이블

#### 3.2.1 INSTRUCTORS (지도자)
| 컬럼명 | 데이터타입 | 제약조건 | 설명 |
|--------|------------|----------|------|
| instructor_id | INT | PK, AUTO_INCREMENT | 지도자 고유 ID |
| user_id | INT | FK, NOT NULL | 사용자 ID |
| business_number | VARCHAR(20) | | 개인 사업자 번호 |
| certification | VARCHAR(500) | | 자격증 정보 |
| experience | TEXT | | 경력 설명 |
| specializations | JSON | | 전문 분야 |
| business_license_url | VARCHAR(500) | | 사업자 등록증 URL |
| profile_image_url | VARCHAR(500) | | 프로필 이미지 URL |
| bio | TEXT | | 한줄소개 |
| portfolio_images | JSON | | 활동 사진들 |
| sns_accounts | JSON | | SNS 계정 정보 |
| hourly_rate | DECIMAL(10, 2) | | 시간당 수업료 |
| status | ENUM('PENDING', 'APPROVED', 'REJECTED', 'SUSPENDED') | DEFAULT 'PENDING' | 승인 상태 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 생성일시 |
| updated_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 수정일시 |

**인덱스:**
- PRIMARY KEY (instructor_id)
- UNIQUE KEY uk_user_id (user_id)
- KEY idx_status (status)
- FOREIGN KEY fk_instructors_user_id (user_id) REFERENCES USERS(user_id)

#### 3.2.2 INSTRUCTOR_FACILITIES (지도자 소속 시설)
| 컬럼명 | 데이터타입 | 제약조건 | 설명 |
|--------|------------|----------|------|
| instructor_facility_id | INT | PK, AUTO_INCREMENT | 지도자 시설 고유 ID |
| instructor_id | INT | FK, NOT NULL | 지도자 ID |
| facility_name | VARCHAR(200) | NOT NULL | 시설명 |
| facility_address | VARCHAR(500) | NOT NULL | 시설 주소 |
| postal_code | VARCHAR(10) | | 우편번호 |
| latitude | DECIMAL(10, 8) | | 위도 |
| longitude | DECIMAL(11, 8) | | 경도 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 생성일시 |

**인덱스:**
- PRIMARY KEY (instructor_facility_id)
- KEY idx_instructor_id (instructor_id)
- FOREIGN KEY fk_instructor_facilities_instructor_id (instructor_id) REFERENCES INSTRUCTORS(instructor_id)

### 3.3 시설 운영자 관련 테이블

#### 3.3.1 FACILITY_OPERATORS (시설 운영자)
| 컬럼명 | 데이터타입 | 제약조건 | 설명 |
|--------|------------|----------|------|
| operator_id | INT | PK, AUTO_INCREMENT | 운영자 고유 ID |
| user_id | INT | FK, NOT NULL | 사용자 ID |
| representative_name | VARCHAR(100) | NOT NULL | 대표자명 |
| business_number | VARCHAR(20) | UNIQUE, NOT NULL | 사업자 번호 |
| operating_hours | JSON | | 요일별 운영시간 |
| holidays | VARCHAR(200) | | 휴관일 |
| business_license_url | VARCHAR(500) | | 사업자 등록증 URL |
| status | ENUM('PENDING', 'APPROVED', 'REJECTED', 'SUSPENDED') | DEFAULT 'PENDING' | 승인 상태 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 생성일시 |
| updated_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 수정일시 |

**인덱스:**
- PRIMARY KEY (operator_id)
- UNIQUE KEY uk_user_id (user_id)
- UNIQUE KEY uk_business_number (business_number)
- KEY idx_status (status)
- FOREIGN KEY fk_facility_operators_user_id (user_id) REFERENCES USERS(user_id)

### 3.4 시설 관련 테이블

#### 3.4.1 FACILITIES (시설)
| 컬럼명 | 데이터타입 | 제약조건 | 설명 |
|--------|------------|----------|------|
| facility_id | INT | PK, AUTO_INCREMENT | 시설 고유 ID |
| operator_id | INT | FK | 운영자 ID (시설운영자가 등록한 시설) |
| name | VARCHAR(200) | NOT NULL | 시설명 |
| description | TEXT | | 시설 설명 |
| address | VARCHAR(500) | NOT NULL | 주소 |
| postal_code | VARCHAR(10) | | 우편번호 |
| latitude | DECIMAL(10, 8) | | 위도 |
| longitude | DECIMAL(11, 8) | | 경도 |
| phone | VARCHAR(20) | | 전화번호 |
| email | VARCHAR(255) | | 이메일 |
| website | VARCHAR(500) | | 웹사이트 |
| facility_type | ENUM('GYM', 'POOL', 'TENNIS', 'FOOTBALL', 'BASKETBALL', 'MULTI') | NOT NULL | 시설 유형 |
| status | ENUM('ACTIVE', 'INACTIVE', 'PENDING', 'SUSPENDED') | DEFAULT 'PENDING' | 시설 상태 |
| amenities | JSON | | 편의시설 (JSON) |
| operating_hours | JSON | | 운영시간 (JSON) |
| average_rating | DECIMAL(3, 2) | DEFAULT 0.00 | 평균 평점 |
| review_count | INT | DEFAULT 0 | 리뷰 수 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 생성일시 |
| updated_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 수정일시 |

**인덱스:**
- PRIMARY KEY (facility_id)
- KEY idx_operator_id (operator_id)
- KEY idx_facility_type (facility_type)
- KEY idx_status (status)
- KEY idx_location (latitude, longitude)
- KEY idx_rating (average_rating)
- FOREIGN KEY fk_facilities_operator_id (operator_id) REFERENCES FACILITY_OPERATORS(operator_id)

#### 3.4.2 FACILITY_IMAGES (시설 이미지)
| 컬럼명 | 데이터타입 | 제약조건 | 설명 |
|--------|------------|----------|------|
| image_id | INT | PK, AUTO_INCREMENT | 이미지 고유 ID |
| facility_id | INT | FK, NOT NULL | 시설 ID |
| image_url | VARCHAR(500) | NOT NULL | 이미지 URL |
| alt_text | VARCHAR(200) | | 이미지 설명 |
| sort_order | INT | DEFAULT 0 | 정렬 순서 |
| image_type | ENUM('MAIN', 'GALLERY', 'FLOOR_PLAN') | DEFAULT 'GALLERY' | 이미지 유형 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 생성일시 |

**인덱스:**
- PRIMARY KEY (image_id)
- KEY idx_facility_id (facility_id)
- KEY idx_sort_order (sort_order)
- FOREIGN KEY fk_facility_images_facility_id (facility_id) REFERENCES FACILITIES(facility_id)

### 3.5 운동 종목 관련 테이블

#### 3.5.1 SPORTS (운동 종목)
| 컬럼명 | 데이터타입 | 제약조건 | 설명 |
|--------|------------|----------|------|
| sport_id | INT | PK, AUTO_INCREMENT | 운동 종목 고유 ID |
| name | VARCHAR(100) | NOT NULL | 운동 종목명 |
| description | TEXT | | 운동 종목 설명 |
| icon_url | VARCHAR(500) | | 아이콘 URL |
| category | ENUM('INDIVIDUAL', 'TEAM', 'AQUATIC', 'COMBAT', 'FITNESS', 'OUTDOOR') | NOT NULL | 카테고리 |
| is_active | BOOLEAN | DEFAULT TRUE | 활성 상태 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 생성일시 |

**인덱스:**
- PRIMARY KEY (sport_id)
- KEY idx_category (category)
- KEY idx_is_active (is_active)

#### 3.5.2 FACILITY_SPORTS (시설 운동 종목)
| 컬럼명 | 데이터타입 | 제약조건 | 설명 |
|--------|------------|----------|------|
| facility_sport_id | INT | PK, AUTO_INCREMENT | 시설 운동 종목 고유 ID |
| facility_id | INT | FK, NOT NULL | 시설 ID |
| sport_id | INT | FK, NOT NULL | 운동 종목 ID |
| price_per_hour | DECIMAL(10, 2) | | 시간당 가격 |
| description | TEXT | | 해당 운동에 대한 시설 설명 |
| is_available | BOOLEAN | DEFAULT TRUE | 이용 가능 여부 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 생성일시 |

**인덱스:**
- PRIMARY KEY (facility_sport_id)
- UNIQUE KEY uk_facility_sport (facility_id, sport_id)
- KEY idx_is_available (is_available)
- FOREIGN KEY fk_facility_sports_facility_id (facility_id) REFERENCES FACILITIES(facility_id)
- FOREIGN KEY fk_facility_sports_sport_id (sport_id) REFERENCES SPORTS(sport_id)


### 3.6 검색 관련 테이블

#### 3.6.1 SEARCH_HISTORY (검색 기록)
| 컬럼명 | 데이터타입 | 제약조건 | 설명 |
|--------|------------|----------|------|
| search_id | INT | PK, AUTO_INCREMENT | 검색 고유 ID |
| user_id | INT | FK, NOT NULL | 사용자 ID |
| keyword | VARCHAR(200) | NOT NULL | 검색 키워드 |
| search_type | ENUM('FACILITY', 'SPORT', 'LOCATION') | NOT NULL | 검색 유형 |
| filters | JSON | | 검색 필터 조건 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 검색일시 |

**인덱스:**
- PRIMARY KEY (search_id)
- KEY idx_user_id (user_id)
- KEY idx_keyword (keyword)
- KEY idx_search_type (search_type)
- FOREIGN KEY fk_search_history_user_id (user_id) REFERENCES USERS(user_id)

### 3.7 약관 관련 테이블

#### 3.7.1 TERMS_AGREEMENTS (약관 동의)
| 컬럼명 | 데이터타입 | 제약조건 | 설명 |
|--------|------------|----------|------|
| agreement_id | INT | PK, AUTO_INCREMENT | 동의 고유 ID |
| user_id | INT | FK, NOT NULL | 사용자 ID |
| terms_type | ENUM('SERVICE', 'PRIVACY', 'MARKETING', 'LOCATION') | NOT NULL | 약관 유형 |
| agreed | BOOLEAN | NOT NULL | 동의 여부 |
| version | VARCHAR(20) | NOT NULL | 약관 버전 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 동의일시 |

**인덱스:**
- PRIMARY KEY (agreement_id)
- UNIQUE KEY uk_user_terms (user_id, terms_type)
- KEY idx_terms_type (terms_type)
- FOREIGN KEY fk_terms_agreements_user_id (user_id) REFERENCES USERS(user_id)

### 3.8 인증 관련 테이블

#### 3.8.1 USER_SESSIONS (사용자 세션)
| 컬럼명 | 데이터타입 | 제약조건 | 설명 |
|--------|------------|----------|------|
| session_id | INT | PK, AUTO_INCREMENT | 세션 고유 ID |
| user_id | INT | FK, NOT NULL | 사용자 ID |
| access_token | VARCHAR(500) | NOT NULL | 액세스 토큰 |
| refresh_token | VARCHAR(500) | NOT NULL | 리프레시 토큰 |
| device_info | JSON | | 디바이스 정보 |
| ip_address | VARCHAR(45) | | IP 주소 |
| user_agent | VARCHAR(500) | | 사용자 에이전트 |
| expires_at | TIMESTAMP | NOT NULL | 토큰 만료 시간 |
| is_active | BOOLEAN | DEFAULT TRUE | 활성 상태 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 생성일시 |
| updated_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 수정일시 |

**인덱스:**
- PRIMARY KEY (session_id)
- UNIQUE KEY uk_access_token (access_token)
- UNIQUE KEY uk_refresh_token (refresh_token)
- KEY idx_user_id (user_id)
- KEY idx_expires_at (expires_at)
- KEY idx_is_active (is_active)
- FOREIGN KEY fk_user_sessions_user_id (user_id) REFERENCES USERS(user_id)

#### 3.8.2 PHONE_VERIFICATIONS (전화번호 인증)
| 컬럼명 | 데이터타입 | 제약조건 | 설명 |
|--------|------------|----------|------|
| verification_id | INT | PK, AUTO_INCREMENT | 인증 고유 ID |
| phone | VARCHAR(20) | NOT NULL | 전화번호 |
| verification_code | VARCHAR(10) | NOT NULL | 인증번호 |
| verification_type | ENUM('SIGNUP', 'PASSWORD_RESET', 'PHONE_CHANGE') | NOT NULL | 인증 유형 |
| purpose_id | VARCHAR(100) | | 목적 ID (비밀번호 찾기 등) |
| attempts | INT | DEFAULT 0 | 시도 횟수 |
| is_verified | BOOLEAN | DEFAULT FALSE | 인증 완료 여부 |
| expires_at | TIMESTAMP | NOT NULL | 만료 시간 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 생성일시 |
| verified_at | TIMESTAMP | | 인증 완료 시간 |

**인덱스:**
- PRIMARY KEY (verification_id)
- KEY idx_phone (phone)
- KEY idx_verification_code (verification_code)
- KEY idx_expires_at (expires_at)
- KEY idx_is_verified (is_verified)

#### 3.8.3 KAKAO_USERS (카카오 사용자)
| 컬럼명 | 데이터타입 | 제약조건 | 설명 |
|--------|------------|----------|------|
| kakao_user_id | INT | PK, AUTO_INCREMENT | 카카오 사용자 고유 ID |
| user_id | INT | FK, NOT NULL | 사용자 ID |
| kakao_id | BIGINT | UNIQUE, NOT NULL | 카카오 ID |
| kakao_email | VARCHAR(255) | | 카카오 이메일 |
| kakao_nickname | VARCHAR(100) | | 카카오 닉네임 |
| kakao_profile_image | VARCHAR(500) | | 카카오 프로필 이미지 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 생성일시 |
| updated_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 수정일시 |

**인덱스:**
- PRIMARY KEY (kakao_user_id)
- UNIQUE KEY uk_kakao_id (kakao_id)
- UNIQUE KEY uk_user_id (user_id)
- FOREIGN KEY fk_kakao_users_user_id (user_id) REFERENCES USERS(user_id)

### 3.9 시스템 설정 테이블

#### 3.9.1 SYSTEM_SETTINGS (시스템 설정)
| 컬럼명 | 데이터타입 | 제약조건 | 설명 |
|--------|------------|----------|------|
| setting_id | INT | PK, AUTO_INCREMENT | 설정 고유 ID |
| setting_key | VARCHAR(100) | UNIQUE, NOT NULL | 설정 키 |
| setting_value | TEXT | NOT NULL | 설정 값 |
| setting_type | ENUM('STRING', 'INTEGER', 'BOOLEAN', 'JSON') | NOT NULL | 설정 타입 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP | 생성일시 |
| updated_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 수정일시 |

**인덱스:**
- PRIMARY KEY (setting_id)
- UNIQUE KEY uk_setting_key (setting_key)
- KEY idx_setting_type (setting_type)

## 4. API 연동 설계

### 4.1 인증 API 연동
- **JWT 토큰 기반 인증**: USER_SESSIONS 테이블과 연동
- **카카오 소셜 로그인**: KAKAO_USERS 테이블로 연동
- **전화번호 인증**: PHONE_VERIFICATIONS 테이블 활용
- **비밀번호 재설정**: 이름+전화번호 인증 플로우

### 4.2 사용자 관리 API 연동
- **프로필 관리**: USERS, USER_PROFILES 테이블
- **계정 정보 변경**: 전화번호/이메일 변경 시 인증 필요
- **회원 탈퇴**: 소프트 삭제 방식 (status = 'DELETED')

### 4.3 시설 관련 API 연동
- **시설 검색**: FACILITIES, FACILITY_IMAGES, FACILITY_SPORTS 조인
- **위치 기반 검색**: latitude, longitude 인덱스 활용
- **운동 종목 필터링**: SPORTS, FACILITY_SPORTS 테이블
- **지도자 연결**: INSTRUCTOR_FACILITIES를 통한 지도자-시설 매칭
- **시설 상세**: 시설운영자 정보 + 지도자 정보 통합 조회

## 5. 1차 개발 테이블 목록

### 5.1 핵심 테이블 (16개)
1. **USERS** - 사용자 기본 정보
2. **USER_PROFILES** - 사용자 프로필
3. **USER_LOCATIONS** - 사용자 위치 정보
4. **INSTRUCTORS** - 지도자 정보
5. **INSTRUCTOR_FACILITIES** - 지도자 소속 시설
6. **FACILITY_OPERATORS** - 시설 운영자 정보
7. **FACILITIES** - 시설 정보
8. **FACILITY_IMAGES** - 시설 이미지
9. **SPORTS** - 운동 종목
10. **FACILITY_SPORTS** - 시설별 운동 종목
11. **SEARCH_HISTORY** - 검색 기록
12. **TERMS_AGREEMENTS** - 약관 동의
13. **USER_SESSIONS** - 사용자 세션
14. **PHONE_VERIFICATIONS** - 전화번호 인증
15. **KAKAO_USERS** - 카카오 사용자
16. **SYSTEM_SETTINGS** - 시스템 설정

---

*작성일: 2025년 10월*  
*버전: 2.0*  
*작성자: 김진석*  
*기반 문서: IA 명세서, 기능 정의서, API 명세서*  
*개발 범위: 1차 개발 (MVP) - 예약/리뷰/찜/문의/알림/커뮤니티 기능 제외*  
