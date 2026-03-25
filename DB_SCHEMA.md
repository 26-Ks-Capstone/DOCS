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

-- 3. AI 여행 플래너 마스터 (itineraries)
CREATE TABLE itineraries (
    itinerary_id BIGSERIAL PRIMARY KEY,                 -- [cite: 16]
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE, -- [cite: 17]
    title VARCHAR(255) NOT NULL,                        -- [cite: 18]
    region VARCHAR(100),                                -- [cite: 18]
    start_date DATE,                                    -- [cite: 18]
    end_date DATE,                                      -- [cite: 18]
    total_cost BIGINT DEFAULT 0,                        -- [cite: 18]
    course_count INT DEFAULT 0                          -- [cite: 18]
);

CREATE INDEX idx_itineraries_user_id ON itineraries(user_id);

-- 4. AI 여행 플래너 상세 코스 (itinerary_details)
CREATE TABLE itinerary_details (
    detail_id BIGSERIAL PRIMARY KEY,                    -- [cite: 20]
    itinerary_id BIGINT REFERENCES itineraries(itinerary_id) ON DELETE CASCADE, -- [cite: 21]
    day_number INT NOT NULL,                            -- [cite: 22]
    start_time TIME,                                    -- [cite: 22]
    duration_minutes INT,                               -- [cite: 22]
    category_type VARCHAR(50),                          -- [cite: 22]
    destination_id BIGINT REFERENCES destinations(destination_id) ON DELETE SET NULL, -- Nullable [cite: 23]
    description TEXT,                                   -- [cite: 24]
    sort_order INT NOT NULL,                            -- [cite: 24]
    location GEOMETRY(Point, 4326)                      -- [GIS] 커스텀 장소 공간 데이터 (Nullable) [cite: 25, 26]
);

CREATE INDEX idx_itinerary_details_itinerary_id ON itinerary_details(itinerary_id);
CREATE INDEX idx_itinerary_details_location ON itinerary_details USING GIST (location);

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
