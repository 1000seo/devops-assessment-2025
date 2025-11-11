# a. 클라우드 아키텍처 다이어그램
</br></br>

# b. 기술 선택 및 설계 근거
### 왜 이러한 아키텍처를 설계했는가?
1. **비즈니스 로직 고려**
    - S3 버킷의 Presigned URL을 발급하여 사용자(Client)가 이미지를 S3에 업로드하는 아키텍처로 설계
      - [**보안 강화**] 서버가 파일을 중계하지 않음 → 잠재적인 공격의 노출을 막을 수 있으며, S3 접근 권한을 제어 하여 업로드된 파일의 민감정보를 보호할 수 있다.
      - [**성능 향상**] 서버가 파일을 중계하지 않음 → 애플리케이션 서버의 네트워크 트래픽과 메모리 사용량을 줄여 애플리케이션 성능 및 업로드 속도를 향상시킬 수 있다.
    - SQS를 활용하여 운영 안정성을 확보하는 아키텍처로 설계
      - [**신뢰도 있는 비동기 처리**] 컨슈머(이미지 처리) 장애/배포 중에도 메시지는 큐에 남아, 처리 유실(서비스 장애) 발생의 확률을 감소시킨다.
      - [**운영 및 장애 대응 편의**] 큐 모니터링 지표는 운영 및 장애 분석에 활용될 수 있다.
    </br> 
2. **트래픽 패턴 고려**
   - 평소에는 요청이 적지만 월말에 급격히 증가하므로 Auto Scaling이 가능한 ECS를 사용하여 비용 효율성과 확장성을 확보했다. </br>

</br>

### 특정 AWS 서비스를 선택한 이유?

- **API Gateway**
  - S3를 외부에 직접 노출하지 않기 위해 API를 진입점으로 두었으며, 인증/인가·Rate Limit·로깅을 기본 제공해 보안과 운영 부담을 동시에 줄일 수 있다고 판단
- **ECS fargate**
  - 컨테이너 오케스트레이션은 필요하지만, EKS는 운영 오버헤드가 크다고 판단하여 제외하였고, 서버 운영 없이 Auto Scaling + Blue/Green 배포를 쉽게 적용 가능하여 선택
- **Aurora(PostgreSQL)**
  - Multi-AZ, 자동 백업/Failover가 기본 제공되어 직접 DB 클러스터를 운영하는 것보다 가용성과 장애 대응 측면에서 적합 
- **SQS**
  -  쉽게 구성 가능하고 운영 관리 부담이 낮은 AWS 관리형 큐가 적합하다고 판단하여 선택
- **Secret manager, parameter store**
  - 비용 효율을 고려하려 민감 정보는 보안이 강한 Secrets Manager, 일반 설정값은 비용 효율이 높은 Parameter Store 활용 
- **ECR**
  - ECS와 IAM 기반 권한 연동이 자연스럽고, 컨테이너 이미지 저장과 배포 안정성이 확보됨
    
</br>

### 주어진 요구사항(고가용성, Auto-scaling, 비용 효율성)을 어떻게 충족시켰는가?
- **고가용성**
  - Multi-AZ로 인프라 구성하여 단일 AZ 장애에도 서비스 지속 가능
  - Aurora Multi-AZ를 활용해 DB 장애 자동 Failover 및 데이터 안정성 확보
  - ALB + ECS Service 기반으로 단일 Task 장애가 전체 서비스에 영향 주지 않도록 분리 </br>
- **Auto-Scaling**
  - ECS(Fargate) + Auto Scaling으로 트래픽 변화에 따른 Task 수 자동 조절
  - 월말 트래픽 상승이 예측되는 구간에 대해서는 Scheduled Scaling을 적용하여, 부하 발생 이전에 안정적 리소스 확보</br>
- **비용 효율성**
  - Fargate 사용으로 서버 유지 비용 없이 Task 실행 시간만 과금 → 트래픽 적은 구간에서 비용 최소화
  - Secrets Manager + Parameter Store 분리 사용으로 민감 데이터는 보안 우선, 일반 설정은 비용 최적화 전략으로 저장

</br> </br>
# c. 운영 자동화 및 장애 대응 계획
### 모니터링 전략

- **모니터링 지표 정의** </br>

| 리소스         | Metric                               | 모니터링 목적 |
|----------------|--------------------------------------|----------------|
| **ECS**        | CPU Utilization                      | CPU 포화로 인한 요청 처리 지연 및 성능 저하 탐지 |
|                | Memory Utilization                   | 메모리 부족(OOM)으로 인한 컨테이너 비정상 종료 탐지 |
|                | RunningTaskCount vs DesiredTaskCount  | 배포 실패 또는 Task 비정상 종료 탐지 |
| **SQS**        | ApproximateAgeOfOldestMessage         | 메시지 백로그 증가로 인한 처리 지연 탐지 |
| **ALB**        | 5XX Error Rate                       | 서비스 장애로 인한 요청 실패 탐지 |
|                | TargetResponseTime (Latency)          | 서비스 응답 지연 및 성능 병목 탐지 |
| **API Gateway**| 5XX / 4XX Error Rate                 | 외부 요청 실패 및 비정상 호출 증가 탐지 |
|                | Latency                              | API 응답 지연 탐지 |
|                | IntegrationLatency                   | 백엔드 연동 구간 병목 및 지연 탐지 |
| **Aurora**     | CPU Utilization                      | DB 리소스 포화 및 성능 저하 탐지 |
|                | Database Connections                 | 과도한 커넥션 점유로 인한 요청 실패 탐지 |
|                | FreeStorageSpace                     | 저장 공간 부족으로 인한 장애 위험 탐지 |

</br>

- **알림 조건**

| 리소스                   | 알림 조건                                                   | 알림 방식 |
|--------------------------|-------------------------------------------------------------|-----------|
| **ECS**                  | CPUUtilization ≥ 80% (5분 지속)                             | Slack     |
|                          | MemoryUtilization ≥ 80% (5분 지속)                          | Slack     |
| **SQS**                  | ApproximateAgeOfOldestMessage ≥ 600s (15분 지속)            | Slack     |
| **ALB**                  | 5xx Error Rate ≥ 3% (1분 지속)                              | Slack     |
| **API Gateway**          | 5xx Error Rate ≥ 5% (3분 지속)                              | Slack     |
|                          | p90 Latency ≥ 2500ms (5분 지속)                             | Slack     |
|                          | p90 IntegrationLatency ≥ 2000ms (5분 지속)                  | Slack     |
| **Aurora**               | FreeStorageSpace < 10% of allocated storage (5분 지속)       | Slack     |
|                          | DatabaseConnections > 90% of max_connections (5분 지속)      | Slack     |

</br> </br>

### 무중단 배포 전략
**Blue/Green** 배포 전략을 사용하여 무중단 배포를 구성한다.

**선택 이유**
| 고려 요소 | 내용 |
|-----------|------|
| **ECS 기반 환경과 자연스럽게 연동됨** | ECS Native Blue/Green은 ECS Service 내부에 내장된 기능으로, 별도 CI/CD 도구 없이도 TaskSet 기반으로 Blue/Green 배포 가능 |
| **서비스 중단 없이 배포 가능** | 신규 버전(Green)을 미리 띄운 뒤 Health Check가 통과될 때만 트래픽을 전환하므로 배포 중에도 서비스 중단이 없음 |
| **롤백이 즉시 가능함** | 문제 발생 시 기존 Blue TaskSet으로 트래픽만 되돌리면 되므로, 롤백 속도가 매우 빠름 |

</br></br>

**배포 방식**
1. 기존 Blue 서비스를 유지한 상태로 신규 Green TaskSet을 병렬로 생성 </br>
2. ALB Target Group을 이용하여 트래픽을 단계적으로 전환 </br>
3. 자동 Health Check 및 Bake Time을 통해 장애 발생 시 안전하게 롤백 가능 </br>

</br></br>
**배포 흐름**
1. **Green TaskSet 생성**
   - 새 Task Definition 기반으로 Green TaskSet이 생성되고 기동됨 </br></br>
2. **Health Check 검증**
   - Container 및 ALB Target Group 헬스 체크로 동작 여부 확인 </br></br>
3. **테스트 트래픽 전송 (선택)**
   - Test Listener를 구성했다면 일부 트래픽을 Green으로 전송하여 검증 </br></br>
4. **프로덕션 트래픽 전환**
   - ALB가 Production Listener 트래픽을 기존 Blue → Green으로 100% 전환 </br></br>
5. **안정화 시간(Bake Time) 대기**
   - 일정 시간 동안 telemetry / 에러율 / Latency 등을 체크하여 이상 여부 판단 </br></br>
6. **Blue 종료**
   - 문제가 없으면 기존 Blue TaskSet 자동 종료

</br></br>

### 장애 대응 시나리오
**DB 연결 실패 시**

1. API 응답지연, 트랜잭션 타임아웃 등의 증상을 모니터링하여 탐지하고, 발생시 로그 및 지표를 확보한다.
2. 확보한 지표를 확인하여 max connection 갯수를 늘리거나 특정 세션을 종료함으로써 즉시 대응한다.
3. 성능 테스트를 통한 적정 커넥션 풀 갯수 튜닝, 쿼리 검수를 통한 슬로우 쿼리 튜닝 등의 방법으로 재발 방지한다. </br></br>
    
**특정 API 서버 장애 시**
1. 알림을 통해 장애발생 상황을 전파하고, 영향범위를 확인하여 장애 수준을 판별한다.
2. 로그 등의 지표를 통해 원인을 분석한다. MSA로 구현된 시스템일 경우 트래픽 분석을 통해 장애가 발생한 서비스를 특정한다.
3. 분석된 원인에 따라 즉시 대응한다.
4. 장애 원인 예시 : AWS 서비스 장애, DB 연결 부족, OOM 발생, 외부 API 토큰 만료 등
5. 추후 재발 방지한다. </br></br>
   
**트래픽 급증 시**
1. 트래픽이 이상적으로 급증할시, DDos 공격인지 확인하고 대응한다.
2. 비즈니스관련 이벤트로 인한 급증시, 부하가 감지되는 리소스들을 Scale out 한다. </br></br>





