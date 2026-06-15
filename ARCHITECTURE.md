# ARCHITECTURE

RAG 파이프라인, 데이터 모델, 품질 관리 구조 요약. 항공 용어는 [README의 "빠른 도메인 용어 풀이"](README.md#빠른-도메인-용어-풀이) 참고.

---

## 1. 전체 그림

세 가지 사용자 흐름이 **하나의 RAG 인프라(embedding + pgvector)** 를 공유.

```
                         ┌─────────────────────────────┐
  공식 교재 PDF  ──────▶ │  청킹 → Gemini 임베딩(3072d) │ ──▶ Supabase
  (PHAK/AFH/IFH/AIM)     │       오프라인 배치           │     document_chunks
                         └─────────────────────────────┘     (pgvector)
                                                                  │
   ┌──────────────────────────────────────────────────────────── ┼ ──────────┐
   │                                  │                           │           │
   ▼                                  ▼                           ▼
 문제 생성(pool+live)            FAA Reference(사전 조회)     사용자 업로드 문서
 ACS 코드 정렬 출제              용어→원문 청크 인용          → 즉석 임베딩→문제
```

핵심 의도: **별도 시스템을 새로 짜지 않고 같은 파이프라인을 재사용**한다. "문제 생성"과 "출처 사전(FAA Reference)"은 동일한 임베딩·검색 위에 얹힌 두 표면.

---

## 2. RAG 파이프라인

1. 문서 → 청킹(1,000자, overlap 200) → Gemini 임베딩 → Supabase `document_chunks` 저장
2. 생성 요청 → 쿼리 임베딩 → `match_chunks` RPC(벡터 유사도) 검색 → 프롬프트 주입
3. RAG 결과가 빈약하면 랜덤 샘플링으로 자동 폴백

**내장 교재 모드(AeroLibrary)**: PHAK/AFH 등은 사전 임베딩 완료. 챕터 선택 시 `chapter_num` 필터로 DB 레벨에서 좁힌다. 전범위는 전용 랜덤 샘플링 RPC 사용.

**임베딩 차원 트레이드오프**: 3072차원을 쓴다. Supabase의 HNSW 인덱스는 2000차원 상한이라 이 임베딩은 인덱스 미사용(정확도 우선, 코퍼스 규모상 brute-force 검색으로 감내 가능한 범위).

---

## 3. 데이터 모델 (핵심 테이블)

| 테이블 | 역할 | 핵심 컬럼 |
|---|---|---|
| `document_chunks` | 교재 청크 + 임베딩 | `file_name`, `chunk_index`, `content`, `embedding`, `page_label`, `chapter_num` |
| `question_pool` | 사전 생성·검수 문제 풀 (자산) | `book`, `chapter_num`, `question_type`, `question`, `options`, `answer`, `evidence_quote`, `acs_code`, `cert_tags[]`, `like_count`, `dislike_count`, `is_active` |
| `feedback` | 사용자 평가 로그 | `question_text`, `rating`, `source` |

`question_pool`은 단순 캐시가 아니라 **curated asset**이다 — ACS 코드로 라벨링되고, `cert_tags`로 자격 등급이 태깅되고, 사용자 평가로 걸러진다.

---

## 4. 품질 관리 (3중 구조)

LLM 생성물의 품질을 세 단계로 관리.

1. **사전 예방 (프롬프트)** — 생성 스크립트의 `Forbidden Question Types`에 출제 가치 없는 9개 패턴을 카탈로그화(예: 출판사·인물명이 답인 문제, 비운항 역사 날짜, "according to this text" 메타 참조, Q↔A 거울 중복). 생성 단계에서 차단.
2. **자동 스캔 (오프라인)** — `check_acs_quality.py`로 MCQ/Short/Oral 전수 패턴 스캔. A/B등급은 생존, C등급 자동 비활성화. 전수 검수 결과 C등급 비율 0.4%.
3. **운영 중 자동 정제 (적자생존)** — 사용자 👍 70%+ 문제는 풀 편입(`promote_to_pool`), 👎 30%+(최소 3명)는 비활성화(`blacklist`). 풀 5,000 초과 시 하위 점수 자동 탈락. 정제로직은 차후 데이터가 쌓이면 최적화 및 조정 가능.

> "최소 3명" 게이트는 첫 한 명의 👎로 멀쩡한 문제가 사라지는 false-positive를 막기 위한 것.

---

## 5. 비용·지연 설계

- **풀 우선 혼합**: 응답의 70~90%를 사전 생성 풀에서 채우고, 나머지만 LLM 라이브 생성. live 수가 0이면 RAG·Gemini 호출을 **완전히 스킵** → 1~2초 응답.
- **thinking 비활성화**: Gemini `ThinkingConfig(thinking_budget=0)`로 생성 시간 1분+ → 15~30초.
- **재시도**: 503/429에 exponential backoff(1→2→4초, 최대 3회).
- **사후 차감**: 일일 쿼터는 실제 전달한 문제 수만큼만 차감(선차감 후 부분 실패 시 과다 차감되는 문제 회피).

---

## 6. 보안 모델 — "서버가 중개한다"

Streamlit 서버사이드 앱이라 브라우저가 DB에 직접 붙지 않고 **서버가 모든 DB 접근을 중개**한다. 민감 작업은 Supabase의 `SECURITY DEFINER` RPC로 감싸고, 클라이언트가 넘기던 `user_id`·`limit` 같은 신뢰 불가 입력을 제거해 `auth.uid()` + 서버측 검증으로 대체했다. `anon` 역할의 mutation RPC 실행 권한은 `REVOKE`.

이 "서버 중개" 원칙이 이후 프레임워크 선택(단순 SPA가 아니라 Next.js Server Actions)까지 좌우했다. 상세는 [01-build-journey.md](01-build-journey.md) §4~5.
