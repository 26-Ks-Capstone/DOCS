PostgreSQL

/***********************************************************
 * 1. 사용자 및 권한 관리 (Users)
 ***********************************************************/
CREATE TABLE public.users (
    user_id              uuid PRIMARY KEY DEFAULT gen_random_uuid(), -- 고유 식별자 (UUID)
    email                character varying(255) NOT NULL,            -- 로그인 이메일 (Unique Index)
    password             text NOT NULL,                              -- 암호화된 비밀번호
    social_provider      character varying(50),                      -- 소셜 로그인 제공자 (google, kakao 등)
    nickname             character varying(100) NOT NULL,            -- 서비스 내 닉네임
    profile_image_url    text,                                       -- 프로필 이미지 주소
    is_guide             boolean DEFAULT false,                      -- 가이드 권한 여부
    failed_login_attempts integer DEFAULT 0                           -- 로그인 실패 횟수 (보안)
);
CREATE UNIQUE INDEX users_email_key ON users USING btree (email);


/***********************************************************
 * 2. 여행지 마스터 데이터 (Travel Places & Details)
 ***********************************************************/
-- [장소 기본 정보]
CREATE TABLE public.travel_places (
    place_id    integer PRIMARY KEY NOT NULL,          -- 장소 고유 번호
    type_id     integer,                               -- 관광 타입 ID (문화시설, 축제 등)
    title       character varying(255) NOT NULL,       -- 장소명
    addr1       character varying(500),                -- 기본 주소
    addr2       character varying(255),                -- 상세 주소
    zipcode     character varying(20),                 -- 우편번호
    location    point,                                 -- 좌표 (PostgreSQL Point 타입: 위도, 경도)
    mlevel      integer,                               -- 지도 확대 레벨
    first_image text,                                  -- 대표 이미지 원본
    first_image2 text,                                 -- 대표 이미지 썸네일
    cat1        character varying(100),                -- 대분류
    cat2        character varying(100),                -- 중분류
    cat3        character varying(100)                 -- 소분류
);

-- [장소 상세 설명]
CREATE TABLE public.travel_descriptions (
    place_id    integer PRIMARY KEY REFERENCES public.travel_places(place_id),
    homepage    text,                                  -- 홈페이지 URL
    overview    text                                   -- 장소 개요/설명 문구
);

-- [장소 이용 요금 및 시간]
CREATE TABLE public.travel_fees (
    place_id    integer PRIMARY KEY REFERENCES public.travel_places(place_id),
    use_fee     text,                                  -- 이용 요금 안내
    use_time    text                                   -- 이용 시간/영업 시간 안내
);


/***********************************************************
 * 3. AI 추천 데이터 (Vectors)
 ***********************************************************/
-- [AI 벡터 저장소]
CREATE TABLE public.travel_vectors (
    vector_id    integer PRIMARY KEY DEFAULT nextval('travel_vectors_vector_id_seq'::regclass),
    place_id     integer REFERENCES public.travel_places(place_id), -- 연결된 장소
    content_chunk text,                                             -- 임베딩에 사용된 원문 텍스트
    embedding    vector(768)                                        -- AI 생성 벡터값 (768차원)
);
-- 검색 최적화를 위한 인덱스들
CREATE UNIQUE INDEX unique_place_id ON travel_vectors USING btree (place_id);
CREATE INDEX idx_travel_vectors_embedding_hnsw ON travel_vectors USING hnsw (embedding vector_cosine_ops);


/***********************************************************
 * 4. 사용자 일정 관리 (Itineraries)
 ***********************************************************/
-- [여행 계획 메인]
CREATE TABLE public.itineraries (
    itinerary_id bigint PRIMARY KEY DEFAULT nextval('itineraries_itinerary_id_seq'::regclass),
    user_id      uuid NOT NULL REFERENCES public.users(user_id) ON DELETE CASCADE, -- 작성자
    title        character varying(255) NOT NULL,       -- 일정 제목
    region       character varying(100),                -- 대상 지역
    start_date   date,                                  -- 여행 시작일
    end_date     date,                                  -- 여행 종료일
    created_at   timestamp without time zone DEFAULT now() -- 생성 일시
);

-- [일정 내 세부 방문지]
CREATE TABLE public.itinerary_details (
    detail_id        bigint PRIMARY KEY DEFAULT nextval('itinerary_details_detail_id_seq'::regclass),
    itinerary_id     bigint REFERENCES public.itineraries(itinerary_id) ON DELETE CASCADE,
    place_id         bigint REFERENCES public.travel_places(place_id) ON DELETE SET NULL,
    day_number       integer NOT NULL,                  -- 여행 몇 일차인가?
    sort_order       integer NOT NULL,                  -- 해당 일차 내 방문 순서
    start_time       time without time zone,            -- 방문 예정 시간
    duration_minutes integer,                           -- 예상 체류 시간
    place_name       character varying(255) NOT NULL,   -- 장소명 (장소 DB에 없어도 수동 입력 가능)
    category_type    text[],                            -- 카테고리 태그들
    latitude         double precision,                  -- 지도 표시용 위도
    longitude        double precision,                  -- 지도 표시용 경도
    description      text                               -- 사용자 메모/설명
);
