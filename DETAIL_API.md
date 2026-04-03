# TravelBusan API 명세서 (API Specification)
본 명세서는 TravelBusan 모바일 애플리케이션의 원활한 백엔드 통신을 위해 설계되었습니다. [cite_start]모든 응답은 `application/json` 포맷을 사용하며, 인증이 필요한 API는 `Authorization: Bearer {JWT_TOKEN}` 헤더를 요구합니다.

베이스 URL : /api/v1

---
## 1. 인증 및 사용자 (Auth & Users)
### 1.1. 일반 로그인
* **1. [cite_start]API 설명**: 이메일과 비밀번호를 사용하여 로그인을 수행하고 JWT 토큰을 반환합니다[cite: 57, 59]. [cite_start]존재하지 않는 계정이거나 비밀번호가 틀릴 경우 실패 횟수를 기록하며, 최대 5번까지만 로그인 시도가 가능하도록 제한합니다[cite: 58, 304].
* **2. 엔드포인트**: `POST /api/v1/auth/login`
* **3. 송수신 JSON 양식**:
    * **Request**:
        ```json
        {
          "email": "example@email.com",
          "password": "user_password_here"
        }
        ```
    * **Response (Success)**:
        ```json
        {
          "status": "success",
          "data": {
            "user_id": "uuid-string",
            "access_token": "jwt_access_token_string",
            "nickname": "김여행",
            "is_guide": false
          }
        }
        ```
    * **Response (Fail - Exceeds 5 attempts)**:
        ```json
        {
          "status": "error",
          "message": "로그인 시도 횟수를 초과했습니다. 관리자에게 문의하세요.",
          "failed_attempts": 5
        }
        ```
### 1.2. 로컬 가이드 등록
* **1. [cite_start]API 설명**: 일반 사용자가 마이페이지에서 로컬 가이드 서비스 제공자로 등록합니다[cite: 30, 296]. [cite_start]`users` 테이블의 `is_guide` 상태를 업데이트합니다[cite: 300, 304].
* **2. 엔드포인트**: `POST /api/v1/users/guide-register`
* **3. 송수신 JSON 양식**:
    * **Request**:
        ```json
        {
          "introduction": "제주에서 15년 살고 있는 토박이 가이드입니다."
        }
        ```
    * **Response**:
        ```json
        {
          "status": "success",
          "message": "로컬 가이드 등록이 완료되었습니다."
        }
        ```
---
## 2. 여행지 및 메인 홈 (Destinations)
### 2.1. 여행지 목록 조회 (추천 및 인기)
* **1. [cite_start]API 설명**: 메인 화면에서 추천 여행지(최고 평점)와 인기 여행지를 2열xN행 형태로 출력하기 위해 조회합니다[cite: 87, 88]. [cite_start]`destinations` 테이블의 데이터를 반환합니다[cite: 306, 308].
* **2. 엔드포인트**: `GET /api/v1/destinations?type=popular&page=1&size=10`
* **3. 송수신 JSON 양식**:
    * **Request**: (Query Parameters)
    * **Response**:
        ```json
        {
          "status": "success",
          "data": {
            "destinations": [
              {
                "destination_id": 1,
                "name": "제주도",
                "region": "제주특별자치도",
                "category": ["자연", "해변", "맛집"],
                "rating": 4.8,
                "image_url": "https://s3.../jeju.jpg"
              }
            ]
          }
        }
        ```
---
## 3. AI 플래너 (AI Planner)
### 3.1. AI 일정 생성 요청
* **1. [cite_start]API 설명**: 사용자가 여행지, 일정, 관심 카테고리(레저, 맛집 등)를 LLM 모델에 전달하여 맞춤형 여행 일정을 생성합니다[cite: 28, 129].
* **2. 엔드포인트**: `POST /api/v1/planner/generate`
* **3. 송수신 JSON 양식**:
    * **Request**:
        ```json
        {
          "prompt": "제주도 2박3일 자연 힐링 여행 만들어줘",
          "categories": ["자연", "힐링", "사진명소"]
        }
        ```
    * **Response**:
        ```json
        {
          "status": "success",
          "data" : {
              "title": "여행 제목",
              "region": "부산광역시",
              "start_date": "2026-04-05",
              "end_date": "2026-04-07",
              "generated_courses": [
                {
                  "day_number": 1,
                  "start_time": "09:00",
                  "duration_minutes": 120,
                  "place": "태종대",
                  "latitude": 35.0531,
                  "longitude": 129.0878,
                  "category_type": ["자연", "명소"],
                  "operating_hours": "09:00 - 18:00 (월요일 휴무)",
                  "description": "아름다운 자연경관과 해안을 즐길 수 있는 산책 코스입니다."
                }
              ]
            }
        }
        ```
### 3.2. AI 생성 일정 저장
* **1. [cite_start]API 설명**: AI가 생성한 일정이 마음에 들 경우 '플래너 저장' 버튼을 눌러 DB(`itineraries`, `itinerary_details` 테이블)에 저장합니다[cite: 130, 131, 311, 315].
* **2. 엔드포인트**: `POST /api/v1/planner/itineraries`
* **3. 송수신 JSON 양식**:
    * **Request**: (3.1의 Response data 전체를 포함하여 전송)
        ```json
        {
          "title": "제주도 2박 3일 자연 힐링 여행",
          "region": "제주특별자치도",
          "start_date": "2026-03-20",
          "end_date": "2026-03-22",
          "total_cost": 500000,
          "courses": [
            {
               "day_number": 1,
               "start_time": "09:00",
               "duration_minutes": 120,
               "category_type": "자연",
               "description": "비자림 숨은 산책로 트레킹",
               "destination_id": 45
            }
          ]
        }
        ```
    * **Response**:
        ```json
        {
          "status": "success",
          "message": "일정이 성공적으로 저장되었습니다.",
          "itinerary_id": 101
        }
        ```
### 3.3. 상세 일정 조회
* **1. [cite_start]API 설명**: 저장된 플래너의 날짜별 방문 장소, 소요 시간, 카테고리 등 상세 정보를 일차별, 시간순으로 조회합니다[cite: 31, 191]. [cite_start]`itinerary_details` 테이블을 참조합니다[cite: 315, 318].
* **2. 엔드포인트**: `GET /api/v1/planner/itineraries/{itinerary_id}`
* **3. 송수신 JSON 양식**:
    * **Request**: (None)
    * **Response**:
        ```json
        {
          "status": "success",
          "data": {
            "itinerary_id": 101,
            "title": "부산 1박 2일 맛집 투어",
            "courses": [
              {
                "detail_id": 501,
                "day_number": 1,
                "start_time": "08:00",
                "duration_minutes": 90,
                "category_type": "맛집",
                "description": "자갈치시장 회 조식"
              }
            ]
          }
        }
        ```
---
## 4. 로컬 가이드 및 예약 (Local Guide & Reservation)
### 4.1. 가이드 투어 목록 탐색
* **1. [cite_start]API 설명**: 관광객들이 지역별(제주도, 부산, 서울 등)로 등록된 가이드 투어 상품(`guide_tours`)을 검색하고 목록을 확인합니다[cite: 201, 265, 324].
* **2. 엔드포인트**: `GET /api/v1/guides/tours?region=제주도`
* **3. 송수신 JSON 양식**:
    * **Request**: (Query Parameters)
    * **Response**:
        ```json
        {
          "status": "success",
          "data": {
            "tours": [
              {
                "tour_id": 10,
                "guide_id": "uuid-guide-1",
                "title": "제주 숨은 맛집 & 자연 투어",
                "duration_hours": 6,
                "price_per_person": 45000,
                "max_participants": 6
              }
            ]
          }
        }
        ```
### 4.2. 투어 예약 및 결제 생성
* **1. [cite_start]API 설명**: 원하는 가이드 투어에 대해 '바로 예약'을 진행하여 포트원 API 연동 전 자체 DB(`reservations`)에 결제 대기 상태의 예약 레코드를 생성합니다[cite: 268, 329].
* **2. 엔드포인트**: `POST /api/v1/reservations`
* **3. 송수신 JSON 양식**:
    * **Request**:
        ```json
        {
          "tour_id": 10,
          "participants_count": 2,
          "total_amount": 90000
        }
        ```
    * **Response**:
        ```json
        {
          "status": "success",
          "data": {
            "reservation_id": "uuid-reservation-1",
            "merchant_uid": "merchant_123456789",
            "amount": 90000
          }
        }
        ```
---
## 5. 소통 (Chat)
### 5.1. 1:1 대화방 개설
* **1. [cite_start]API 설명**: 가이드와 관광객 간의 1:1 대화하기 버튼 클릭 시 대화방(`chat_rooms`)을 개설합니다[cite: 268, 335, 337].
* **2. 엔드포인트**: `POST /api/v1/chat/rooms`
* **3. 송수신 JSON 양식**:
    * **Request**:
        ```json
        {
          "tour_id": 10,
          "guide_id": "uuid-guide-1"
        }
        ```
    * **Response**:
        ```json
        {
          "status": "success",
          "data": {
            "room_id": 999
          }
        }
        ```
