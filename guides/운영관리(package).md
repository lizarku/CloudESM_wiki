# 2. 운영관리 (Package 버전)

> **eyeCloudXOAR V4 Maven 소스 프로젝트 개발 및 빌드 가이드**

[← 통합매뉴얼로 돌아가기](../통합매뉴얼.md)

---

## 프로젝트 버전 정보

> ⚠️ **이 문서는 Package 버전 (Maven 빌드 전 소스 프로젝트)을 대상으로 합니다.**

| 구분 | 설명 |
|------|------|
| **현재 버전** | Package (Maven Multi-Module 소스) |
| **특징** | `mvn clean package`로 빌드 필요 |
| **장점** | 전체 소스 접근, 모듈별 빌드 가능 |
| **Unpackage 버전** | 빌드/배포 완료 후 상태 ([운영관리(unpackage).md](운영관리(unpackage).md) 참조) |

---

## 프로젝트 개요

eyeCloudXOAR V4는 다양한 소스로부터 보안 및 시스템 로그를 수집, 정규화, 분석, 시각화하는 분산 엔터프라이즈 보안 데이터 분석 플랫폼입니다. 버전 4.0.4는 Java 11과 Maven을 사용합니다.

---

## 빌드 명령어

```bash
# 전체 빌드
mvn clean package

# 개별 모듈 빌드
cd <모듈명> && mvn clean package

# 테스트 스킵
mvn clean package -DskipTests

# 프론트엔드 빌드 (Web 모듈)
cd web/src/main/frontend && npm install && grunt

# 프론트엔드 개발 모드 (파일 변경 감시)
cd web/src/main/frontend && grunt watch
```

---

## 배포

각 모듈은 `/CloudESM/app/`로 산출물을 복사하는 배포 스크립트를 가지고 있습니다:

```bash
cd agent && ./deploy_agent.sh
cd collector && ./deploy_collector.sh
cd analyzer && ./deploy_analyzer.sh
cd web && ./deploy_web.sh
```

**중요**: 배포 스크립트는 `db.properties`, `initial_adaptors` 같은 민감한/운영 설정 파일을 제외합니다. 배포 시 기존 설정이 `backup/YYYYMMDDHHMMSS/` 디렉토리에 백업됩니다.

---

## 아키텍처

### 모듈 구조 (23개 모듈)

| 계층 | 모듈 |
|------|------|
| **데이터 수집** | `agent`, `agent-proxy`, `collector`, `ha_collector`, `ha_sender` |
| **데이터 처리** | `nomalizer`, `parser-tailer`, `reducer` |
| **분석 & 검색** | `analyzer`, `analyzer-assistant`, `searcher`, `soar_analyzer` |
| **모니터링** | `netflow`, `pms`, `procmonitor` |
| **Web & UI** | `web`, `terminal`, `launcher`, `visualization` |
| **지원** | `common`, `proxy`, `datamanager`, `integrity` |

### 데이터 흐름

```
Agents → Collector → Normalizer → Searcher → Analyzer → SOAR → Web UI
                ↓                      ↓           ↓
          /CloudESM/data         전문검색     데이터베이스
                                   인덱스      (544+ 테이블)
```

### 주요 기술

| 기술 | 버전 | 용도 |
|------|------|------|
| **Java** | 11 (일부 1.8) | 백엔드 |
| **Spring Framework** | 4.2.9 | MVC, JDBC, 트랜잭션 |
| **MyBatis** | 3.2.8 | 데이터베이스 ORM |
| **Quartz** | 2.1.4 | 작업 스케줄링 |
| **Lucene** | 6.6.0 | 전문 검색 |
| **Database** | MariaDB/MySQL, Oracle, Tibero | 멀티 DB 지원 |
| **Frontend** | Grunt + Babel | 빌드 도구 |

---

## 설정 패턴

모든 모듈은 `[@placeholder]` 속성 치환을 사용하는 XML 기반 설정을 사용합니다:

```
<모듈>/conf/
├── <모듈>-conf.xml          # 애플리케이션 설정
├── db.properties           # 데이터베이스 자격증명 (배포 시 제외)
├── mybatis-configuration.xml  # ORM 설정
└── log4j2.properties       # 로깅 설정
```

---

## 데이터베이스 테이블 명명 규칙

| 접두어 | 설명 |
|--------|------|
| `AGENT_*` | Agent 설정 및 어댑터 정의 (40개 이상) |
| `ALARM_*` | 알람 규칙, 조건, 이력, 수신자 |
| `APE_*` | AI/ML 데이터셋, 모델 (AI-based Prediction Engine) |
| `SMS_*` | 시스템 모니터링 데이터 |
| `NETFLOW_*` | 네트워크 플로우 데이터 |
| `USER_*` | 사용자 관리 및 권한 |
| `DASHBOARD_*` | 대시보드 정의 |
| `VW_*` | 사용자별 데이터 접근을 위한 뷰 |
| `MGR_*` | 시스템 관리 (MGR_AGENT 등) |

**파티셔닝**: 대용량 테이블은 YYYYMMDD 컬럼에 일일 범위 파티셔닝을 사용합니다.

---

## 어댑터 아키텍처

Agent 모듈은 플러그인 가능한 어댑터 패턴을 사용합니다:

- **기본 클래스**: `AbstractAdaptor`, `Task` (`agent/src/main/java/com/seculayer/collection/adaptor/`)
- **명명 규칙**: `[소스/프로토콜][목적]Adaptor` (예: `SyslogAdaptor`, `TCPAdaptor`)
- **주요 어댑터**: `CharSetDirFileTailingAdaptor`, `DBLogAdaptor`, `SyslogAdaptor`, `SNMPTrapAdaptor`, `JMXAdaptor`, `SystemMonitor`

---

## 일반적인 개발 작업

### 새 어댑터 추가

1. `agent/src/main/java/com/seculayer/collection/adaptor/`에 `AbstractAdaptor`를 상속하는 클래스 생성
2. `open()`, `read()`, `close()` 메서드 구현
3. `AGENT_ADAPTOR` 데이터베이스 테이블에 등록
4. `AGENT_ADAPTOR_PARAMETER` 테이블에 파라미터 추가
5. Agent 모듈 빌드 및 배포

### 데이터베이스 스키마 수정

1. `db/db/table/` 디렉토리에서 스키마 업데이트
2. `Documents/db_update_script(ko).sql`에 마이그레이션 스크립트 추가
3. `src/main/java/com/seculayer/*/mapper/`의 MyBatis 매퍼 XML 파일 업데이트
4. 영향받는 모듈 재빌드

---

## 중요 사항

| 항목 | 설명 |
|------|------|
| **멀티 DB 지원** | MariaDB, Oracle, Tibero 처리 필요. MyBatis 매퍼 확인 |
| **HA 복제** | Collector는 primary와 HA 백업 위치 모두에 출력 |
| **큐 관리** | Agent/Collector는 OOM 방지를 위해 메모리 제한 큐 사용 |
| **파티셔닝** | 시계열 테이블의 일일 파티션 생성 필요 |
| **설정 보존** | 배포 스크립트는 운영 설정을 덮어쓰지 않음 |

---

## 개발 환경 및 디버깅

### Agent 로그 설정

Agent 모듈의 `agent.sh`는 기본적으로 nohup 출력을 `/dev/null`로 리다이렉션합니다. 디버깅을 위해:

```bash
# agent.sh의 start 섹션 수정 예시
LOG_DIR="./logs"
mkdir -p $LOG_DIR
LOG_FILE="$LOG_DIR/agent_$(date +%Y%m%d).out"
nohup $JAVA_HOME/bin/java $JAVA_OPTS -classpath $CLASSPATH com.seculayer.collection.agent.CollectorForAgent >> "$LOG_FILE" 2>&1 &

# 로그 확인
tail -f logs/agent_$(date +%Y%m%d).out
```

### Agent 관련 데이터베이스 테이블

```sql
-- 등록된 Agent 확인
SELECT * FROM MGR_AGENT;

-- 사용 가능한 Adapter 목록
SELECT adaptor_name, connect_type, description FROM AGENT_ADAPTOR WHERE use_yn = 'Y';

-- 특정 Agent의 설정 확인
SELECT * FROM MGR_AGENT WHERE agent_ip = '<IP주소>';
```

### Agent 트러블슈팅

| 에러 | 원인 | 해결 |
|------|------|------|
| `Address already in use` | 중복 포트 바인딩 | `netstat -tulpn \| grep <포트>` |
| `NullPointerException` in processAddCommandE | DB Adapter 설정 null | DB 연결 및 `AGENT_ADAPTOR` 테이블 확인 |
| 514 포트 권한 문제 | 1024 이하 포트 | `sudo ./agent.sh start` |

### 외부 Syslog 수신 테스트

```bash
# 원격 Linux에서 rsyslog 설정 (/etc/rsyslog.d/50-remote.conf)
*.* @<Agent-IP>:514   # UDP
*.* @@<Agent-IP>:514  # TCP

# 테스트 메시지 전송
logger -n <Agent-IP> -P 514 "Test message - $(date)"
echo "test syslog message" | nc -u <Agent-IP> 514

# Agent 측 확인
tail -f /CloudESM/app/agent/logs/agent_$(date +%Y%m%d).out
sudo netstat -tulpn | grep 514
ps aux | grep CollectorForAgent | grep -v grep
```

---

## 중요 주의사항

> ⚠️ 운영 환경에서 Agent의 `agent-conf.xml` 또는 DB 설정을 임의로 수정하면 서비스 장애가 발생할 수 있습니다.

설정 변경 절차:
1. 테스트 환경에서 먼저 검증
2. 백업 수행
3. 유지보수 시간에 적용
4. 로그 모니터링을 통한 정상 작동 확인

---

[← 통합매뉴얼로 돌아가기](../통합매뉴얼.md)

**작성일**: 2025-12-17 | **버전**: 1.0
