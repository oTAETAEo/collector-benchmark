# Collector Benchmark

#### Binance 형식 캔들 스트림을 기준으로 Spring MVC와 WebFlux collector의 Kafka publish 안정성과 흐름 제어 방식을 비교한다.

이 프로젝트에서 안정성은 다음과 같이 정의한다.
> 동일한 Binance 형식 캔들 스트림이 정상 부하와 burst 부하로 유입될 때, collector가 메시지를 유실하지 않고, symbol 단위 순서를 유지하며, Kafka publish 실패 없이 처리하고, 부하 종료 후 backlog를 회복하는 능력.

---

### Modules

```text
collector-benchmark/
├── common-domain
├── collector-mvc
├── collector-webflux
└── mock-binance-server
```

| 모듈 | 역할 |
| --- | --- |
| `common-domain` | 두 collector가 공통으로 사용하는 캔들 이벤트, Kafka topic, 측정용 메시지 계약을 정의합니다. |
| `collector-mvc` | Spring MVC/Tomcat 기반 collector입니다. Binance 형식 WebSocket 스트림을 수신한 뒤, 별도 worker 처리 구조로 Kafka 발행까지 수행합니다. |
| `collector-webflux` | Spring WebFlux/Reactor Netty 기반 collector입니다. Binance 형식 WebSocket 스트림을 수신한 뒤, Reactor 기반 처리 흐름으로 Kafka 발행까지 수행합니다. |
| `mock-binance-server` | Binance combined stream 형식의 캔들 WebSocket stream을 생성하는 테스트용 mock server입니다. rate, burst, disconnect 같은 시나리오를 만들어 collector에 동일한 입력을 제공합니다. |

---

## 1차 Benchmark Scope

> 1차 벤치마크는 로컬 단일 머신에서 진행하며, 실제 Binance 네트워크 환경을 재현하지 않는다. 동일 입력 조건에서 MVC의 worker 기반 흐름 제어와 WebFlux의 Reactor 기반 흐름 제어가 Kafka publish 안정성에 어떤 차이를 만드는지 비교한다.

| 지표 | 의미 |
| --- | --- |
| `generatedMessages` | mock-binance-server가 생성한 전체 메시지 수 |
| `receivedMessages` | collector가 WebSocket에서 수신한 메시지 수 |
| `publishedMessages` | Kafka publish ack를 받은 메시지 수 |
| `lossRate` | 생성된 메시지 대비 Kafka publish ack까지 도달하지 못한 비율 |
| `orderViolationCount` | 같은 `symbol` 기준 sequence 또는 event time 역전 횟수 |
| `publishFailureCount` | Kafka publish 실패 수 |
| `dlqCount` | 파싱 실패 또는 비정상 payload 처리 수 |
| `recoveryTimeAfterBurst` | burst 종료 후 backlog가 정상화되기까지 걸린 시간 |

---
