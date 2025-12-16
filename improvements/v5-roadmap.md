# CloudESM V5 개선 로드맵

> **버전**: 5.0 | **작성일**: 2025-12-16 | **상태**: 검토 중

---

## 목차

1. [검증된 이슈](#검증된-이슈)
2. [보안 강화](#보안-강화)
3. [성능 개선](#성능-개선)
4. [기능 개선](#기능-개선)
5. [코드 품질](#코드-품질)
6. [운영 자동화](#운영-자동화)
7. [로드맵 일정](#로드맵-일정)

---

## 검증된 이슈

### 🔴 ISSUE-001: API 권한 체크 일관성 부재 (Critical)

**발견일**: 2025-12-16  
**심각도**: 🔴 Critical  
**상태**: Open

#### 현상

| URL 타입 | 권한 체크 | 등록 현황 |
|----------|----------|----------|
| `.html` | ✅ COM_MENU + COM_ROLE_MENU | 121개 등록 |
| `.do` | ✅ COM_MENU + COM_ROLE_MENU | 148개 등록 |
| `.json` | ❌ 세션만 확인 | **1개만 등록** |

#### 검증 데이터 (실제 DB 조회 결과)

```sql
-- F타입 메뉴 URL 분포
url_type  | cnt
----------|-----
.do       | 148
.html     | 121
.json     | 1      ← 문제!
other     | 1

-- 실제 사용 중인 .json API: 792개
-- 권한 등록된 .json API: 1개 (/monitoring/string_decrypt.json)
```

#### 문제점

```
예시:
/system/comuser_list.html  → 권한 체크 O (관리자만 접근)
/system/comuser_list.json  → 권한 체크 X (누구나 호출 가능)
```

**791개의 .json API가 권한 체크 없이 노출됨**

#### 권장 조치

1. **단기**: 민감 조회 API를 COM_MENU에 F타입으로 등록
2. **중기**: Interceptor에서 `.html` ↔ `.json` 동일 권한 적용 로직 추가
3. **장기**: API Gateway 도입으로 통합 권한 관리

---

### 🟡 ISSUE-002: 테스트 코드 부재

**발견일**: 2025-12-16  
**심각도**: 🟡 Medium  
**상태**: Open

#### 검증 결과

```bash
# 테스트 파일 검색 결과
$ find . -path "**/test/**/*.java" 
No files found

# JUnit 관련 파일
No test classes found
```

#### 영향

- 회귀 버그 발생 위험 높음
- 리팩토링 시 안정성 검증 불가
- CI/CD 파이프라인 구축 어려움

---

## 보안 강화

### 우선순위 높음 🔴

- [ ] **API 권한 체크 일관성 확보** (ISSUE-001)
  - `.json` API 권한 등록 또는 Interceptor 수정
  - 예상 작업량: 2주

- [ ] **세션 보안 강화**
  ```java
  session.setMaxInactiveInterval(1800);  // 30분
  cookie.setSecure(true);
  cookie.setHttpOnly(true);
  ```

- [ ] **CSRF 토큰 검증 강화**
  - 현재: slKey 사용 중
  - 개선: 모든 POST 요청에 일관된 검증

### 우선순위 중간 🟡

- [ ] Multi-Factor Authentication (MFA/OTP)
- [ ] 비밀번호 정책 강화 (10자 이상, 특수문자, 90일 변경)
- [ ] 민감 데이터 DB 암호화 (AES-256)
- [ ] 로그 마스킹 (비밀번호, 개인정보)

### 우선순위 낮음 🟢

- [ ] OWASP Dependency Check 도입
- [ ] 보안 헤더 설정 (X-Frame-Options, CSP)
- [ ] 정기 보안 감사 로그

---

## 성능 개선

### 캐싱 전략 🔴

- [ ] **Redis 캐싱 도입**
  - 대상: 메뉴 목록, 코드 테이블, 사용자 권한
  - 기대 효과: DB 부하 60% 감소

  ```java
  @Cacheable(value = "menuCache", key = "#userId")
  public List<MenuVO> getMenuList(String userId) {
      return menuDAO.selectMenuList(userId);
  }
  ```

### 쿼리 최적화 🟡

- [ ] COM_MENU 조회 쿼리 최적화 (매 페이지 로드 방지)
- [ ] 로그 검색 복합 인덱스 추가 (날짜 + 로그레벨)
- [ ] 대시보드 통계 5분 단위 캐싱

### JVM 튜닝 🟢

```bash
# 현재
JAVA_OPTS="-Xms1024m -Xmx2048m"

# 권장
JAVA_OPTS="-Xms2048m -Xmx4096m -XX:+UseG1GC -XX:MaxGCPauseMillis=200"
```

---

## 기능 개선

### UI/UX 🟡

- [ ] 다크 모드 추가
- [ ] 반응형 웹 디자인 (모바일/태블릿)
- [ ] Toast UI Grid 가상 스크롤링 (1,000건+ 성능)
- [ ] 키보드 단축키 지원

### 검색 기능 🟡

- [ ] Elasticsearch 도입 (전문 검색)
- [ ] 고급 검색 필터 (AND/OR, 날짜 프리셋)
- [ ] 검색 결과 저장 기능

### 대시보드 🟢

- [ ] WebSocket 실시간 모니터링
- [ ] 커스텀 위젯 생성 (SQL 기반)
- [ ] 위젯 드래그앤드롭 배치

### 리포팅 🟢

- [ ] 일일/주간/월간 자동 보고서
- [ ] PDF/Excel 내보내기
- [ ] 이메일 자동 발송

---

## 코드 품질

### 테스트 코드 🔴

- [ ] JUnit 5 + Mockito 도입
- [ ] Service 레이어 단위 테스트 (목표: 60% 커버리지)
- [ ] Controller 통합 테스트
- [ ] Selenium E2E 테스트 (주요 시나리오)

### 리팩토링 🟡

- [ ] 중복 코드 제거 (BaseController 상속)
- [ ] Magic Number 상수화
- [ ] 100줄 이상 메서드 분리
- [ ] Exception 처리 표준화

### 문서화 🟢

- [ ] JavaDoc 작성 (public 메서드)
- [ ] Swagger/OpenAPI 3.0 도입
- [ ] 신규 개발자 온보딩 문서

---

## 운영 자동화

### CI/CD 🟡

- [ ] Jenkins Pipeline 구축
- [ ] Blue-Green 무중단 배포
- [ ] 자동 롤백

### 모니터링 🟡

- [ ] Prometheus + Grafana (APM)
- [ ] ELK Stack (로그 중앙화)
- [ ] Slack 장애 알림

### 백업 🟢

- [ ] DB 일일 자동 백업
- [ ] 설정 파일 백업
- [ ] 백업 복구 테스트 자동화 (월 1회)

---

## 로드맵 일정

### 2025 Q1 (1-3월) - 보안 집중

| 항목 | 우선순위 | 예상 기간 |
|------|----------|----------|
| API 권한 체크 일관성 | 🔴 Critical | 2주 |
| 세션 보안 강화 | 🔴 High | 1주 |
| Redis 캐싱 도입 | 🟡 Medium | 2주 |

### 2025 Q2 (4-6월) - 품질 향상

| 항목 | 우선순위 | 예상 기간 |
|------|----------|----------|
| 테스트 코드 작성 (30%) | 🔴 High | 4주 |
| CI/CD 파이프라인 | 🟡 Medium | 2주 |
| UI/UX 개선 | 🟡 Medium | 4주 |

### 2025 Q3 (7-9월) - 기능 확장

| 항목 | 우선순위 | 예상 기간 |
|------|----------|----------|
| Elasticsearch 도입 | 🟡 Medium | 3주 |
| APM 모니터링 | 🟡 Medium | 2주 |
| 테스트 커버리지 60% | 🟡 Medium | 4주 |

### 2025 Q4 (10-12월) - 고도화

| 항목 | 우선순위 | 예상 기간 |
|------|----------|----------|
| MFA 인증 | 🟢 Low | 2주 |
| 자동 보고서 기능 | 🟢 Low | 3주 |
| 테스트 커버리지 80% | 🟢 Low | 4주 |

---

## 변경 이력

| 버전 | 날짜 | 작성자 | 변경 내용 |
|------|------|--------|----------|
| 1.0 | 2025-12-16 | Claude | 초안 작성, 이슈 검증 완료 |

---

**참고 문서**:
- 원본 이슈트래킹.md
- 원본 개선사항.md
