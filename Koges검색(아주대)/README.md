1. 전체 시스템 아키텍처
1.1 Phase 1 아키텍처 (MySQL 기반)
mermaidgraph TB
    subgraph "사용자 레이어"
        User[연구자]
    end
    
    subgraph "프론트엔드 레이어"
        UI[Web UI<br/>HTML/CSS/JavaScript]
        SearchInput[검색 입력창]
        Autocomplete[자동완성 UI]
        SearchResult[검색 결과 표시]
    end
    
    subgraph "Flask API Server"
        QueryProcessor[검색 쿼리 프로세서<br/>• 텍스트 정규화<br/>• 형태소 분석 Mecab-ko<br/>• 동의어 확장<br/>• 초성 검색 판단]
        SearchEngine[MySQL 검색 엔진<br/>• 정확 매칭<br/>• 동의어 매칭<br/>• 초성 검색<br/>• 부분 매칭<br/>• FULLTEXT 검색]
        Ranking[랭킹 알고리즘<br/>Final Score = Base Score<br/>+ Popularity + Category]
        TrieAC[Trie 자동완성<br/>• 접두사 매칭<br/>• 초성 자동완성<br/>• 인기 검색어]
        Recommend[연관 변수 추천<br/>• 도메인 지식<br/>• 검색 컨텍스트]
    end
    
    subgraph "MySQL Database"
        Variables[(variables<br/>변수 마스터)]
        Synonyms[(synonyms<br/>동의어 사전)]
        Abbreviations[(abbreviations<br/>약어 사전)]
        MedicalTerms[(medical_terms<br/>의학 용어)]
        InitialMappings[(initial_mappings<br/>초성 매핑)]
        SpacingCorrections[(spacing_corrections<br/>띄어쓰기 교정)]
        SearchLogs[(search_logs<br/>검색 로그)]
        VarRelationships[(variable_relationships<br/>변수 연관관계)]
    end
    
    User --> UI
    UI --> SearchInput
    UI --> Autocomplete
    UI --> SearchResult
    
    SearchInput --> QueryProcessor
    QueryProcessor --> SearchEngine
    SearchEngine --> Ranking
    Ranking --> SearchResult
    
    Autocomplete --> TrieAC
    SearchResult --> Recommend
    
    QueryProcessor --> Synonyms
    QueryProcessor --> SpacingCorrections
    SearchEngine --> Variables
    SearchEngine --> InitialMappings
    Ranking --> SearchLogs
    TrieAC --> Variables
    TrieAC --> SearchLogs
    Recommend --> VarRelationships
    Recommend --> SearchLogs
    
    style User fill:#e1f5ff
    style UI fill:#fff4e1
    style QueryProcessor fill:#e8f5e9
    style SearchEngine fill:#e8f5e9
    style Ranking fill:#e8f5e9
    style Variables fill:#f3e5f5
1.2 Phase 2 아키텍처 (Elasticsearch + FAISS)
mermaidgraph TB
    subgraph "사용자 레이어"
        User[연구자]
    end
    
    subgraph "프론트엔드"
        UI[Web UI]
    end
    
    subgraph "Flask API Server"
        QueryProcessor[검색 쿼리 프로세서]
        HybridSearch[하이브리드 검색 엔진]
        ESSearch[Elasticsearch<br/>키워드 검색]
        FAISSSearch[FAISS<br/>벡터 검색]
        ResultMerge[결과 통합 및 정규화<br/>ES 0.6 + FAISS 0.4]
        HybridRanking[하이브리드 랭킹<br/>ES×0.6 + FAISS×0.4×2.0<br/>+ Popularity×0.8<br/>+ Recency×0.6<br/>+ Category×1.1]
    end
    
    subgraph "캐싱 레이어"
        Redis[(Redis Cache<br/>검색 결과: 1시간<br/>자동완성: 24시간<br/>변수 정보: 6시간)]
    end
    
    subgraph "데이터 레이어"
        MySQL[(MySQL 8.0<br/>Source of Truth)]
        
        subgraph "Elasticsearch Cluster"
            ES1[Node 1]
            ES2[Node 2]
            ES3[Node 3]
        end
        
        FAISS[(FAISS Index<br/>paraphrase-multilingual<br/>384차원 벡터<br/>IndexFlatIP)]
    end
    
    User --> UI
    UI --> QueryProcessor
    QueryProcessor --> HybridSearch
    
    HybridSearch --> ESSearch
    HybridSearch --> FAISSSearch
    
    ESSearch --> ES1
    ESSearch --> ES2
    ESSearch --> ES3
    
    FAISSSearch --> FAISS
    
    ESSearch --> ResultMerge
    FAISSSearch --> ResultMerge
    
    ResultMerge --> HybridRanking
    HybridRanking --> Redis
    HybridRanking --> MySQL
    
    MySQL -.동기화.-> ES1
    MySQL -.동기화.-> ES2
    MySQL -.동기화.-> ES3
    MySQL -.동기화.-> FAISS
    
    HybridRanking --> UI
    
    style User fill:#e1f5ff
    style UI fill:#fff4e1
    style HybridSearch fill:#e8f5e9
    style ResultMerge fill:#fff3e0
    style Redis fill:#ffebee
    style MySQL fill:#f3e5f5
2. 데이터 흐름 다이어그램
2.1 데이터 적재 흐름
mermaidflowchart TD
    Start[Excel 파일<br/>openkoges_codingbook.xls] --> Read[pandas로 파일 읽기<br/>17개 원본 컬럼]
    
    Read --> Preprocess[데이터 전처리 파이프라인]
    
    Preprocess --> Step1[1. 원본 17개 컬럼 읽기<br/>integrated_var_name, tablename, Label...]
    Step1 --> Step2[2. 검색 최적화 필드 생성]
    
    Step2 --> Sub2A[search_text:<br/>3개 필드 통합]
    Step2 --> Sub2B[search_tokens:<br/>Mecab-ko 형태소 분석]
    Step2 --> Sub2C[search_initials:<br/>jamotools 초성 추출]
    Step2 --> Sub2D[normalized_text:<br/>정규화]
    
    Sub2A --> Step3[3. 동의어/약어 자동 추출]
    Sub2B --> Step3
    Sub2C --> Step3
    Sub2D --> Step3
    
    Step3 --> Sub3A[Label 파싱]
    Step3 --> Sub3B[변수 코드 분해]
    
    Sub3A --> Step4[4. 초성 매핑 생성]
    Sub3B --> Step4
    
    Step4 --> BulkInsert[MySQL Bulk Insert<br/>1000개씩 배치]
    
    BulkInsert --> MySQLDB[(MySQL Database<br/>variables: 21개 컬럼<br/>synonyms: 자동+수동<br/>abbreviations: 자동<br/>initial_mappings: 자동)]
    
    MySQLDB --> Phase2{Phase 2 전환?}
    
    Phase2 -->|Yes| Sync[동기화 프로세스]
    Phase2 -->|No| End1[Phase 1 완료]
    
    Sync --> FullSync[전체 동기화<br/>최초 1회]
    Sync --> IncrSync[증분 동기화<br/>매일 새벽 2시]
    Sync --> RealtimeSync[실시간 동기화<br/>변수 추가/수정 시]
    
    FullSync --> ES[(Elasticsearch)]
    FullSync --> FAISS[(FAISS)]
    
    IncrSync --> ES
    IncrSync --> FAISS
    
    RealtimeSync --> ES
    RealtimeSync --> FAISS
    
    ES --> End2[Phase 2 완료]
    FAISS --> End2
    
    style Start fill:#e3f2fd
    style Preprocess fill:#fff3e0
    style MySQLDB fill:#f3e5f5
    style ES fill:#e8f5e9
    style FAISS fill:#fce4ec
3. 검색 시퀀스 다이어그램
3.1 Phase 1 - 기본 검색 시퀀스
mermaidsequenceDiagram
    actor User as 사용자
    participant UI as 프론트엔드
    participant API as Flask API
    participant Processor as 검색프로세서
    participant MySQL as MySQL
    participant Ranking as 랭킹엔진
    
    User->>UI: 입력: "혈압"
    UI->>API: POST /search<br/>{query: "혈압"}
    
    API->>Processor: 전처리 요청
    
    activate Processor
    Processor->>Processor: 1. 텍스트 정규화
    Processor->>Processor: 2. 형태소 분석<br/>(Mecab-ko)
    Processor->>MySQL: 동의어 조회
    MySQL-->>Processor: ["혈압", "BP", "blood pressure"]
    Processor->>Processor: 3. 동의어 확장 완료
    deactivate Processor
    
    Processor-->>API: 처리 결과 반환
    
    API->>MySQL: 1. 정확 매칭 쿼리
    MySQL-->>API: [var_1, var_2]
    
    API->>MySQL: 2. 동의어 매칭 쿼리
    MySQL-->>API: [var_3, var_4]
    
    API->>MySQL: 3. 부분 매칭 쿼리
    MySQL-->>API: [var_5, var_6]
    
    API->>MySQL: 4. FULLTEXT 검색
    MySQL-->>API: [var_7, var_8]
    
    API->>MySQL: 5. 인기도 조회<br/>(search_logs 집계)
    MySQL-->>API: 인기도 데이터
    
    API->>Ranking: 결과 통합 및 랭킹
    
    activate Ranking
    Ranking->>Ranking: • 중복 제거
    Ranking->>Ranking: • Base Score 계산
    Ranking->>Ranking: • Popularity 추가
    Ranking->>Ranking: • Category Match 추가
    Ranking->>Ranking: • 정렬 (상위 20개)
    deactivate Ranking
    
    Ranking-->>API: 최종 결과
    
    API->>MySQL: 검색 로그 저장
    
    API-->>UI: 검색 결과 (JSON)
    UI-->>User: 결과 표시
3.2 Phase 1 - 자동완성 시퀀스
mermaidsequenceDiagram
    actor User as 사용자
    participant UI as 프론트엔드
    participant API as Flask API
    participant Trie as Trie 엔진
    participant MySQL as MySQL
    
    User->>UI: 타이핑: "혈"
    
    Note over UI: 300ms 디바운싱 대기
    
    UI->>API: GET /autocomplete?q=혈
    
    API->>API: 1. 초성 검사<br/>"혈" → 한글
    
    API->>Trie: 2. Trie 탐색<br/>접두사: "혈"
    
    activate Trie
    Trie->>Trie: 접두사 매칭
    Trie-->>API: 후보: ["혈압", "혈당",<br/>"혈색소", "혈중지질", ...]
    deactivate Trie
    
    API->>MySQL: 3. 인기도 조회<br/>(search_logs)
    MySQL-->>API: 빈도 정보
    
    API->>API: 4. 정렬<br/>(인기도순, 상위 5개)
    
    API-->>UI: 제안 목록<br/>["혈압", "혈당", "혈색소",<br/>"혈중지질", "혈청"]
    
    UI-->>User: 드롭다운 표시
    
    User->>UI: 선택: "혈압"
    
    Note over UI,API: 검색 실행<br/>(기본 검색 시퀀스 참조)
3.3 Phase 2 - 하이브리드 검색 시퀀스
mermaidsequenceDiagram
    actor User as 사용자
    participant UI as 프론트엔드
    participant API as Flask API
    participant Processor as 검색프로세서
    participant Redis as Redis Cache
    participant ES as Elasticsearch
    participant FAISS as FAISS
    participant MySQL as MySQL
    participant Merge as 결과통합
    
    User->>UI: 입력: "혈압 측정"
    UI->>API: POST /search
    
    API->>Redis: 1. 캐시 확인
    Redis-->>API: 캐시 미스
    
    API->>Processor: 2. 전처리
    activate Processor
    Processor->>Processor: • 정규화
    Processor->>Processor: • 형태소 분석
    Processor->>Processor: • 동의어 확장
    deactivate Processor
    Processor-->>API: 처리 완료
    
    par 병렬 실행
        API->>ES: 3a. ES 키워드 검색
        activate ES
        ES->>ES: Multi-match 쿼리
        ES->>ES: Bool 쿼리
        ES->>ES: Function score
        ES-->>API: ES 결과<br/>[var_1:0.95, var_2:0.87, ...]
        deactivate ES
    and
        API->>FAISS: 3b. 쿼리 임베딩 생성
        activate FAISS
        FAISS->>FAISS: "혈압 측정" → 384차원
        FAISS->>FAISS: 3c. 코사인 유사도 계산
        FAISS-->>API: FAISS 결과<br/>[var_3:0.92, var_1:0.88, ...]
        deactivate FAISS
    end
    
    API->>MySQL: 4. 메타데이터 조회
    MySQL-->>API: • 인기도 (search_logs)<br/>• 최신성 (updated_at)
    
    API->>Merge: 5. 결과 통합 및 랭킹
    
    activate Merge
    Merge->>Merge: ES 점수 정규화 (0-1)
    Merge->>Merge: 가중 평균<br/>ES×0.6 + FAISS×0.4
    Merge->>Merge: 최종 점수 계산<br/>(가중평균×2.0) +<br/>Popularity×0.8 +<br/>Recency×0.6 +<br/>Category×1.1
    Merge->>Merge: 정렬 및 상위 20개 선택
    deactivate Merge
    
    Merge-->>API: 최종 결과
    
    API->>Redis: 6. 캐시 저장<br/>(TTL: 1시간)
    API->>MySQL: 7. 검색 로그 저장
    
    API-->>UI: 검색 결과 (JSON)
    UI-->>User: 결과 표시
3.4 연관 변수 추천 시퀀스
mermaidsequenceDiagram
    actor User as 사용자
    participant UI as 프론트엔드
    participant API as Flask API
    participant MySQL as MySQL
    participant VarRel as variable_relationships
    
    User->>UI: 클릭: "수축기혈압"
    UI->>API: GET /recommend?var_id=1
    
    API->>VarRel: 1. 도메인 지식 기반 조회
    activate VarRel
    VarRel->>VarRel: WHERE variable_id_1=1<br/>OR variable_id_2=1
    VarRel-->>API: [확장기혈압:0.95,<br/>심박수:0.85, ...]
    deactivate VarRel
    
    API->>MySQL: 2. 검색 컨텍스트 분석
    activate MySQL
    MySQL->>MySQL: SELECT v2, COUNT(*)<br/>FROM search_logs sl1<br/>JOIN search_logs sl2<br/>WHERE sl1.clicked=1<br/>AND sl1.session_id=sl2.session_id<br/>GROUP BY v2<br/>HAVING COUNT(*)>=3
    MySQL-->>API: [BMI:15회,<br/>혈압약:8회, ...]
    deactivate MySQL
    
    API->>MySQL: 3. 카테고리 기반 조회
    activate MySQL
    MySQL->>MySQL: SELECT * FROM variables<br/>WHERE category='심혈관계'<br/>ORDER BY popularity_score DESC
    MySQL-->>API: [맥압, 콜레스테롤, ...]
    deactivate MySQL
    
    API->>API: 4. 통합 및 정렬
    activate API
    API->>API: • 중복 제거
    API->>API: • 점수 통합
    API->>API: • 상위 5-10개 선택
    deactivate API
    
    API-->>UI: 추천 목록<br/>[확장기혈압, 심박수,<br/>BMI, 혈압약, 맥압]
    
    UI-->>User: 연관 변수 표시
4. 데이터베이스 ERD
mermaiderDiagram
    variables ||--o{ initial_mappings : "generates"
    variables ||--o{ search_logs : "clicked in"
    variables ||--o{ variable_relationships : "relates to"
    
    synonyms ||--o{ initial_mappings : "generates"
    medical_terms ||--o{ initial_mappings : "generates"
    
    variables {
        int variable_id PK
        string integrated_var_name UK "검색대상"
        string tablename "검색대상"
        string Variable
        string Variable_design
        text Label "검색대상"
        string FU
        string Type
        string AS
        string NC
        string CT
        string AS_NM
        string NC_NM
        string CT_NM
        string Value
        string Survey_Form
        text ETC
        string SER_NUM
        text search_text
        text search_tokens
        string search_initials
        text normalized_text
        decimal popularity_score
        timestamp created_at
        timestamp updated_at
    }
    
    synonyms {
        int synonym_id PK
        string standard_term
        string synonym_term
        string category
        enum language
        decimal confidence
        string source
        boolean is_active
        timestamp created_at
    }
    
    abbreviations {
        int abbr_id PK
        string abbreviation UK
        string full_term_en
        string full_term_ko
        string medical_domain
        int usage_frequency
        boolean is_active
    }
    
    medical_terms {
        int term_id PK
        string term_ko UK
        string term_en
        string term_latin
        text definition
        string major_category
        string minor_category
        json related_terms
        boolean is_active
    }
    
    initial_mappings {
        int mapping_id PK
        string initials
        string full_term
        enum term_type
        int reference_id FK
        int search_frequency
    }
    
    spacing_corrections {
        int correction_id PK
        string incorrect_form UK
        string correct_form
        string rule_type
        int priority
        boolean is_active
    }
    
    search_logs {
        bigint log_id PK
        string session_id
        string query_text
        string normalized_query
        int result_count
        int response_time_ms
        int clicked_variable_id FK
        int clicked_position
        string search_type
        string query_category
        timestamp created_at
    }
    
    variable_relationships {
        int relationship_id PK
        int variable_id_1 FK
        int variable_id_2 FK
        string relationship_type
        decimal strength
        text reason
        boolean is_active
    }
5. 배포 아키텍처
5.1 Phase 1 배포 구성
mermaidgraph TB
    subgraph "Production Server"
        Nginx[Nginx Web Server<br/>• 정적 파일 서빙<br/>• Reverse Proxy<br/>• SSL/TLS<br/>Port: 80, 443]
        
        Nginx --> Gunicorn[Flask + Gunicorn<br/>• Workers: 4<br/>• Port: 5000<br/>• Timeout: 30s]
        
        Gunicorn --> MySQL[(MySQL 8.0<br/>• Port: 3306<br/>• Max connections: 200<br/>• InnoDB buffer: 4GB)]
    end
    
    Internet[Internet] --> Nginx
    
    style Nginx fill:#e3f2fd
    style Gunicorn fill:#e8f5e9
    style MySQL fill:#f3e5f5
5.2 Phase 2 배포 구성 (프로덕션)
mermaidgraph TB
    Internet[Internet] --> LB[Load Balancer<br/>Nginx/HAProxy]
    
    LB --> App1[Flask App 1<br/>Gunicorn]
    LB --> App2[Flask App 2<br/>Gunicorn]
    LB --> App3[Flask App 3<br/>Gunicorn]
    
    subgraph "Application Layer"
        App1
        App2
        App3
    end
    
    App1 --> Redis
    App2 --> Redis
    App3 --> Redis
    
    App1 --> ES
    App2 --> ES
    App3 --> ES
    
    App1 --> MySQL_Master
    App2 --> MySQL_Master
    App3 --> MySQL_Master
    
    App1 --> FAISS
    App2 --> FAISS
    App3 --> FAISS
    
    subgraph "Cache Layer"
        Redis[Redis Cluster<br/>• Master<br/>• Replica 1<br/>• Replica 2]
    end
    
    subgraph "Search Layer"
        ES[Elasticsearch Cluster<br/>• Node 1<br/>• Node 2<br/>• Node 3]
        FAISS[(FAISS Index<br/>파일 시스템)]
    end
    
    subgraph "Database Layer"
        MySQL_Master[(MySQL Master<br/>Port: 3306)]
        MySQL_Slave[(MySQL Slave<br/>읽기 전용)]
    end
    
    MySQL_Master -.Replication.-> MySQL_Slave
    MySQL_Master -.동기화.-> ES
    MySQL_Master -.동기화.-> FAISS
    
    style LB fill:#e3f2fd
    style App1 fill:#e8f5e9
    style App2 fill:#e8f5e9
    style App3 fill:#e8f5e9
    style Redis fill:#ffebee
    style ES fill:#fff3e0
    style FAISS fill:#fce4ec
    style MySQL_Master fill:#f3e5f5
    style MySQL_Slave fill:#f3e5f5
6. 모니터링 아키텍처
mermaidgraph TB
    subgraph "모니터링 대시보드"
        Grafana[Grafana Dashboard<br/>• 실시간 차트<br/>• 알람 설정<br/>• 커스텀 패널]
    end
    
    subgraph "메트릭 수집"
        Prometheus[Prometheus<br/>• API 응답 시간<br/>• 검색 성공률<br/>• 제로 결과율<br/>• 캐시 히트율<br/>• 동시 접속자 수]
    end
    
    subgraph "Exporters"
        FlaskExporter[Flask Exporter]
        MySQLExporter[MySQL Exporter]
        RedisExporter[Redis Exporter]
        ESExporter[Elasticsearch Exporter]
    end
    
    subgraph "애플리케이션"
        Flask[Flask Apps]
        MySQL[(MySQL)]
        Redis[(Redis)]
        ES[(Elasticsearch)]
    end
    
    Grafana --> Prometheus
    
    Prometheus --> FlaskExporter
    Prometheus --> MySQLExporter
    Prometheus --> RedisExporter
    Prometheus --> ESExporter
    
    FlaskExporter --> Flask
    MySQLExporter --> MySQL
    RedisExporter --> Redis
    ESExporter --> ES
    
    style Grafana fill:#e3f2fd
    style Prometheus fill:#fff3e0
    style Flask fill:#e8f5e9
7. 지속적 개선 사이클
mermaidflowchart TD
    Start[검색 로그 수집<br/>실시간] --> Analysis[실패 패턴 분석<br/>주간]
    
    Analysis --> Zero[제로 결과 쿼리 분석]
    Analysis --> Slow[느린 쿼리 분석]
    Analysis --> LowClick[낮은 클릭률 분석]
    
    Zero --> Improvement[개선 방안 도출]
    Slow --> Improvement
    LowClick --> Improvement
    
    Improvement --> Synonym[동의어 추가]
    Improvement --> Ranking[랭킹 파라미터 튜닝]
    Improvement --> Index[인덱스 최적화]
    Improvement --> UIUX[UI/UX 개선]
    
    Synonym --> ABTest[A/B 테스트<br/>2주]
    Ranking --> ABTest
    Index --> ABTest
    UIUX --> ABTest
    
    ABTest --> Validate[성능 검증]
    
    Validate --> KPI{KPI 달성?}
    
    KPI -->|Yes| Deploy[단계적 배포]
    KPI -->|No| Analysis
    
    Deploy --> Deploy10[10% 트래픽]
    Deploy10 --> Monitor10{모니터링}
    
    Monitor10 -->|문제 발견| Rollback[롤백]
    Monitor10 -->|정상| Deploy50[50% 트래픽]
    
    Deploy50 --> Monitor50{모니터링}
    Monitor50 -->|문제 발견| Rollback
    Monitor50 -->|정상| Deploy100[100% 배포]
    
    Rollback --> Analysis
    Deploy100 --> Start
    
    style Start fill:#e3f2fd
    style Analysis fill:#fff3e0
    style Improvement fill:#e8f5e9
    style ABTest fill:#f3e5f5
    style Deploy fill:#c8e6c9