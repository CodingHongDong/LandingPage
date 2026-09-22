# 시스템 마이그레이션 분석 리포트 및 체크리스트

**문서 개요**
본 문서는 레거시 웹 서비스 환경(Apache 2.4.34, JDK 1.7, Tomcat 7)을 모던 환경(Apache 2.4.x, AWS Corretto 11, Tomcat 9)으로 전환하기 위한 핵심 점검 사항과 실행 계획을 담고 있습니다.

---

## 1. Java 마이그레이션 (JDK 1.7 → AWS Corretto 11)

Java 11 LTS로의 전환은 메모리 구조, 기본 패키지 구성, 보안 정책에 큰 변화를 가져오므로 철저한 대비가 필요합니다.

### 1.1 JVM 메모리 옵션 재구성 (PermGen 제거)
*   **이슈:** Java 8부터 `PermGen` 영역이 제거되고 네이티브 메모리를 사용하는 `Metaspace`가 도입되었습니다.
*   **조치 사항:** 
    *   기존 구동 스크립트(catalina.sh 등)에서 `-XX:PermSize`, `-XX:MaxPermSize` 파라미터를 완전히 제거합니다. (유지 시 JVM 기동 실패)
    *   대신 `-XX:MetaspaceSize`, `-XX:MaxMetaspaceSize`를 적절히 설정합니다.

### 1.2 Garbage Collector (GC) 변경 및 로깅
*   **이슈:** Java 11의 기본 GC는 `G1GC`입니다. 과거의 `CMS GC`는 제거되었습니다.
*   **조치 사항:** 
    *   기존 GC 튜닝 옵션을 G1GC에 맞게 재조정합니다.
    *   GC 로그 옵션을 통합 로깅 시스템(Xlog) 문법으로 변경합니다. (예: `-Xlog:gc*=info:file=gc.log:time,uptime:filecount=5,filesize=10M`)

### 1.3 누락된 Java EE 모듈(의존성) 복구
*   **이슈:** JDK 11부터 JAXB, JAX-WS, JTA, JAF 등 Java EE 관련 패키지가 JDK 코어에서 완전히 제거되었습니다.
*   **조치 사항:** 프로젝트 빌드 파일(pom.xml, build.gradle)에 필요한 라이브러리를 명시적으로 추가합니다.
    *   *점검 대상:* `javax.xml.bind`, `javax.activation`, `javax.annotation` 패키지 사용 여부

### 1.4 보안 프로토콜 (TLS) 업데이트
*   **이슈:** Corretto 11은 보안 강화를 위해 TLS 1.0과 1.1을 기본적으로 비활성화(`java.security`)합니다.
*   **조치 사항:** 레거시 외부 API, 구형 결제 모듈 등 연동 시스템이 TLS 1.2 이상을 지원하는지 사전 검증해야 합니다.

---

## 2. WAS 마이그레이션 (Tomcat 7 → Tomcat 9)

Tomcat 9는 향상된 성능과 엄격한 보안 표준을 요구합니다. 특히 Apache와의 AJP 통신 설정에 주의해야 합니다.

### 2.1 Ghostcat 취약점 패치에 따른 AJP 설정 강화
*   **이슈:** AJP 커넥터의 기본 보안 정책이 매우 엄격해졌습니다.
*   **조치 사항 (`server.xml`):**
    *   `secretRequired="true"`가 기본값이므로, Apache 측과 사전에 맞춘 암호키를 `secret="your_secret_key"` 속성으로 반드시 추가해야 합니다.
    *   물리적으로 다른 서버에 있다면 `address` 속성을 `0.0.0.0` 또는 외부 통신 IP로 명시해야 합니다. (기본값이 localhost 루프백으로 제한됨)

### 2.2 NIO 커넥터로의 전환
*   **이슈:** 과거의 BIO 커넥터(`Http11Protocol`)가 제거되었습니다.
*   **조치 사항:** NIO 커넥터(`Http11NioProtocol`)를 사용하도록 설정하고, 비동기 처리에 맞게 스레드 풀(`maxThreads`, `acceptCount` 등) 크기를 재산정합니다.

### 2.3 엄격한 쿠키 파싱 (RFC 6265)
*   **이슈:** Tomcat 9는 쿠키 값 내의 특수문자, 공백, 한글 등을 엄격하게 차단합니다.
*   **조치 사항:** 
    *   클라이언트 쿠키 생성 로직 점검 (URL Encoding 필수 적용).
    *   수정이 어려운 경우, `context.xml`에 `LegacyCookieProcessor`를 임시로 설정하여 하위 호환성을 유지합니다.

---

## 3. Web Server 마이그레이션 (Apache HTTPD 2.4.x)

*참고: 목표 버전이 '2.4.5'라면 오타일 확률이 높습니다(2013년 빌드). 최소 2.4.5x 이상 또는 최신 2.4 계열 유지를 권장합니다.*

### 3.1 mod_jk / mod_proxy_ajp 시크릿 키 설정
*   **이슈:** Tomcat 9의 AJP 보안 강화에 발맞추어 웹 서버 측 설정도 변경이 필요합니다.
*   **조치 사항:**
    *   `mod_jk`: `workers.properties`에 `worker.<이름>.secret=your_secret_key` 추가.
    *   `mod_proxy_ajp`: `ProxyPass / ajp://tomcat-ip:8009/ secret=your_secret_key` 형태로 옵션 추가.

### 3.2 최신 보안 통신(HTTPS) 적용
*   **조치 사항:** OpenSSL 버전을 점검하고, `ssl.conf`에서 SSLv3, TLS 1.0, 1.1을 차단(`SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1`)하여 보안 등급을 상향합니다.

---

## 4. 애플리케이션 및 서드파티 라이브러리 점검

| 점검 대상 | 주요 이슈 및 조치 방안 |
| :--- | :--- |
| **Spring Framework** | Spring 3.x/4.x 초반 버전은 Java 11을 지원하지 않음. **최소 Spring 4.3.x, 권장 5.x 이상**으로 프레임워크 업그레이드 필수. |
| **바이트코드 라이브러리** | ASM, CGLIB, Javassist 등의 구버전은 Java 11 클래스 로딩 시 에러 발생. 최신 버전으로 교체 필요. |
| **JDBC 드라이버** | DB 연결을 위한 드라이버(예: `ojdbc6.jar`)를 Java 11과 호환되는 최신 버전(예: `ojdbc8.jar`)으로 교체. |
| **로깅 프레임워크** | Log4j 1.x는 보안 취약점 및 호환성 문제가 있으므로 **Logback** 또는 **Log4j2**로 마이그레이션 권장. |

---

## 5. 권장 실행 파이프라인 (Action Plan)

1.  **[준비]** 인프라 구성: 테스트 서버에 AWS Corretto 11, Tomcat 9, Apache 최신 버전 설치
2.  **[빌드]** JDK 11 환경에서 애플리케이션 컴파일 테스트 및 누락된 의존성(JAXB 등) 추가
3.  **[배포]** Tomcat 9에 WAR 배포 후 Spring 컨텍스트 정상 로드 확인 (바이트코드 에러 모니터링)
4.  **[연동]** Apache - Tomcat 간 AJP 통신(Secret Key 적용) 테스트
5.  **[검증]** DB 연동, 대외 기관 API 호출(TLS 호환성), 쿠키 기반 세션 유지 기능 전수 테스트
6.  **[최적화]** G1GC 튜닝 및 부하 테스트(Load Test)를 통한 스레드 풀 조정
