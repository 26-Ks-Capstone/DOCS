PostgreSQL

-- PostGIS 확장 활성화 (GIS 데이터 처리를 위해 필수)
CREATE EXTENSION IF NOT EXISTS postgis;

-- 1. 사용자 및 인증 (users)
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(), -- [cite: 5]
    email VARCHAR(255) UNIQUE NOT NULL,                 -- UNIQUE 제약조건으로 인해 자동 인덱스 생성됨 [cite: 6]
    social_provider VARCHAR(50),                        -- [cite: 7]
    nickname VARCHAR(100) NOT NULL,                     -- [cite: 7]
    profile_image_url TEXT,                             -- [cite: 7]
    is_guide BOOLEAN DEFAULT FALSE,                     -- [cite: 8]
    failed_login_attempts INT DEFAULT 0                 -- [cite: 8]
);

-- 2. 여행지 기초 데이터 (destinations)
CREATE TABLE destinations (
    destination_id BIGSERIAL PRIMARY KEY,               -- [cite: 11]
    name VARCHAR(255) NOT NULL,                         -- [cite: 12]
    region VARCHAR(100),                                -- [cite: 12]
    category VARCHAR(100),                              -- [cite: 12]
    rating NUMERIC(3, 2) DEFAULT 0.0,                   -- [cite: 12]
    image_url TEXT,                                     -- [cite: 12]
    location GEOMETRY(Point, 4326)                      -- [GIS] 위경도 공간 데이터 [cite: 13]
);

-- 공간 데이터(GIS) 조회를 위한 GIST 인덱스
CREATE INDEX idx_destinations_location ON destinations USING GIST (location);

-- 1. 상위 여행 일정 테이블 (참고용)
CREATE TABLE itineraries (
                             itinerary_id BIGSERIAL PRIMARY KEY,
                             title VARCHAR(255) NOT NULL,       -- "부모님과 함께하는 1박 2일..."
                             region VARCHAR(100),               -- "부산광역시"
                             start_date DATE,                   -- "2026-04-06"
                             end_date DATE,                     -- "2026-04-07"
                             created_at TIMESTAMP DEFAULT NOW()
);

-- PostGIS 확장 프로그램 활성화
CREATE EXTENSION postgis;

-- 2. AI 여행 플래너 상세 코스 (itinerary_details)
CREATE TABLE itinerary_details (
                                   detail_id BIGSERIAL PRIMARY KEY,
                                   user_id uuid REFERENCES users(user_id),
                                   itinerary_id BIGINT REFERENCES itineraries(itinerary_id) ON DELETE CASCADE,
                                   day_number INT NOT NULL,
                                   start_time TIME,
                                   duration_minutes INT,
                                   place_name VARCHAR(255) NOT NULL,
                                   category_type TEXT[], -- 배열 타입
                                   operating_hours VARCHAR(255),
                                   description TEXT,
                                   place_id BIGINT REFERENCES travel_places(place_id) ON DELETE SET NULL,
                                   sort_order INT NOT NULL,
                                   latitude DOUBLE PRECISION,  -- 위도 (예: 35.157...)
                                   longitude DOUBLE PRECISION  -- 경도 (예: 129.182...)
);
CREATE INDEX idx_itinerary_details_itinerary_id ON itinerary_details(itinerary_id);

-- 5. 로컬 가이드 투어 상품 (guide_tours)
CREATE TABLE guide_tours (
    tour_id BIGSERIAL PRIMARY KEY,                      -- [cite: 29]
    guide_id UUID REFERENCES users(user_id) ON DELETE CASCADE, -- [cite: 30]
    title VARCHAR(255) NOT NULL,                        -- [cite: 31]
    description TEXT,                                   -- [cite: 31]
    duration_hours NUMERIC(4, 1),                       -- [cite: 31]
    price_per_person BIGINT NOT NULL,                   -- [cite: 31]
    max_participants INT NOT NULL,                      -- [cite: 32]
    included_items JSONB                                -- [cite: 32]
);

CREATE INDEX idx_guide_tours_guide_id ON guide_tours(guide_id);

-- 6. 예약 및 결제 (reservations)
CREATE TABLE reservations (
    reservation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(), -- [cite: 34]
    tour_id BIGINT REFERENCES guide_tours(tour_id) ON DELETE CASCADE, -- [cite: 35]
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE, -- [cite: 35]
    status VARCHAR(50) NOT NULL,                        -- [cite: 36]
    payment_id VARCHAR(100),                            -- [cite: 36]
    total_amount BIGINT NOT NULL                        -- [cite: 36]
);

CREATE INDEX idx_reservations_tour_id ON reservations(tour_id);
CREATE INDEX idx_reservations_user_id ON reservations(user_id);

-- 7. 1:1 대화방 메타데이터 (chat_rooms)

CREATE TABLE chat_rooms (
    room_id BIGSERIAL PRIMARY KEY,                      -- [cite: 40]
    tour_id BIGINT REFERENCES guide_tours(tour_id) ON DELETE CASCADE, -- [cite: 41]
    tourist_id UUID REFERENCES users(user_id) ON DELETE CASCADE, -- [cite: 42]
    guide_id UUID REFERENCES users(user_id) ON DELETE CASCADE  -- [cite: 43]
);

CREATE INDEX idx_chat_rooms_tour_id ON chat_rooms(tour_id);
CREATE INDEX idx_chat_rooms_users ON chat_rooms(tourist_id, guide_id);

-- 8. 실제 대화 내역 (chat_messages)
CREATE TABLE chat_messages (
    message_id BIGSERIAL PRIMARY KEY,                   -- [cite: 45]
    room_id BIGINT REFERENCES chat_rooms(room_id) ON DELETE CASCADE, -- [cite: 46]
    sender_id UUID REFERENCES users(user_id) ON DELETE CASCADE, -- [cite: 47]
    message TEXT NOT NULL,                              -- [cite: 48]
    is_read BOOLEAN DEFAULT FALSE,                      -- [cite: 49]
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP      -- 
);

CREATE INDEX idx_chat_messages_room_id ON chat_messages(room_id);
-- 최신 메시지 조회를 위해 created_at에 인덱스 추가 (설계도 [IDX] 명시) 
CREATE INDEX idx_chat_messages_created_at ON chat_messages(created_at DESC); 

-- 9. 투어 리뷰 (reviews)
CREATE TABLE reviews (
    review_id BIGSERIAL PRIMARY KEY,                    -- [cite: 52]
    tour_id BIGINT REFERENCES guide_tours(tour_id) ON DELETE CASCADE, -- [cite: 53]
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE, -- [cite: 54]
    rating INT CHECK (rating >= 1 AND rating <= 5),     -- 별점 범위 제한 제약조건 [cite: 55]
    content TEXT                                        -- [cite: 55]
);

CREATE INDEX idx_reviews_tour_id ON reviews(tour_id);

--여행지 기본 정보 (지도 핀 및 목록 출력용)
CREATE TABLE travel_places (
                               place_id INT PRIMARY KEY,              -- contentid
                               type_id INT,                           -- contenttypeid
                               title VARCHAR(255) NOT NULL,           -- title
                               addr1 VARCHAR(500),                    -- addr1
                               addr2 VARCHAR(255),                    -- addr2
                               zipcode VARCHAR(20),                   -- zipcode
                               location POINT,                        -- mapx, mapy를 하나로 묶은 좌표 (경도, 위도)
                               mlevel INT,                            -- mlevel
                               first_image TEXT,                      -- firstimage
                               first_image2 TEXT                      -- firstimage2
);

--여행지 상세 설명 (개요 및 외부 링크)
CREATE TABLE travel_descriptions (
                                     place_id INT PRIMARY KEY REFERENCES travel_places(place_id),
                                     homepage TEXT,                         -- homepage (HTML 태그 포함 원문)
                                     overview TEXT                          -- overview (한류 정보 포함 전체 설명)
);

--여행지 운영 및 요금 정보 (이용 조건 전용)
CREATE TABLE travel_fees (
                             place_id INT PRIMARY KEY REFERENCES travel_places(place_id),
                             use_fee TEXT,                          -- 이용 요금 안내 (예: 정보 없음, 무료 등)
                             use_time TEXT                          -- 이용 시간 (예: 매년 6월~8월)
);

-- pgvector 확장 활성화
CREATE EXTENSION IF NOT EXISTS vector;

-- RAG 검색용 벡터 테이블
CREATE TABLE travel_vectors (
    vector_id SERIAL PRIMARY KEY,
    place_id INT REFERENCES travel_places(place_id),
    content_chunk TEXT,                    -- 검색 대상 텍스트 (명칭 + 주소 + 요금 + 개요 합본)
    embedding VECTOR(768)                  -- Gemini text-embedding-004 모델 기준
);

-- 코사인 유사도 검색을 위한 인덱스 (속도 향상)
CREATE INDEX ON travel_vectors USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
