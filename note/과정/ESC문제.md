# ECS 태스크 배포 문제 진단 

### 문제 1: Context Path 불일치로 인한 라우팅 실패

**현재 설정:**

**1) Spring Boot 애플리케이션 설정** (application.yml)
```yaml
# customers-service
spring:
  server:
    servlet:
      context-path: /customers-service  # 실제 경로

# vets-service  
spring:
  server:
    servlet:
      context-path: /vets-service       # 실제 경로
```

**2) ECS 모듈 컨테이너 환경 변수** (ecs.tf:68-73)
```hcl
environment_variables = {
  "SPRING_PROFILES_ACTIVE" = "mysql,aws",
  "SERVER_SERVLET_CONTEXT_PATH" = each.value.context_path,  # "/customers", "/vets" 등
  "MANAGEMENT_HEALTH_PROBES_ENABLED" = "true"
}
```
- `each.value.context_path = "/customers"` (서비스 이름에서 `-service` 제거)

**3) ALB 리스너 규칙** (modules/ecs/main.tf:40-45)
```hcl
condition {
  path_pattern {
    values = ["/${var.context_path}*"]  # "/customers*"
  }
}
```

**4) ALB 타겟 그룹 헬스 체크** (modules/ecs/main.tf:15-23)
```hcl
health_check {
  path = "${var.context_path}/actuator/health"  # "/customers/actuator/health"
  # ...
}
```

**문제점:**
```
API Gateway 요청: /api/customers
      ↓
ALB 라우팅 (패턴: /customers*)
      ↓
ECS 컨테이너 (Context Path: /customers)
      ↓
Spring Boot 실제 경로: /customers-service
      ↓
❌ 404 Not Found (경로 불일치!)
```

**실제 애플리케이션 경로:**
- `/customers-service/actuator/health` (실제 존재)
- `/customers-service/api/...` (실제 존재)

**ALB가 체크하는 경로:**
- `/customers/actuator/health` (존재하지 않음!)

**결과:**
- ✅ ALB 리스너 규칙은 올바르게 작동 (요청을 타겟 그룹으로 전달)
- ❌ 헬스 체크 실패 → ECS 태스크가 unhealthy 상태
- ❌ 실제 애플리케이션 요청도 404 반환

---

### 문제 2: 보안 그룹 포트 제한

**보안 그룹 설정** (modules/sg/main.tf:53-60)
```hcl
resource "aws_security_group" "app" {
  ingress {
    protocol        = "tcp"
    from_port       = 8080
    to_port         = 8080
    security_groups = [var.alb_source_security_group_id]
    description     = "HTTP traffic on port 8080 from ALB"
  }
}
```

**실제 컨테이너 포트** (Parameter Store에서 동적으로 가져옴)
- ecs.tf:13-16에서 `/petclinic/dev/${path_name}/server.port` 조회
- 예상 값: 8080

**잠재적 문제:**
1. **Parameter Store에 다른 포트가 저장되어 있을 경우**
   - 예: customers-service가 8081로 설정되어 있다면
   - ALB → ECS 통신이 보안 그룹에서 차단됨

2. **검증 필요:**
   ```bash
   # AWS Parameter Store 확인 필요
   aws ssm get-parameter --name "/petclinic/dev/customers/server.port"
   aws ssm get-parameter --name "/petclinic/dev/vets/server.port"
   aws ssm get-parameter --name "/petclinic/dev/visits/server.port"
   aws ssm get-parameter --name "/petclinic/dev/admin/server.port"
   ```

**권장 사항:**
- 옵션 A: 모든 서비스를 8080 포트로 통일
- 옵션 B: 보안 그룹을 포트 범위로 확장 (8080-8089)
- 옵션 C: 동적으로 보안 그룹 규칙 생성 (복잡함)

---

### 문제 3: API Gateway와 ALB 경로 매핑 복잡성

**현재 API Gateway 라우팅** (modules/api-gateway/main.tf:129)
```hcl
# /api/customers -> http://ALB/customers-service/customers
uri = "http://${var.alb_dns_name}/${each.value}/${each.key}"
```
- `each.value` = "customers-service"
- `each.key` = "customers"
- 결과 URI: `/customers-service/customers`

**ALB 리스너 규칙 패턴** (modules/ecs/main.tf:43)
```hcl
values = ["/${var.context_path}*"]  # "/customers*"
```

**문제:**
```
API Gateway: /api/customers 요청
      ↓
API Gateway → ALB: /customers-service/customers
      ↓
ALB 리스너 규칙: "/customers*" 패턴 확인
      ↓
❌ "/customers-service/customers"는 "/customers*" 패턴과 매치되지 않음!
      ↓
ALB Default Action: 404 "Cannot route request."
```

**실제 동작:**
- API Gateway가 `/customers-service/customers`로 요청
- ALB는 `/customers*` 패턴을 찾음
- **매치 실패** → 404 응답

**필요한 수정:**
1. **ALB 리스너 규칙 패턴 수정**
   ```hcl
   # 현재
   values = ["/${var.context_path}*"]  # "/customers*"
   
   # 수정안 1: 서비스 이름 전체 사용
   values = ["/${var.service_name}*"]  # "/customers-service*"
   
   # 수정안 2: 두 패턴 모두 허용
   values = ["/${var.context_path}*", "/${var.service_name}*"]
   ```

2. **또는 API Gateway URI 수정**
   ```hcl
   # 현재
   uri = "http://${var.alb_dns_name}/${each.value}/${each.key}"
   # /customers-service/customers
   
   # 수정안: context_path만 사용
   uri = "http://${var.alb_dns_name}/${each.key}"
   # /customers
   ```

---

### 문제 4: Spring Boot Context Path 이중 설정

**Environment Variables** (ecs.tf:70)
```hcl
"SERVER_SERVLET_CONTEXT_PATH" = each.value.context_path  # "/customers"
```

**Application YAML** (application.yml:5-6)
```yaml
spring:
  server:
    servlet:
      context-path: /customers-service
```

**충돌 시나리오:**
1. ECS 환경 변수가 우선순위가 더 높을 수 있음
2. `/customers`로 덮어쓰기 시도
3. 그러나 application.yml의 `config.import`가 Parameter Store를 참조하면 복잡해짐

**결과:**
- 애플리케이션이 실제로 어떤 context path를 사용하는지 불명확
- 런타임 로그를 확인해야 함

**권장 사항:**
- **통일된 접근 방식 선택**
  - 옵션 A: application.yml에서 context-path 제거, 환경 변수에만 의존
  - 옵션 B: 환경 변수에서 `SERVER_SERVLET_CONTEXT_PATH` 제거, application.yml에만 의존

---

### 문제 5: admin-server DB 접근 설정

**현재 설정** (ecs.tf:4)
```hcl
"admin-server" = { priority = 100, path_name = "admin", needs_db = false }
```

**Secrets 조건부 설정** (ecs.tf:61-66)
```hcl
secrets_variables = each.value.needs_db ? {
  "SPRING_DATASOURCE_PASSWORD" = "...",
  "SPRING_DATASOURCE_URL"      = "...",
  "SPRING_DATASOURCE_USERNAME" = "..."
} : {}
```

**문제:**
- admin-server가 실제로 DB가 필요한지 확인 필요
- Spring Boot Admin은 일반적으로 DB가 필요하지 않음 (메모리 기반)
- 그러나 persistence가 활성화되어 있다면 DB 필요

**Environment Variables** (ecs.tf:68-73)
```hcl
environment_variables = {
  "SPRING_PROFILES_ACTIVE" = "mysql,aws",  # mysql 프로파일 활성화!
  # ...
}
```

**충돌:**
- `needs_db = false`로 설정했지만
- `SPRING_PROFILES_ACTIVE = "mysql,aws"`로 MySQL 프로파일 활성화
- admin-server가 MySQL 설정을 찾으려 하지만 secrets가 없음
- **애플리케이션 시작 실패 가능**

**권장 사항:**
```hcl
# 옵션 1: admin-server는 DB 사용하지 않음
"admin-server" = { 
  priority = 100, 
  path_name = "admin", 
  needs_db = false,
  profiles = "aws"  # mysql 프로파일 제외
}

# 옵션 2: admin-server도 DB 사용
"admin-server" = { 
  priority = 100, 
  path_name = "admin", 
  needs_db = true,
  profiles = "mysql,aws"
}
```

---

### 문제 6: 헬스 체크 타임아웃 및 임계값

**현재 타겟 그룹 헬스 체크** (modules/ecs/main.tf:15-23)
```hcl
health_check {
  path                = "${var.context_path}/actuator/health"
  protocol            = "HTTP"
  matcher             = "200"
  interval            = 30
  timeout             = 10
  healthy_threshold   = 2
  unhealthy_threshold = 3
}
```

**ECS 서비스 헬스 체크 유예 기간** (modules/ecs/main.tf:118)
```hcl
health_check_grace_period_seconds = var.health_check_grace_period  # default: 150
```

**분석:**
- ✅ 설정은 대체로 적절함
- ✅ 150초 유예 기간은 Spring Boot 시작 시간을 고려하면 적절
- ⚠️ 그러나 context path 문제로 인해 헬스 체크 자체가 실패함

**현재 헬스 체크 경로:**
```
/customers/actuator/health
```

**실제 필요한 경로:**
```
/customers-service/actuator/health
```

---

## 문제 우선순위 및 영향도

| 순위 | 문제 | 심각도 | 영향 | 수정 복잡도 |
|------|------|--------|------|------------|
| 1 | Context Path 불일치 | 🔴 치명적 | 모든 요청 실패, 헬스 체크 실패 | 중간 |
| 2 | API Gateway-ALB 경로 매핑 | 🔴 치명적 | 라우팅 실패 | 낮음 |
| 3 | Context Path 이중 설정 | 🟡 높음 | 예측 불가능한 동작 | 낮음 |
| 4 | admin-server DB 설정 충돌 | 🟡 높음 | 서비스 시작 실패 | 낮음 |
| 5 | 보안 그룹 포트 제한 | 🟢 중간 | Parameter Store 값에 따라 문제 가능 | 낮음 |
| 6 | 헬스 체크 설정 | 🟢 낮음 | context path 수정 후 해결됨 | 없음 |

---

## 🔧 권장 해결 방안

### 해결 방안 A: Context Path 통일 (권장)

**목표:** Spring Boot 애플리케이션의 실제 context path와 모든 설정을 일치시킴

**1단계: Spring Boot 애플리케이션 수정**
```yaml
# 모든 서비스의 application.yml
spring:
  server:
    servlet:
      context-path: /${SERVICE_NAME}  # 예: /customers (not /customers-service)
```

**2단계: 환경 변수 확인**
```hcl
# ecs.tf - 이미 올바름
environment_variables = {
  "SERVER_SERVLET_CONTEXT_PATH" = each.value.context_path,  # "/customers"
}
```

**3단계: ALB 리스너 규칙 확인**
```hcl
# modules/ecs/main.tf - 이미 올바름
condition {
  path_pattern {
    values = ["/${var.context_path}*"]  # "/customers*"
  }
}
```

**4단계: 헬스 체크 경로 확인**
```hcl
# modules/ecs/main.tf - 이미 올바름
health_check {
  path = "${var.context_path}/actuator/health"  # "/customers/actuator/health"
}
```

**5단계: API Gateway 수정**
```hcl
# modules/api-gateway/main.tf:129
# 현재
uri = "http://${var.alb_dns_name}/${each.value}/${each.key}"

# 수정
uri = "http://${var.alb_dns_name}/${each.key}"
# /api/customers -> http://ALB/customers
```

**결과:**
```
API Gateway: /api/customers
      ↓
ALB: /customers (리스너 규칙 "/customers*" 매치)
      ↓
ECS: context-path = /customers
      ↓
Spring Boot: /customers/actuator/health ✅
```

---

### 해결 방안 B: ALB 패턴 수정 (대안)

**목표:** Spring Boot의 기존 context path를 유지하고 ALB 패턴만 수정

**1단계: ALB 리스너 규칙 수정**
```hcl
# modules/ecs/main.tf
# 현재
condition {
  path_pattern {
    values = ["/${var.context_path}*"]  # "/customers*"
  }
}

# 수정
locals {
  service_path = "${var.service_name}"  # "customers-service"
}

condition {
  path_pattern {
    values = ["/${local.service_path}*"]  # "/customers-service*"
  }
}
```

**2단계: 헬스 체크 경로 수정**
```hcl
health_check {
  path = "/${local.service_path}/actuator/health"  # "/customers-service/actuator/health"
}
```

**3단계: 환경 변수 제거**
```hcl
# ecs.tf
environment_variables = {
  "SPRING_PROFILES_ACTIVE" = "mysql,aws",
  # "SERVER_SERVLET_CONTEXT_PATH" 제거 (application.yml 사용)
  "MANAGEMENT_HEALTH_PROBES_ENABLED" = "true"
}
```

**4단계: API Gateway는 그대로**
```hcl
# 이미 올바른 형식
uri = "http://${var.alb_dns_name}/${each.value}/${each.key}"
# /customers-service/customers
```

---

### 해결 방안 C: 하이브리드 접근 (복잡, 권장하지 않음)

두 패턴 모두 허용하도록 ALB 설정
```hcl
condition {
  path_pattern {
    values = [
      "/${var.context_path}*",           # "/customers*"
      "/${var.service_name}*"            # "/customers-service*"
    ]
  }
}
```

---

## ✅ 추천 해결 방안: **방안 A (Context Path 통일)**

**이유:**
1. **단순성**: 모든 레이어에서 일관된 경로 사용
2. **RESTful**: `/customers`, `/vets` 등 깔끔한 API 경로
3. **유지보수**: 이해하기 쉽고 디버깅 용이
4. **확장성**: 새 서비스 추가 시 패턴 명확

**단점:**
- Spring Boot 애플리케이션 코드 수정 필요
- 기존 로컬 개발 환경 영향 가능

---

## 🔍 추가 검증 필요 사항

### 1. Parameter Store 포트 확인
```bash
aws ssm get-parameter --name "/petclinic/dev/customers/server.port"
aws ssm get-parameter --name "/petclinic/dev/vets/server.port"
aws ssm get-parameter --name "/petclinic/dev/visits/server.port"
aws ssm get-parameter --name "/petclinic/dev/admin/server.port"
```
- 모든 값이 `8080`인지 확인
- 다른 포트 사용 시 보안 그룹 수정 필요

### 2. ECS 태스크 로그 확인
```bash
# CloudWatch Logs 확인
/ecs/petclinic/customers-service
/ecs/petclinic/vets-service
/ecs/petclinic/visits-service
/ecs/petclinic/admin-server
```
- Spring Boot 시작 로그
- 실제 사용 중인 context path
- 에러 메시지

### 3. ALB 타겟 그룹 상태 확인
```bash
# AWS Console 또는 CLI
aws elbv2 describe-target-health --target-group-arn <arn>
```
- 타겟이 `healthy` 또는 `unhealthy`?
- unhealthy 이유 확인

### 4. 보안 그룹 규칙 확인
```bash
# ALB 보안 그룹
aws ec2 describe-security-groups --group-ids <alb-sg-id>

# App 보안 그룹  
aws ec2 describe-security-groups --group-ids <app-sg-id>
```
- ALB → App 연결 확인
- 포트 8080 허용 확인

### 5. API Gateway 테스트
```bash
# API Gateway 엔드포인트 직접 호출
curl -v https://<api-id>.execute-api.ap-northeast-2.amazonaws.com/dev/api/customers

# ALB 직접 호출 (비교용)
curl -v http://<alb-dns-name>/customers
```

---

## 📝 수정 체크리스트

### Phase 1: 필수 수정 (치명적 문제 해결)

- [ ] **Spring Boot application.yml 수정**
  - [ ] customers-service: context-path를 `/customers`로 변경
  - [ ] vets-service: context-path를 `/vets`로 변경
  - [ ] visits-service: context-path를 `/visits`로 변경
  - [ ] admin-server: context-path를 `/admin`로 변경

- [ ] **API Gateway 라우팅 수정**
  - [ ] `modules/api-gateway/main.tf:129` - URI 패턴 수정
  - [ ] `modules/api-gateway/main.tf:160` - URI 패턴 수정

- [ ] **admin-server 설정 수정**
  - [ ] `needs_db` 값 확인 및 조정
  - [ ] `SPRING_PROFILES_ACTIVE` 조정

### Phase 2: 검증

- [ ] **Parameter Store 확인**
  - [ ] 모든 서비스 포트가 8080인지 확인

- [ ] **Terraform Plan 실행**
  ```bash
  cd terraform/envs/dev/application
  terraform plan
  ```

- [ ] **Terraform Apply 실행**
  ```bash
  terraform apply
  ```

### Phase 3: 배포 후 검증

- [ ] **ECS 서비스 상태 확인**
  - [ ] 모든 태스크가 RUNNING 상태
  - [ ] 원하는 개수와 실행 중인 개수 일치

- [ ] **타겟 그룹 헬스 체크**
  - [ ] 모든 타겟이 healthy 상태

- [ ] **API 테스트**
  - [ ] `/api/customers` 응답 확인
  - [ ] `/api/vets` 응답 확인
  - [ ] `/api/visits` 응답 확인
  - [ ] `/admin` 응답 확인

- [ ] **로그 확인**
  - [ ] CloudWatch Logs에 에러 없음
  - [ ] 애플리케이션 정상 시작 확인

---

## 🎯 결론

**핵심 문제:**
1. **Context Path 불일치** - Spring Boot 앱은 `/customers-service`를 사용하지만, ALB는 `/customers`를 체크
2. **API Gateway-ALB 경로 매핑** - `/customers-service/customers`가 `/customers*` 패턴과 매치되지 않음

**가장 간단한 해결책:**
- **방안 A 채택**: 모든 서비스의 context path를 짧은 형식(`/customers`)으로 통일
- API Gateway에서 ALB로 전달할 때 경로 단순화
- 일관되고 예측 가능한 라우팅


**위험도:**
- 🟢 낮음 (설정 변경만 필요, 코드 로직 변경 없음)