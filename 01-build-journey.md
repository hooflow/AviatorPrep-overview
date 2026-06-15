# AviatorPrep — FAA 항공 시험 prep 플랫폼

> 1인 개발. RAG 기반 FAA 시험 prep 웹 앱. 핵심 엔지니어링 결정과 한계를 정리한다.
>
> 항공 용어(FAA·PHAK·ACS·구술시험 등)는 [README의 "빠른 도메인 용어 풀이"](README.md#빠른-도메인-용어-풀이)에 쉬운 번역으로 정리해 뒀다.

## 무엇을 만들었나

[AviatorPrep](https://aviatorprep.app)은 FAA 항공 시험(PPL·IFR·CPL) 준비 웹 앱이다. 공식 교재(PHAK·AFH·IFH·AIM)를 임베딩한 vector DB에서 ACS(Airman Certification Standards) 코드 기준으로 문제를 제공하고, 사용자 업로드 자료로도 문제를 생성한다.

- 사전 생성·검수 문제 풀: 4,000+ 개
- ACS 구술시험 풀: 600+ 개 (PPL 201 / IFR 110 / CPL 305)
- 운영: 2026-04 ~ 현재 / 결제 미도입(도네이션만)

## 핵심 결정 4가지

1. **문제 풀을 자산으로** — 매 요청 LLM 생성 대신 사전 생성·검수 풀. 비용·latency·일관성.
2. **RAG 인프라를 두 기능에 재사용** — 같은 embedding·pgvector가 문제 생성 + FAA Reference 사전을 만든다.
3. **Streamlit 상태 격리 원칙** — OAuth PKCE race를 `cache_resource` vs `session_state` 구분으로 해결.
4. **보안 모델 보존이 기준** — 출시 전 보안 하드닝과 Next.js 이주 결정 모두 "서버가 중개한다"를 기준으로 내렸다.

---

## 1. 문제 풀을 자산으로

매 요청마다 LLM으로 문제를 생성하면 비용·latency·일관성이 모두 약하다. 사전 생성·검수 풀을 자산으로 키우는 방향으로 갔다.

- **사전 생성**: 오프라인 스크립트로 PHAK 17챕터 + AFH 16챕터 = 1,759개 생성(비용 ~₩3,000). IFH·AIM 추가해 현재 4,000+.
- **풀 비율 50/50 → 70/30**: 풀 의존을 높여 LLM 호출이 줄고 API 비용 ~40% 감소.
- **품질 정제**: PHAK+AFH+IFH MCQ ~2,034개 전수 검수, ~232개 비활성화. 출제 가치 없는 C등급 비율 0.4%까지 하락. 생성 스크립트의 Forbidden Question Types에 9패턴을 카탈로그화해 사전 예방.
- **사용자 피드백 기반 자동 운영**: 👍 70%+ → 풀 편입, 👎 30%+(최소 3명) → 비활성화. 풀 5,000 초과 시 하위 점수 자동 비활성화. "최소 3명"은 첫 한 명의 👎로 사라지는 false-positive 방지.

## 2. RAG 인프라 재사용

FAA Reference 탭은 단어·약어·CFR 조항을 던지면 공식 소스의 정확한 청크를 페이지 단위로 돌려주는 사전이다. 문제 생성과 **같은 RAG 파이프라인을 재사용**한다 — 별도 시스템을 새로 짜지 않았다.

도메인 튜닝:
- **V-speed 약어 보정**: `Vy` 같은 짧은 코드는 expansion 사전(`Vy → best rate of climb speed`)으로 맥락을 부풀린 뒤 임베딩.
- **CFR 패턴 감지**: `91.213`·`Part 61` 정규식으로 Regulation 슬롯 확대.
- **품질 필터**: CFR boilerplate·TOC·150자 미만 청크 제외.
- **fake hit 회피**: top 유사도 0.58 미만이면 빈 결과. 비항공 입력에 그럴듯한 청크를 붙이지 않는다.

## 3. Streamlit 상태 격리

Google OAuth가 hot-reload·동시 세션에서 "code challenge does not match"로 실패하는 race를 발견했다. 원인은 `@st.cache_resource`로 만든 Supabase 클라이언트의 PKCE 저장소가 모든 세션에서 공유된 것 — 한 세션의 verifier가 다른 세션 것을 덮어썼다.

| 패턴 | 용도 |
|------|------|
| `@st.cache_resource` | DB 연결·모델 같은 진짜 stateless 리소스 |
| `st.session_state` | OAuth·인증 같은 사용자별 mutable state |

이 구분이 이후 모든 인증·결제·세션 코드의 기준이 됐다.

## 4. 출시 전 보안 하드닝

마케팅 진입 전에 보안·법적·라이선스 정비를 끝냈다. 트래픽이 늘면 변경 비용이 커진다. 위험도순으로 LLM 비용 abuse를 1순위로 잡았다.

- **`increment_usage` RPC 재설계 (무중단)**: 구버전은 `user_id`·`limit`를 클라이언트가 넘겨 한도 위조·우회가 가능했다. v2 양립 패턴(`auth.uid()` 가드 + 서버측 limit lookup) 추가 → 전환 → 구버전 DROP → `REVOKE EXECUTE FROM anon`. 다운타임 0. 같은 패턴을 mutation RPC 4개에 적용.
- **secrets.toml import-order race**: 재배포 후 첫 진입에서 "API Key 미설정" 간헐 발생. `import streamlit`이 secrets watcher를 등록해 그 뒤 작성된 `secrets.toml`이 캐시 race. 작성 코드를 import 이전으로 옮겨 해결.
- **라이선스**: PyMuPDF(AGPL) → pdfplumber(MIT) 5파일 이주. 네트워크 서비스 소스공개 의무 회피.
- **PIPA**: 한국인 학생 잠재 시장 대응 — KO ToS·Privacy + 국외 이전 표 + DPO, 언어별 자동 분기.

## 5. Next.js 이주 — 보안 모델 보존이 기준

`AviatorPrep.py`가 3,500줄+ 단일 파일에 `st.*` ~500회(`session_state` ~280회)로 누적. rerun 모델·SEO·모바일·디자인 한계로 Next.js 15(App Router)+TypeScript 이주를 결정했다.

단순 SPA가 아닌 Next.js를 택한 이유는 **보안 모델 보존**이다. SPA는 DB 접근이 브라우저 JWT로 직접 이뤄져, 현재 `USING(true)` RLS상 인증된 누구나 전체 행 접근이 가능해진다 — 미뤄둔 RLS 전면 강화가 강제 선행됐을 것이다. Next.js의 Server Actions/Route Handlers는 서버사이드라 Gemini 키·Supabase service role을 서버에 두고, §4의 "서버가 중개한다" 모델이 그대로 유효하다.

호스팅은 Railway $5/월 유지.

## 포지셔닝

필기 prep 시장은 포화(Sporty's·King·최근 AI 서비스)다. ACS 코드로 정렬된 구술시험 풀은 자료가 흩어져 있다. ACS 코드 정렬 + curated 풀 + 피드백 누적을 중심에 뒀다. (사업 모델·가격 검토는 별도 전략 노트로 분리.)

## 한계

- 1인 개발 / 출시 후 본격 마케팅 미진입(사용자 메트릭 누적 전) / 결제 미도입
- ML 학습·fine-tuning 없음 — Gemini 2.5 Flash 호출 + 프롬프트 + RAG
- ACS 정렬 검증은 자동 패턴 스캔 + 샘플 인간 검수까지. 풀 인스펙션은 자원상 어려움
- 보안 정비는 1인 audit — 외부 pentest 미실시

---

*작성: 2026-05-29(재구성 2026-06-02). 출시 후 실 메트릭이 누적되면 정식 retrospective로 별도 작성 예정.*
