---
title: 경매 권리 분석 서비스 아키텍처
created: 2026-09-05
updated: 2026-09-05
status: active
tags:
  - project
  - backend
  - 개발
---

# 경매 권리 분석 서비스 아키텍처

> 설계 초안. 실제 구현이나 배포가 완료된 구성이 아니다. 두 입력 방식, 비동기 처리, 모니터링, Kubernetes 배포를 목표로 한다.

## 1. 목표와 설계 범위

- 법원과 사건번호로 대상 물건을 선택해 자료를 자동 수집한다.
- 사용자가 문서를 직접 업로드하거나, 자동 수집한 자료를 업로드로 보완한다.
- 공통 파이프라인에서 원문을 파싱하고 검증한 뒤 AI 분석 결과를 근거가 포함된 리포트로 제공한다.
- 접수·조회와 오래 걸리는 작업을 격리하고, 실패한 작업을 저장된 중간 결과에서 재개한다.
- 메트릭·로그·트레이스를 수집하고 Kubernetes에서 배포·복구·확장을 실험한다.
- 리소스 절약을 위한 언어 변경은 동일한 부하에서 측정한 결과로 결정한다.

초기 제안은 세 개의 애플리케이션 배포 단위다. 입력, 수집, 파싱, 분석, 리포트라는 다섯 처리 단계가 각각 별도 서비스일 필요는 없다.

## 2. 전체 구성도

실선은 요청 또는 저장소 접근, 점선은 비동기 전달 또는 외부 연동이다. 화살표는 요청 방향이며 응답은 같은 경로로 돌아온다. 큐와 저장소의 클러스터 내부·외부 배치 및 제품은 미정이다.

```mermaid
flowchart LR
    subgraph client ["사용자"]
        webApp["웹 화면: 사건 조회·문서 업로드·리포트 조회"]
    end

    subgraph platform ["Kubernetes 배포 대상"]
        ingress["인그레스: HTTPS 진입점"]
        apiService["접수·조회 API"]
        ingestWorker["수집·파싱 워커"]
        analysisWorker["분석·리포트 워커"]
    end

    subgraph messaging ["비동기 메시징"]
        jobBroker["내구성 있는 작업 큐"]
    end

    subgraph persistence ["영구 저장 계층"]
        database[("관계형 DB: 작업·문서 메타데이터·분석 버전·발행 대기 이벤트")]
        objectStore[("오브젝트 저장소: 원문·리포트 파일")]
    end

    subgraph external ["외부 시스템"]
        sourceApi["경매 자료 제공처: API·문서 수집 경로 확인 필요"]
        aiApi["AI 제공자 API"]
    end

    webApp -->|"입력·상태 조회·결과 조회"| ingress
    ingress -->|"요청 전달"| apiService
    apiService -->|"작업·이벤트 저장 및 결과 조회"| database
    apiService -->|"업로드 저장·리포트 읽기"| objectStore
    apiService -.->|"저장된 수집·파싱 요청 발행"| jobBroker

    jobBroker -.->|"수집·파싱 작업 전달"| ingestWorker
    ingestWorker -.->|"자료 조회·수집"| sourceApi
    ingestWorker -.->|"필요할 때 문서 추출 보조"| aiApi
    ingestWorker -->|"원문 읽기·저장"| objectStore
    ingestWorker -->|"파싱 결과·상태·후속 이벤트 저장"| database
    ingestWorker -.->|"저장된 분석 요청 발행"| jobBroker

    jobBroker -.->|"분석 작업 전달"| analysisWorker
    analysisWorker -->|"입력 스냅샷 읽기·분석 결과 저장"| database
    analysisWorker -.->|"검증된 입력으로 분석 요청"| aiApi
    analysisWorker -->|"리포트 파일 저장"| objectStore
```

| 배포 단위 | 책임 | 장애 격리 목적 |
|---|---|---|
| 접수·조회 API | 입력 검증, 대상 확인, 업로드, 작업 접수, 진행 상태·리포트 조회 | 워커 장애 중에도 가능한 접수·기존 결과 조회 유지 |
| 수집·파싱 워커 | 자동 수집, 문서 식별, 추출, 정규화, 누락·상충 확인 | 외부 자료 제공처 장애와 문서 처리 부하 격리 |
| 분석·리포트 워커 | 입력 스냅샷 검증, AI 호출, 결과 검증, 리포트 생성·버전 저장 | AI 지연·실패와 리포트 생성 부하 격리 |

DB와 큐를 공유하는 초기 구조는 완전히 독립된 마이크로서비스 구조가 아니다. 먼저 프로세스별 복구와 자원 제한을 검증하고, 독립 배포·확장의 이득이 확인되는 부분부터 경계를 강화한다.

## 3. 두 입력 경로와 공통 처리

```mermaid
sequenceDiagram
    autonumber
    actor user as 사용자
    participant apiService as 접수·조회 API
    participant database as 관계형 DB
    participant objectStore as 오브젝트 저장소
    participant jobBroker as 작업 큐
    participant ingestWorker as 수집·파싱 워커
    participant sourceApi as 자료 제공처
    participant analysisWorker as 분석·리포트 워커
    participant aiApi as AI API

    alt 사건번호로 자동 수집
        user->>apiService: 법원·사건번호 입력 후 대상 물건 확인
        apiService->>database: 작업과 수집 요청 이벤트를 같은 트랜잭션으로 저장
    else 문서 업로드 또는 기존 사건 보완
        user->>apiService: 문서 업로드와 분석 대상 지정
        apiService->>objectStore: 업로드 원문 저장
        apiService->>database: 문서 메타데이터·작업·파싱 요청 이벤트 저장
    end
    apiService-->>user: 접수 완료와 작업 식별자 반환
    apiService->>jobBroker: 저장된 이벤트 발행 후 발행 여부 기록
    jobBroker-->>ingestWorker: 수집 또는 파싱 작업 전달

    opt 자동 수집이 필요한 작업
        ingestWorker->>sourceApi: 대상 자료 요청
        sourceApi-->>ingestWorker: 수집 가능한 자료 또는 실패 정보
        ingestWorker->>objectStore: 수집된 원문 저장
    end
    ingestWorker->>objectStore: 파싱 대상 원문 읽기
    ingestWorker->>ingestWorker: 문서 식별·파싱·정규화·누락·상충 검증

    alt 필수 자료 부족 또는 대상·내용 충돌
        ingestWorker->>database: 보완·확인 필요 상태와 이유 저장
        user->>apiService: 상태 조회
        apiService->>database: 현재 상태 읽기
        apiService-->>user: 필요한 문서 또는 확인 항목 반환
    else 분석 가능한 입력 확보
        ingestWorker->>database: 입력 스냅샷·분석 요청 이벤트를 함께 저장
        ingestWorker->>jobBroker: 저장된 분석 요청 이벤트 발행
        jobBroker-->>analysisWorker: 분석 작업 전달
        analysisWorker->>database: 고정된 입력 스냅샷 읽기
        analysisWorker->>aiApi: 근거와 출력 형식을 지정해 분석 요청
        aiApi-->>analysisWorker: 분석 초안
        analysisWorker->>analysisWorker: 형식·원문 근거·미확인 사항 검증
        analysisWorker->>objectStore: 리포트 파일 저장
        analysisWorker->>database: 분석 결과·리포트 버전·완료 상태 저장
    end

    user->>apiService: 작업 상태 또는 완료 리포트 조회
    apiService->>database: 상태·분석 결과·리포트 위치 조회
    opt 완료된 리포트 파일 요청
        apiService->>objectStore: 접근 권한을 확인한 리포트 읽기
    end
    apiService-->>user: 현재 상태 또는 리포트 반환
```

- 사건은 법원·사건번호로 식별하고, 개별 매각 물건을 구분한다. 문서에서 식별한 정보가 불명확하면 사용자 확인을 받는다.
- 문서 업로드는 외부 자료 수집 시스템이 동작하지 않아도 접수·처리할 수 있어야 한다.
- 보완 문서를 올리면 같은 사건에 새 입력 버전을 만들고 공통 처리 흐름으로 다시 진입한다.
- 한 번의 분석은 고정된 문서 버전 집합을 사용한다. 새 문서가 추가되어도 이미 실행 중인 입력을 바꾸지 않는다.
- 자동 수집·업로드라는 출처만으로 우선순위를 정하지 않는다. 문서 작성일과 기준 시점을 비교하고 충돌은 드러낸다.
- 자료가 없다는 사실과 권리가 없다는 판단은 구분한다. 부족한 근거로 확정 결론을 만들지 않는다.

## 4. 상태와 신뢰성

작업 상태의 예시는 다음과 같다. 명칭과 세부 전이는 구현 시 확정한다.

| 상태 | 의미 | 후속 처리 |
|---|---|---|
| 접수됨 | 입력과 작업이 영구 저장됨 | 큐 발행 및 워커 처리 대기 |
| 수집 중·파싱 중 | 자료 확보 및 구조화 진행 | 중간 결과 저장 |
| 보완 필요·확인 필요 | 자료 누락 또는 대상·내용 충돌 | 업로드·확인 후 새 입력 버전 처리 |
| 분석 중·리포트 생성 중 | 검증된 입력을 분석하고 결과 작성 | 결과 검증 및 저장 |
| 완료 | 리포트가 저장되고 조회 가능함 | 새 자료가 오면 새 분석 버전 생성 |
| 재시도 대기 | 일시적 오류로 다음 시도 예약 | 횟수·기한 제한 내 재개 |
| 실패 | 자동 복구 한도 초과 또는 복구 불가 오류 | 원인 확인 후 명시적 재처리 |

### 작업 유실과 중복 방지

- 상태 변경과 후속 이벤트는 같은 DB 트랜잭션으로 저장하는 발행 대기함 방식을 사용한다.
- 이벤트 발행기는 각 생산 프로세스의 백그라운드 작업으로 시작한다. DB 트랜잭션을 종료한 뒤 큐의 발행 확인을 받고 발행 상태를 갱신한다.
- 큐 발행 확인과 DB 갱신 사이에 종료되면 중복 발행될 수 있다. 전달은 최소 한 번을 전제로 설계한다.
- 작업 식별자·단계·입력 버전에 고유 제약과 원자적 작업 선점을 적용한다. 처리 중 종료된 작업은 임대 만료 후 재개하며 오래된 실행이 최신 결과를 덮어쓰지 못하게 한다.
- 중간 결과와 상태를 저장한 뒤 메시지 처리를 확인한다. 메시지에는 파일 본문 대신 식별자·스키마 버전·추적 문맥을 전달한다.
- 파일 저장과 DB 저장은 하나의 트랜잭션이 아니다. 임시 업로드의 완료 여부를 확인하고, 연결되지 않은 파일은 유예 기간 후 정리한다.
- 외부 AI가 요청을 처리한 뒤 응답만 유실될 수 있다. 제공자의 멱등성 지원을 확인하고, 재시도 한도·호출 이력·비용 상한으로 중복 비용을 통제한다.
- 외부 API 호출 동안 DB 트랜잭션이나 연결을 장시간 점유하지 않는다.

### 장애 격리

- 외부 호출별 타임아웃, 동시 실행 제한, 제공자별 호출 한도를 둔다.
- 일시적 오류는 지수 백오프와 임의 지연으로 제한적으로 재시도한다. 잘못된 입력은 무한 재시도하지 않는다.
- 반복 실패 작업은 실패 큐 또는 영구 실패 기록으로 격리하고, 원인과 재처리 이력을 남긴다.
- 큐 장애 시 접수 데이터가 DB에 저장되면 발행 대기 상태로 유지한다. 적체 한도를 넘으면 신규 접수를 제한한다.
- 외부 제공처 장애 중 기존 리포트 조회는 유지할 수 있지만 새 분석 완료는 지연될 수 있다.
- DB·오브젝트 저장소 장애까지 애플리케이션 분리만으로 해결되지는 않는다.

## 5. 모니터링 구성도

다음은 목표 구성이다. 메트릭 수집부터 도입하고 로그·트레이스를 순차적으로 연결한다. 로그 본문에 문서 원문이나 인증정보를 기록하지 않는다.

```mermaid
flowchart LR
    subgraph applications ["관측 대상"]
        apiMetrics["접수·조회 API"]
        ingestMetrics["수집·파싱 워커"]
        analysisMetrics["분석·리포트 워커"]
        clusterMetrics["노드·컨테이너·Pod 상태 수집 지점"]
    end

    subgraph telemetry ["수집 계층"]
        collector["OpenTelemetry Collector"]
    end

    subgraph monitoring ["모니터링 저장·조회"]
        prometheus["Prometheus"]
        loki["Loki"]
        tempo["Tempo"]
        grafana["Grafana"]
        alertmanager["Alertmanager"]
    end

    subgraph operations ["운영"]
        operator["운영자"]
    end

    apiMetrics -.->|"메트릭·로그·트레이스"| collector
    ingestMetrics -.->|"메트릭·로그·트레이스"| collector
    analysisMetrics -.->|"메트릭·로그·트레이스"| collector
    prometheus -->|"노출된 메트릭 수집"| collector
    prometheus -->|"클러스터 메트릭 수집"| clusterMetrics
    collector -.->|"로그 전송"| loki
    collector -.->|"트레이스 전송"| tempo
    grafana -->|"메트릭 조회"| prometheus
    grafana -->|"로그 조회"| loki
    grafana -->|"트레이스 조회"| tempo
    prometheus -.->|"알림 규칙 평가 결과"| alertmanager
    alertmanager -.->|"장애 알림"| operator
    operator -->|"대시보드 확인"| grafana
```

클러스터 메트릭 수집 지점은 kubelet의 컨테이너 메트릭, 노드 메트릭 수집기, kube-state-metrics 등의 논리적 묶음이다. 실제 배포 컴포넌트는 구현 시 확정한다.

| 영역 | 우선 지표 |
|---|---|
| 사용자 경험 | 접수 성공률, 리포트 완료율, 접수부터 완료까지의 시간 |
| 작업 큐 | 단계별 대기 수, 가장 오래 대기한 시간, 재시도·실패 수, 발행 대기 이벤트 수 |
| 수집·파싱 품질 | 제공처별 실패율, 문서 종류별 추출 실패, 보완·충돌 비율 |
| AI 분석 | 호출 지연, 오류, 토큰 사용량·비용, 근거 검증 실패 |
| 리소스 | 프로세스 CPU·메모리, 재시작·OOM, DB 연결 수, 저장소 잔여량 |

- HTTP와 큐 메시지에 추적 문맥을 전파한다. 재처리는 이전 시도와 연결하고 영구 작업 식별자로 조회 가능하게 한다.
- 작업·사건 식별자는 로그·트레이스에 사용하되 메트릭 라벨에는 넣지 않는다.
- 모델·프롬프트·파서·규칙 버전과 입력 문서 버전을 기록해 결과의 변경 원인을 추적한다.
- 접수 성공률과 분석 완료율을 분리해, 접수만 되고 작업이 멈춘 상태를 감지한다.
- 모니터링 저장소의 보관 기간과 디스크 사용량도 제한하고 관측 시스템 자체 장애를 확인한다.

## 6. Kubernetes 배포와 가용성

- 세 애플리케이션을 각각 Deployment로 배포하고 독립적인 리소스 요청·제한과 복제본 수를 설정한다.
- 인그레스는 접수·조회 API에만 연결한다. 워커와 데이터 계층은 외부에 직접 노출하지 않는다.
- 정상 종료 시 새 작업 수신을 멈추고 실행 중 작업의 상태를 저장한다. 종료 기한을 넘기면 다른 워커가 재개한다.
- 시작·준비·생존 상태 검사를 구분한다. 외부 AI 일시 장애만으로 프로세스를 계속 재시작하지 않는다.
- 워커 증설은 큐 대기 시간과 외부 API 허용량을 함께 본다. 복제본 증가로 제공자 제한을 넘지 않게 한다.
- 영구 데이터는 Pod 로컬 파일시스템에 의존하지 않는다. PVC 또는 외부 저장소의 백업·복구를 별도로 검증한다.
- 단일 노드에서는 프로세스 복구를 실험한다. 노드 장애에 대한 고가용성을 주장하지 않는다.
- 실제 고가용성은 다중 노드 분산, 충분한 여유 용량, DB·큐·저장소·제어 평면의 복구 설계를 함께 다룬다.

## 7. 기술 선택 상태

| 항목 | 현재 상태 |
|---|---|
| 언어 | Java·Spring Boot 기준 구현 제안. Go·Python 전환은 측정 후 결정 |
| 저장소 구성 | 관계형 DB + 오브젝트 저장소. 제품과 운영 위치 미정 |
| 비동기 처리 | 내구성 있는 큐와 검증된 클라이언트 사용. 제품 미정 |
| AI | 외부 API 호출 전제의 초안. 제공자·모델·입력 보관 조건 미정 |
| 자동 수집 | 문서별 제공처·API 지원·인증·비용·이용 조건 확인 필요 |
| 사용자 화면 | 사건 조회, 문서 업로드, 보완 안내, 진행 상태, 리포트 조회. 프레임워크 미정 |
| 배포 | Kubernetes 목표. 로컬 개발 환경은 언어 선정 후 구성 |
| 관측성 | OpenTelemetry, Prometheus, Loki, Tempo, Grafana, Alertmanager 목표 |

리소스 비교는 동일한 문서·동시 처리량에서 최대 메모리, CPU 사용량, 처리 시간, 실패율, 전체 비용을 측정한다. 프로세스 기본 사용량·DB 연결·관측 시스템·외부 AI 비용을 포함하며, 언어 변경만으로 절약을 보장하지 않는다.

## 8. 첫 검증 범위

- [ ] 자동 수집할 문서별 확보 경로와 조건 확인
- [ ] 검토된 학습용 사례와 기대 추출 결과 준비
- [ ] 두 입력 경로가 동일한 문서·입력 스냅샷 구조로 합류
- [ ] 자동 수집 실패를 업로드로 보완하고 새 버전 분석
- [ ] 분석 결과에서 원문 근거와 미확인 사항 확인
- [ ] 워커 강제 종료 후 작업 유실 없이 재개
- [ ] 중복 메시지 처리 후 리포트·상태 중복 없음 확인
- [ ] 외부 API 장애 중 기존 리포트 조회 유지
- [ ] 메트릭·로그·트레이스로 실패 단계 추적

## 참고 자료

- [법제처: 경매 물건과 자료 확인](https://www.easylaw.go.kr/CSP/CnpClsMainBtr.laf?ccfNo=2&cciNo=2&cnpClsNo=1&csmSeq=306)
- [법원 사법정보공유포털](https://openapi.scourt.go.kr/kgso000m02.do)
- [AWS: 재시도와 멱등성](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)
- [OpenTelemetry: 추적 문맥 전파](https://opentelemetry.io/docs/concepts/context-propagation/)
- [Kubernetes: 장애 대응](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [NIST: 생성형 AI 위험 프로필](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf)
