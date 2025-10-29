flowchart TD
    %% User & Frontend
    A[사용자 입력: 혈밥] --> E1[React UI - 검색창]
    E1 --> D1[Flask API 서버]

    %% LLM Query Layer
    D1 --> LLM1[LLM 오타/동의어 처리]
    LLM1 --> D1

    %% RAG Layer
    D1 --> B2[Vector Store: FAISS + KoBERT Embedding]
    B2 --> C2[LLM Re-ranking]
    C2 --> D1

    %% Results & Feedback
    D1 --> E2[결과 표시: 변수 목록 + 점수]
    E2 --> E3[추천어/동의어 표시]
    D1 -.-> D3[검색 로그 & 피드백 반영]
    D3 --> B3[자동 학습형 후보 DB 업데이트]

    %% Support Modules
    D1 -.-> D4[자동완성 / 추천어 모듈]

    %% Deployment
    D1 -.-> F1[AWS EC2 - Flask + KoBERT + LLaMA 서버]
    B2 -.-> F2[AWS RDS + Vector Store 동기화]
    D3 -.-> F3[AWS S3 - 로그/백업]
    D4 -.-> F4[Redis - 자동완성/인기검색]
    F1 -.-> F5[Auto Scaling + CloudWatch]
