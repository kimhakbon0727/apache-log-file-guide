# 🌐 Apache 로그 파일 완전 가이드

> 언제 어떤 로그를 봐야 하는지 빠르게 찾기 위한 실무 레퍼런스

---

## 📁 Apache 로그 파일 종류

| 로그 파일 | 한 줄 요약 |
|---|---|
| `access_log` | 외부에서 들어온 모든 HTTP 요청/응답 기록 |
| `error_log` | Apache 자체 오류 및 경고 메시지 |
| `ssl_access_log` | HTTPS 요청/응답 기록 (SSL 설정 시) |
| `ssl_error_log` | SSL/TLS 관련 오류 |
| `mod_jk.log` | Tomcat 연동(mod_jk) 관련 오류 |

---

## 🔍 로그별 상세 설명

### 1. `access_log` — HTTP 접근 로그

**= 외부 사용자의 모든 요청이 찍히는 로그**

#### 이럴 때 확인
- 사용자 요청이 **Apache까지 도달했는지** 확인할 때
- **응답코드**(200, 403, 404, 500 등) 전체 현황 파악 시
- **특정 IP**의 접근 이력 추적 시
- **트래픽 급증** 또는 **DDoS 의심** 시
- Tomcat에는 로그가 없는데 오류 신고가 들어올 때 → Apache에서 막힌 것인지 확인

#### 주요 내용
```
192.168.1.10 - - [01/Jul/2025:14:32:01 +0900] "POST /board/register HTTP/1.1" 200 5678
192.168.1.10 - - [01/Jul/2025:14:32:05 +0900] "GET /static/main.js HTTP/1.1" 304 -
```
→ IP / 날짜·시간 / 메서드+URL / **응답코드** / 바이트 수

#### 실무 grep 명령어
```bash
# 500 에러 요청만 추출
grep " 500 " /var/log/httpd/access_log

# 특정 IP의 요청만 추출
grep "^192.168.1.10" /var/log/httpd/access_log

# 특정 URL 요청 확인
grep "/board/register" /var/log/httpd/access_log

# 오늘 날짜 요청만 확인
grep "$(date +%d/%b/%Y)" /var/log/httpd/access_log | tail -50

# 403/404 에러만 모아보기
grep -E " 403 | 404 " /var/log/httpd/access_log | tail -30
```

---

### 2. `error_log` — Apache 오류 로그

**= Apache 서버 자체에서 발생한 오류와 경고**

#### 이럴 때 확인
- Apache가 **뜨지 않거나 재시작 실패** 시
- **설정 파일 오류** (`httpd.conf`, `.htaccess` 등)
- **Tomcat 연동 실패** (mod_jk, mod_proxy 오류)
- 403 Forbidden이 왜 나는지 원인 파악 시
- **SSL 인증서 오류** 또는 HTTPS 설정 문제

#### 주요 내용
```
[Wed Jul 01 14:32:01.123456 2025] [error] [client 192.168.1.10] File does not exist: /var/www/html/favicon.ico
[Wed Jul 01 14:33:00.000000 2025] [proxy:error] AH00898: Error reading from remote server
[Wed Jul 01 14:33:01.000000 2025] [auth_basic:error] AH01617: user test: authentication failure
```

#### 실무 grep 명령어
```bash
# 오류만 추출
grep "\[error\]" /var/log/httpd/error_log | tail -50

# Tomcat 연동 오류 확인
grep -E "proxy|jk" /var/log/httpd/error_log | tail -30

# 실시간 모니터링
tail -f /var/log/httpd/error_log
```

---

### 3. `mod_jk.log` — Tomcat 연동 로그

**= Apache ↔ Tomcat 사이의 연동 오류 전용 로그**

#### 이럴 때 확인
- Apache는 정상인데 **Tomcat으로 요청이 안 넘어갈 때**
- **503 Service Unavailable** 이 날 때
- Tomcat 인스턴스 중 하나가 **죽었는지** 확인할 때
- **로드밸런싱** 동작이 이상할 때

#### 주요 내용
```
[Wed Jul 01 14:35:00.000] [error] ajp_send_request::jk_ajp_common.c: failed sending request to tomcat-instance1
[Wed Jul 01 14:35:00.000] [error] ajp_connect_to_endpoint::jk_ajp_common.c: connect to 127.0.0.1:8009 failed
```

#### 실무 grep 명령어
```bash
grep -i "error\|failed" /var/log/httpd/mod_jk.log | tail -30

# 특정 인스턴스 오류만
grep "tomcat-instance1" /var/log/httpd/mod_jk.log | tail -20
```

---

## 📂 OS별 기본 로그 경로

### CentOS / RHEL / Rocky 계열
```bash
/var/log/httpd/access_log
/var/log/httpd/error_log
/var/log/httpd/ssl_access_log
/var/log/httpd/ssl_error_log
```

### Ubuntu / Debian 계열
```bash
/var/log/apache2/access.log
/var/log/apache2/error.log
/var/log/apache2/other_vhosts_access.log
```

### 직접 컴파일 설치 (공공기관 서버에 많은 방식)
```bash
/usr/local/apache/logs/access_log
/usr/local/apache2/logs/access_log
/opt/apache/logs/access_log
```

### 🔍 경로 모를 때 찾는 법
```bash
# httpd.conf에서 CustomLog 위치 직접 확인
grep -i "CustomLog\|ErrorLog" /etc/httpd/conf/httpd.conf
grep -rni "CustomLog\|ErrorLog" /etc/httpd/conf.d/

# 아파치 설정 경로 확인
apachectl -V | grep SERVER_CONFIG_FILE
```

---

## 🚦 상황별 로그 선택 가이드

```
오류 발생!
    │
    ├─ Apache 자체가 안 뜬다 / 설정 오류
    │       └─→ error_log
    │
    ├─ 요청이 Apache까지 왔는지 확인 (응답코드, IP 추적)
    │       └─→ access_log
    │
    ├─ Apache는 정상인데 Tomcat으로 안 넘어감 (503)
    │       └─→ mod_jk.log 또는 error_log에서 proxy 오류 확인
    │
    └─ HTTPS / SSL 인증서 오류
            └─→ ssl_error_log
```

---

## 🔗 Apache + Tomcat 연동 구조에서 로그 흐름

```
사용자 요청
    │
    ▼
Apache access_log  ← 요청 기록
    │
    ▼ (mod_jk / mod_proxy)
mod_jk.log         ← 연동 오류 기록
    │
    ▼
Tomcat localhost_access_log  ← Tomcat 도달 기록
    │
    ▼
Tomcat localhost.log         ← 앱 예외 기록
```

> 500 에러 디버깅 순서: **Apache access_log → mod_jk.log → Tomcat localhost.log** 순으로 확인

---

## 💡 실무 팁

### access_log 포맷 읽는 법
```
[IP] [식별자] [유저] [날짜시간] "[메서드 URL 프로토콜]" [응답코드] [바이트수]
192.168.1.1  -       -         [01/Jul/2025:14:00:00 +0900] "GET /index.jsp HTTP/1.1" 200 4523
```

### 응답코드 빠른 참조
| 코드 | 의미 | 주로 보는 로그 |
|---|---|---|
| 200 | 정상 | - |
| 304 | 캐시 사용 (변경 없음) | - |
| 403 | 접근 거부 | error_log |
| 404 | 리소스 없음 | access_log |
| 500 | 서버 내부 오류 | Tomcat localhost.log |
| 503 | 서비스 불가 (Tomcat 다운) | mod_jk.log |

### 실시간 동시 모니터링
```bash
tail -f /var/log/httpd/access_log /var/log/httpd/error_log
```

---

## 📌 요약 한 줄 정리

- **access_log** → 외부 요청/응답 전체 기록, IP 추적, 트래픽 분석
- **error_log** → Apache 기동 오류, 설정 오류, 403 원인 파악
- **mod_jk.log** → Apache ↔ Tomcat 연동 오류, 503 원인 파악
- **ssl_error_log** → HTTPS, 인증서 관련 오류

---

*마지막 업데이트: 2025-07*
