```mermaid
graph TB
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
