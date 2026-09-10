# 05-1. 로그 및 메트릭 수집 — PLG 스택 & Prometheus

> 수업 참고자료: `05-1_로그_및_메트릭_수집_PLG스택과_Prometheus.pdf`
> 프로젝트 코드: `complex-back`
> 정리일: 2026-09-10

---

## 0. 한 줄 요약

> 앱을 **건드리지 않고**(비침투), 로그는 **Promtail → Loki**로, 메트릭은 **Prometheus가 Actuator를 Pull**해서 모은 뒤, **Grafana 한 화면**에서 같이 본다.

---

## 1. 왜 필요한가 (개념)

| 구분 | 전통적 모니터링 | 클라우드 네이티브 관측성(Observability) |
|---|---|---|
| 핵심 질문 | "정상 작동하는가?" | "**왜** 이런 현상이 생겼나?" |
| 대응 대상 | 예측 가능한 문제(Known-unknowns) | 예측 불가능한 복합 장애(Unknown-unknowns) |
| 분석 방식 | 고정 임계치 + 단방향 경보 | 텔레메트리(로그·메트릭·추적) 다차원 추론 |
| 관찰 관점 | 인프라·서버의 외부 증상 | 시스템 내부 상태와 인과관계 |

- **컨테이너는 Cattle**: 수초 내에 생기고 사라짐 → 컨테이너가 죽으면 내부 `/app/logs` 파일도 같이 소멸. 그래서 **외부로 실시간 스트리밍 후 중앙 수집**이 필수 전제.
- **경보 체계**: 고정 임계치(`CPU > 85%`)는 최소 안전선으로만 두고, 복잡한 비즈니스 지표는 과거 시계열을 학습한 **동적 이상 감지**를 결합하는 것이 실무 표준.

### 로그 vs 메트릭 (상호 보완)

| 항목 | 로그 (Logs) | 메트릭 (Metrics) |
|---|---|---|
| 데이터 성격 | 이벤트 시점의 상세 맥락·원인 | 주기적 수치형 시계열 집계·추세 |
| 표현 형식 | 비정형 텍스트·JSON·스택 트레이스 | 숫자(Float)·타임스탬프·라벨 키-값 |
| 주요 타입 | 로그 레벨(TRACE ~ ERROR) | Counter · Gauge · Histogram · Summary |
| 보관 비용 | 볼륨·인덱싱 비용 큼 (단기 보관) | 초경량 (장기 보관 최적화) |
| 주 활용 | 장애 원인 심층 분석, 감사 추적 | 실시간 이상 감지, 임계치 경보 |

- **감사 추적(Audit Trail)**: '누가·언제·무엇을 했는가(5W1H)'를 증명하는 불변 이력 → 오직 로그 형태로만 보존 가능(컴플라이언스).

### 관측성 스택 비교

| | ELK / EFK | **PLG (이번 실습)** | LGTM |
|---|---|---|---|
| 구성 요소 | Elasticsearch·Logstash·Kibana | Promtail·Loki·Grafana | Loki·Grafana·Tempo·Mimir |
| 인덱싱 방식 | 로그 **본문 전체** 역색인 | **라벨만** 인덱싱 | 분산 추적·장기 메트릭 결합 |
| 자원 소모 | 매우 높음 | ELK 대비 대폭 경량 | 클라우드 네이티브 올인원 |
| 적합 환경 | 대규모 텍스트 검색 | 중소형 컨테이너 환경 | 대규모 MSA |

→ Loki는 본문을 압축 청크로만 저장하고 **라벨만 인덱싱**해서 메모리·스토리지 비용을 크게 줄임.

### Push vs Pull (메트릭 수집 모델)

| | Push 모델 | **Pull 모델 (Prometheus 표준)** |
|---|---|---|
| 수집 방식 | 앱이 수집 서버로 직접 전송 | 서버가 엔드포인트를 주기적으로 스크랩 |
| 부하 통제 | 트래픽 폭주 시 전송량도 폭주 | **서버 주도 주기 제어** → 추가 부하 차단 |
| 상태 감지 | 전송 두절 시 원인 파악 지연 | 스크랩 실패 시 **Target Down 즉시 감지** |

### Spring Boot 로깅 아키텍처

| 구성 요소 | 역할 |
|---|---|
| **SLF4J** | 로깅 엔진과 코드를 분리하는 표준 추상화 파사드(인터페이스) |
| **Logback** | SLF4J 공식 구현체, Spring Boot 기본 내장 고성능 엔진 |
| **@Slf4j** | Lombok 애너테이션. 컴파일 시점에 `Logger log` 필드 자동 주입 → 보일러플레이트 제거 |

| 레벨 | 의미 / 권장 환경 |
|---|---|
| TRACE | 가장 상세한 저수준 디버깅 (로컬 전용) |
| DEBUG | 개발·테스트 흐름 추적 (개발/스테이징 기본) |
| INFO | 핵심 운영 이벤트·정상 가동 상태 (운영 기본) |
| WARN | 잠재적 장애 경고 (모든 환경 상시) |
| ERROR | 예외·조치 필요 장애 (경보 연동 대상) |

---

## 2. 로그 파이프라인 (PLG)

**흐름:** `Spring Boot(Logback)` → 로그파일 → `Promtail`(파일 테일링) → `Loki`(저장) → `Grafana`(LogQL)

### ① Logback 로그 순환 — `src/main/resources/logback-spring.xml`

```xml
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${LOG_PATH}/${LOG_FILE_NAME}.log</file>   <!-- logs/spring-app.log -->
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>${LOG_PATH}/${LOG_FILE_NAME}-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
        <maxFileSize>10MB</maxFileSize>      <!-- 10MB 넘으면 분할(Rolling) -->
        <maxHistory>7</maxHistory>           <!-- 7일치만 보관 -->
        <totalSizeCap>100MB</totalSizeCap>   <!-- 전체 용량 상한 -->
    </rollingPolicy>
    <encoder>
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
</appender>

<root level="DEBUG">
    <appender-ref ref="CONSOLE"/>
    <appender-ref ref="FILE"/>
</root>
```

- **Log Rotation** = 날짜/용량 기준으로 파일을 쪼개고 오래된 건 자동 삭제 → 디스크 폭주 방지.
- `RollingFileAppender` + `SizeAndTimeBasedRollingPolicy` 조합이 핵심.

### ② Loki (저장 엔진) — `config/loki.yml`

```yaml
auth_enabled: false
server:
  http_listen_port: 3100
common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore: { store: inmemory }
schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index: { prefix: index_, period: 24h }
```

- Loki = 로그를 **저장**하는 엔진. 본문은 압축 청크로 보관하고 라벨만 인덱싱.

### ③ Promtail (수집 에이전트) — `config/promtail.yml`

```yaml
server:
  http_listen_port: 9080
positions:
  filename: /tmp/positions.yaml       # 체크포인트 파일
clients:
  - url: http://loki:3100/loki/api/v1/push   # 수집한 로그를 보낼 곳(endpoint)
scrape_configs:
  - job_name: spring-boot-logs
    static_configs:
      - targets: [localhost]
        labels:
          job: spring-boot
          app: complex-back
          __path__: /var/log/app/*.log        # 컨테이너 안에서 감시할 경로
```

- `__path__` = 감시할 파일 경로, `labels` = 검색용 메타데이터.
- **체크포인트(`positions.yaml`)**: 파일 inode + 읽은 바이트 오프셋을 실시간 기록 → 에이전트 재시작해도 **유실·중복 없이** 이어서 수집.
- 동작 방식: `tail -f`처럼 파일 추적 → 라벨 부착 → Loki API로 HTTP push.

### ④ compose 연결 — `compose.yml`

```yaml
services:
  app:
    volumes:
      - log-data:/app/logs              # 앱이 로그 파일 생성 (rw)

  loki:
    image: grafana/loki:latest
    ports: ["3100:3100"]
    volumes:
      - ./config/loki.yml:/etc/loki/local-config.yaml:ro
    command: [ -config.file=/etc/loki/local-config.yaml ]

  promtail:
    image: grafana/promtail:latest
    volumes:
      - log-data:/var/log/app:ro        # Promtail은 같은 볼륨을 읽기 전용으로 구독
      - ./config/promtail.yml:/etc/promtail/config.yml:ro
    depends_on: [ loki, app ]

  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]

networks: { mon-net: }
volumes: { log-data: }
```

- 같은 네임드 볼륨 `log-data`를 앱은 쓰고(`rw`), Promtail은 읽음(`ro`).
- 모두 같은 네트워크(`mon-net`) → Loki 주소는 호스트 포트가 아니라 **컨테이너 DNS 이름** `loki:3100`.
- (자료 원문은 바인드 마운트 `./logs`를 쓰지만, 이 프로젝트는 네임드 볼륨 `log-data`로 구현 — 계속 바뀌는 파일은 네임드 볼륨이 유리.)

### ⑤ LogQL 조회 (Grafana Explore)

```logql
{job="spring-boot"}                 # 라벨 일치 스트림 전체 조회
{job="spring-boot"} |= "ERROR"      # 본문에 ERROR 포함된 라인만 필터링
{job="spring-boot"} != "health"     # health 제외
{job="spring-boot"} |~ "status=[45][0-9]{2}"   # 정규식 필터
{job="spring-boot"} | json          # 파서로 필드 라벨화
rate({job="spring-boot"} |= "ERROR" [5m])       # 로그 발생 빈도를 메트릭으로 변환
```

트래픽 발생 예시:
```bash
curl -s http://localhost:8080/
for i in {1..50}; do curl -s http://localhost:8080/ > /dev/null; sleep 0.1; done
```

---

## 3. 메트릭 파이프라인 (Prometheus)

**흐름:** `Actuator/Micrometer` → `/actuator/prometheus` → `Prometheus`(Pull scrape) → `Grafana`(PromQL)

### ① 의존성 — `build.gradle`

```gradle
implementation 'org.springframework.boot:spring-boot-starter-actuator'
runtimeOnly   'io.micrometer:micrometer-registry-prometheus'   // Prometheus 텍스트 포맷 노출
```

### ② 엔드포인트 노출 — `src/main/resources/application.yaml`

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
        # health     : 서버 정상 구동 여부
        # info       : 관련 정보
        # prometheus : Pull이 이해하는 정보 포맷
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true      # kubernetes probe
```

- `/actuator/health` → `status`, `db` 모두 `UP` 확인
- `/actuator/prometheus` → `jvm_memory_used_bytes` 등 텍스트로 노출
- **의존성/설정을 새로 추가하지 않고** 클론한 프로젝트 구성만 확인 후 컨테이너만 재구동 (비침투).

```bash
grep -nE "actuator|micrometer" build.gradle
cat src/main/resources/application.yaml
curl -s http://localhost:8080/actuator/health
curl -s http://localhost:8080/actuator/prometheus
```

### ③ 스크랩 설정 — `config/prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']   # 자기 자신이므로 localhost 가능
  - job_name: spring-actuator
    metrics_path: /actuator/prometheus  # 스크랩 경로
    scrape_interval: 10s
    static_configs:
      - targets: [app:8080]             # 같은 네트워크의 서비스 이름
```

```yaml
# compose.yml
prometheus:
  image: prom/prometheus:latest
  ports: ["9090:9090"]
  volumes:
    - ./config/prometheus.yml:/etc/prometheus/prometheus.yml:ro
  depends_on: [ app ]
```

- `Status → Targets`에서 `spring-actuator`가 **UP**이면 성공.

### ④ 메트릭 4대 타입

| 타입 | 특성 | 사용 |
|---|---|---|
| **Counter** | 계속 증가만 하는 누적값, 재시작 시 리셋 | `rate()` · `increase()`로 변화율 계산 |
| **Gauge** | 오르내리는 순간 수치 | 현재 값 직접 조회 (메모리, 스레드 수) |
| **Histogram** | 구간별 빈도 집계 | `histogram_quantile(0.95, ...)`로 p95/p99 |
| **Summary** | 클라이언트가 백분위수 미리 계산해 전달 | — |

- `http_..._count`는 **순간 벡터**, `[1m]` 붙인 **구간 벡터**는 그래프로 못 그리고 `rate()` 같은 함수 인자로만 사용.

### ⑤ 핵심 PromQL 5종 (`http://localhost:9090/graph`)

```promql
http_server_requests_seconds_count                              # 원시 카운터
rate(http_server_requests_seconds_count[1m])                    # 초당 처리량(RPS), 리셋 자동 보정
sum(rate(http_server_requests_seconds_count[1m])) by (uri, status)   # 라벨 기준 시계열 합산
(sum(jvm_memory_used_bytes{area="heap"}) / sum(jvm_memory_max_bytes{area="heap"})) * 100   # 힙 사용률 %
jvm_threads_live_threads                                        # 라이브 스레드 수
```

---

## 4. Grafana 통합

1. **Connections → Data sources** 등록 (같은 네트워크이므로 컨테이너 DNS 사용)
   - Loki: `http://loki:3100`
   - Prometheus: `http://prometheus:9090`
   - 각각 **Save & test**
2. **커스텀 패널 3종**

   | 패널 | 유형 | 쿼리 |
   |---|---|---|
   | Request Rate | Time series | `sum(rate(http_server_requests_seconds_count[1m])) by (uri)` |
   | Heap Usage | Gauge | `sum(jvm_memory_used_bytes{area="heap"}) / sum(jvm_memory_max_bytes{area="heap"}) * 100` |
   | Thread Count | Stat | `jvm_threads_live_threads` |

   - Gauge는 Threshold(Green 0 / Yellow 70 / Red 85)로 **색만 보고 위기 감지**.
3. **표준 대시보드 임포트**

   | 대시보드 | ID | 주요 관측 항목 |
   |---|---|---|
   | JVM (Micrometer) | 4701 | 힙·논힙 메모리 풀, GC 멈춤 시간, 스레드 상태 |
   | Spring Boot System Monitor | 12856 | HTTP 처리량, 상태코드 비율, 응답 지연 |

   ```bash
   curl -s -L https://grafana.com/api/dashboards/4701/revisions/latest/download -o grafana/dashboards/jvm-micrometer-4701.json
   ```
4. **정리**: `docker compose down` (볼륨까지 지우려면 `-v`)

---

## 5. 전체 그림

```
                   [ 로그 ]                          [ 메트릭 ]
 Spring Boot ──파일 테일링──> Promtail            Spring Boot Actuator
   (Logback)                    │ HTTP push          /actuator/prometheus
      │                         ▼                          ▲
  spring-app.log             Loki  <라벨만 인덱싱>          │ Pull scrape (10~15s)
                               │                       Prometheus (TSDB)
                               │  LogQL                     │ PromQL
                               └────────►  Grafana  ◄───────┘
                                        (통합 대시보드)
```

| 영역 | 핵심 요약 | 비고 |
|---|---|---|
| 로그 수집 | App(Logback) → Promtail → Loki(인덱싱·저장) | 텍스트 로그 중앙 집중 |
| 메트릭 수집 | Micrometer/Actuator 엔드포인트를 Prometheus가 Pull | 시계열 메트릭 다차원 모니터링 |
| 시각화 통합 | Grafana 단일 플랫폼에서 로그(LogQL) + 메트릭(PromQL) | 장애 원인 다각도 추적 |
| 비침투 관측 | 소스 코드 수정 없이 표준 라이브러리 + 에이전트로 계측 | 운영 앱 영향 최소화 |

---

# 부록. 핵심 기술 면접 대비

## OpenSearch 탄생 배경

**Q. 2021년 Elasticsearch의 라이선스 전환 배경과, 이에 대응해 탄생한 OpenSearch의 역사적 의의는?**

### 1) 배경 — 라이선스 전환

Elasticsearch와 Kibana는 원래 **Apache 2.0**(자유로운 상업적 이용 가능) 라이선스였다. 그런데 AWS 같은 클라우드 벤더들이 Elasticsearch를 **관리형 서비스로 재판매**하면서도 개발사(Elastic)에 기여·수익 환원을 하지 않는 상황이 벌어졌다.

이에 Elastic사는 **2021년, 7.11 버전부터** 라이선스를 **SSPL(Server Side Public License) + Elastic License**의 듀얼 라이선스로 전환했다. SSPL은 "서비스로 제공하려면 관련 소프트웨어 전체를 공개하라"는 조건을 달아 사실상 클라우드 사업자의 상용 재판매를 막는 것으로, **OSI 공인 오픈소스가 아니다.**

### 2) 대응 — 포크(Fork)

순수 오픈소스의 지속 가능성을 지키기 위해 **AWS, Red Hat, SAP** 등 글로벌 커뮤니티가 **Apache 2.0 라이선스였던 마지막 코드베이스(Elasticsearch 7.10.2)**를 공식 포크해 **OpenSearch**와 **OpenSearch Dashboards**(Kibana 포크)를 출범시켰다. **Apache 2.0 영구 보장**을 표방한다.

### 3) 역사적 의의

- **오픈소스 라이선스 거버넌스의 대표 사례**: "오픈소스 개발사의 수익 모델" vs "커뮤니티의 자유로운 이용"이 충돌할 때 커뮤니티가 포크로 대응한 전형 (MongoDB, Redis 사례와 함께 자주 인용).
- **중립적 재단으로 이관**: 초기엔 AWS 주도라는 지적이 있었으나, 이후 **Linux Foundation 산하 OpenSearch Software Foundation**으로 이관되어 특정 벤더에 종속되지 않는 중립 프로젝트가 되었다.
- **실무적 선택지 확대**: 라이선스 리스크 없이 로그 분석·검색 스택을 쓰고 싶은 조직에 완전한 Apache 2.0 대안을 제공. 다만 이후 두 프로젝트는 기능이 갈라지기 시작(벡터 검색, ML 기능 등)해 상호 호환성은 점차 감소.

> 이번 수업의 **PLG(Loki)** 스택은 이 무거운 역색인 기반 스택(ELK/OpenSearch)의 **비용 문제를 회피**하려는 흐름의 산물이다. Loki 슬로건이 *"Like Prometheus, but for logs"* — 본문을 인덱싱하지 않고 라벨만 인덱싱한다.

---

## 그 외 예상 질문 요약

| 질문 | 핵심 답변 |
|---|---|
| **모니터링 vs 관측성** | 모니터링 = 사전 정의 임계치로 "정상인가?"(Known-unknowns) 외부 증상 감지. 관측성 = 텔레메트리로 "왜 이런 일이?"(Unknown-unknowns) 내부 상태를 다차원 추론하는 시스템의 내재적 능력. |
| **Loki vs ELK** | ES는 본문 전체를 역색인 → 전문 검색 빠르지만 메모리·스토리지 비용 큼. Loki는 본문 미인덱싱(압축 청크) + 라벨만 인덱싱 → 메모리 적고 운영 단순, 저비용 객체 스토리지 활용. |
| **Grafana 탄생 배경** | 2014년 Torkel Ödegaard가 **Kibana 3를 포크**. 벤더 중립적 플러그인 아키텍처로 단일 대시보드에서 Prometheus·Loki·Tempo·SQL을 패널 단위로 결합·상관 분석. Dashboard as Code(JSON) 지원. |
| **Prometheus Pull 모델 이점** | ① 중앙 서버가 스크랩 주기 주도 → 앱 트래픽 폭주해도 모니터링 트래픽은 안 폭주(부하 원천 차단). ② 중앙 서버가 엔드포인트를 주기 호출 → **Target Down 즉시 감지**(신뢰성). |
| **JUL vs SLF4J** | JUL = 7단계 모호한 명칭(SEVERE~FINEST), 설정 경직, 비동기 미지원. SLF4J = 파사드 패턴으로 인터페이스/구현 분리, 직관적 5단계(ERROR~TRACE), 런타임 엔진 교체 가능. `@Slf4j`는 컴파일 시점에 로거 필드를 바이트코드에 자동 삽입. |
| **Log4Shell & Logback 안전성** | Log4j 2가 로그 메시지의 `${jndi:ldap://...}` 표현식을 자동 평가 → 외부 LDAP에서 악성 클래스 다운로드·실행(RCE, CVE-2021-44228, CVSS 10.0). Logback은 **JNDI 룩업 기능 자체가 없어** 구조적으로 안전. SLF4J 뒤 구현체 선택이 보안 영향도를 좌우. |
| **Actuator 운영 보안** | ① 관리 포트 분리(`management.server.port: 8081`) + `address: 127.0.0.1` 사설망 격리. ② `base-path` 난독화로 자동 스캔 무력화. ③ `show-details: when-authorized`로 상세 조회 제한 + 노출 엔드포인트 최소화. |

---

## 깊이 알아보기

- Prometheus: <https://prometheus.io/docs/introduction/overview/>
- Loki: <https://grafana.com/docs/loki/latest/>
- Grafana: <https://grafana.com/docs/grafana/latest/>
- Micrometer: <https://micrometer.io/docs/>
- OpenSearch: <https://docs.opensearch.org/latest/>
- Spring Boot Actuator: <https://docs.spring.io/spring-boot/reference/actuator/index.html>
- SLF4J: <https://www.slf4j.org/manual.html>
