# 2. 운영관리

> **CloudESM SOAR 플랫폼 일상 운영 및 모니터링 가이드**

[← 통합매뉴얼로 돌아가기](../통합매뉴얼.md)

---

## 프로젝트 버전 정보

> ⚠️ **이 문서는 Unpackage 버전 (빌드/배포 완료 후 상태)의 프로젝트를 대상으로 합니다.**

| 구분 | 설명 |
|------|------|
| **현재 버전** | Unpackage (WAR 압축 해제 후) |
| **특징** | `mvn package` → 빌드 → 배포 완료 상태 |
| **장점** | 직접 컴파일, 핫 리로드 가능 |
| **Package 버전** | Maven 빌드 전 소스 프로젝트 (별도 문서 예정) |

---

## 목차
1. [서비스 관리](#서비스-관리)
2. [로그 모니터링](#로그-모니터링)
3. [성능 모니터링](#성능-모니터링)
4. [백업 및 복구](#백업-및-복구)
5. [보안 관리](#보안-관리)
6. [개발 환경 설정](#개발-환경-설정)

---

## 서비스 관리

### 전체 서비스 시작

```bash
cd /CloudESM/app
./start.sh
```

### 전체 서비스 중지

```bash
cd /CloudESM/app
./stop.sh
```

### 개별 서비스 제어

```bash
# Tomcat
cd /CloudESM/app/tomcat9/bin
./startup.sh    # 시작
./shutdown.sh   # 중지
./force_stop.sh # 강제 종료
```

---

## 개발 환경 설정

> 이 섹션은 Unpackage 버전에서의 개발 작업을 위한 가이드입니다.

### 디렉토리 구조

| 구분 | 경로 | 설명 |
|------|------|------|
| **Java 소스** | `src/main/java/com/seculayer/web/` | 컨트롤러, 서비스, DAO |
| **컴파일 출력** | `app/www/ROOT/WEB-INF/classes/` | 배포된 .class 파일 |
| **라이브러리** | `app/www/ROOT/WEB-INF/lib/` | JAR 의존성 |
| **JSP** | `app/www/ROOT/WEB-INF/jsp/` | 뷰 템플릿 |
| **정적 리소스** | `app/www/ROOT/resources/` | JS, CSS, 이미지 |
| **MyBatis Mapper** | `app/www/ROOT/WEB-INF/classes/.../mapper/` | SQL 매핑 XML |

### Java 컴파일 (단일 파일)

```bash
cd "C:\projects\eyeCloudXOAR_V4_unpackage\CloudESM_CRAT"

javac -encoding UTF-8 \
  -d "app/www/ROOT/WEB-INF/classes" \
  -cp "app/www/ROOT/WEB-INF/lib/*;app/www/ROOT/WEB-INF/classes" \
  "src/main/java/com/seculayer/web/[패키지경로]/[파일명].java"
```

### 애플리케이션 리로드

```bash
touch "C:\projects\eyeCloudXOAR_V4_unpackage\CloudESM_CRAT\app\www\ROOT\WEB-INF\web.xml"
```

### 코딩 규칙 요약

| 구분 | 규칙 |
|------|------|
| **Java 로깅** | `logger.debug()`, `logger.error(msg, e)` 사용 |
| **금지** | `System.out.println`, `e.printStackTrace()` |
| **MyBatis** | `#{param}` 사용, `${param}` 금지 |
| **JS POST** | `$("body").requestData()` 필수 |
| **CSRF** | 모든 POST에 `slKey` 파라미터 필수 |
| **알림/확인** | `_alert()`, `_confirm()` 사용 |

---

[← 통합매뉴얼로 돌아가기](../통합매뉴얼.md)

**작성일**: 2025-12-17 | **버전**: 1.1
