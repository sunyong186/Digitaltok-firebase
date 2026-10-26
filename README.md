# DigitalTok Firebase Backend

> **DigitalTok** (E-ink NFC 디스플레이 서비스)의 Firebase 백엔드 및 인프라 프로젝트입니다.  
> 기존에 종료된 AWS Spring Boot REST 서버를 분석하여, Firebase(Auth, Firestore, Cloud Storage, Security Rules)로 완전히 재구축합니다.

---

## 📁 디렉토리 구조

```
digitaltok-firebase/
├── docs/                      # 프로젝트 분석 및 설계 문서
│   ├── migration_roadmap.md   # 전체 마이그레이션 로드맵 (Phase 0 ~ 8)
│   ├── firebase_backend_plan.md# Firebase 아키텍처 및 DB/스토리지 설계
│   ├── server_api_spec.md     # 기존 레거시 Spring Boot API 25개 역설계 문서
│   └── edep_protocol.md       # E-ink 전송용 ISO-DEP APDU NFC 프로토콜
├── scripts/                   # DB 시드 및 템플릿 생성 스크립트 (예정)
├── firestore.rules            # Firestore 보안 규칙 (예정)
├── storage.rules              # Storage 보안 규칙 (예정)
└── README.md
```

---

## 🤖 새 대화 세션에서 AI와 이어서 작업하는 방법

새 창에서 이 `digitaltok-firebase` 폴더를 열고 대화를 시작할 때, 아래 문장을 그대로 입력하시면 AI가 모든 맥락을 즉시 파악하고 바로 작업을 이어갈 수 있습니다:

> **"docs/ 폴더에 있는 migration_roadmap.md와 firebase_backend_plan.md 확인하고 다음 단계 진행해줘."**

---

## 📌 핵심 설계 요약

1. **이미지 양자화 & 바이너리 변환**
   - Cloud Functions 대신 **안드로이드 앱(Kotlin) 내부에서 직접 처리**하여 속도 향상 및 비용 최적화.
   - 200×200 해상도, 4색(흑/백/황/적), Floyd-Steinberg 디더링, MSB-first 비트 패킹 (10,000 bytes).

2. **인증 (Authentication)**
   - Firebase Auth (이메일/비밀번호 방식).

3. **데이터베이스 (Firestore)**
   - `users/{uid}`: 프로필
   - `users/{uid}/devices/{nfcUid}`: 연동된 E-ink 기기 목록
   - `users/{uid}/images/{imageId}`: 이미지 메타데이터
   - `templates_priority`, `templates_subway`: 교통약자 및 지하철 공용 템플릿

4. **스토리지 (Storage)**
   - `users/{uid}/images/{imageId}/` 경로에 원본, preview.png, eink.bin 저장.
