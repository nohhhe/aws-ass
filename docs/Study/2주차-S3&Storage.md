# W2. S3 & Storage (S3, EFS, FSx, Snow Family)

---

## 1. Amazon S3

### 1.1 S3란?

- HTTP/HTTPS REST API로 접근하는 객체 스토리지 서비스
- 버킷과 객체 2가지로 구성
- 내구성 99.999999999% — 💡 1억 개 객체를 저장하면 1만 년에 1개 손실될 확률. 사실상 데이터가 유실되지 않는다는 의미
- 가용성 99.99% (Standard 기준) — 💡 1년 중 약 52분만 접근 불가. 서비스가 거의 항상 떠 있다는 의미
- key-value 저장소이며 폴더 개념이 없음 — 💡 콘솔에서 폴더처럼 보이지만, 실제로는 키 이름에 `/`가 포함된 것일 뿐 (예: `images/photo.jpg`는 "images 폴더 안의 photo.jpg"가 아니라, 키 이름 자체가 `images/photo.jpg`인 하나의 객체)
- 무제한 스토리지: 버킷 크기 제한 없음, 객체 수 제한 없음

### 1.2 버킷

- 객체를 담는 컨테이너, 모든 객체는 반드시 어떤 버킷에 포함
- 이름은 전 세계에서 유일해야 함 — 누군가 이미 `my-data-bucket`이라는 이름을 사용 중이면, 다른 계정에서 같은 이름으로 버킷을 생성할 수 없음 (에러 발생). AWS 전체에서 버킷 이름은 하나의 네임스페이스를 공유하기 때문. 💡 그래서 보통 `회사명-프로젝트-환경` 같은 접두사를 붙여서 충돌을 피함 (예: `acme-logs-prod`)
- 💡 이름 규칙: 소문자/숫자/하이픈만, 3~63자, IP 형태 금지
- 💡 계정당 100개 기본 한도 (서비스 한도 증가 요청으로 최대 1,000개)
- 리전에 귀속: 버킷 생성 시 리전 지정. 한 번 만들면 다른 리전으로 이동 불가 → 데이터를 다른 리전에도 두고 싶으면 복제 기능을 사용해야 함 (1.10절에서 설명)
- 강한 일관성 (Strong Consistency) — 💡 S3에 데이터를 쓰거나 삭제한 직후, 다른 곳에서 읽어도 항상 최신 상태가 반영됨:
  - 새 객체를 넣자마자 바로 읽을 수 있음
  - 덮어쓰기/삭제 직후에도 즉시 최신 상태 반영
  - 추가 비용/성능 저하 없이 자동 적용

### 1.3 객체

- 구성: 키, 버전 ID, 값, 메타데이터
- 키: 객체의 전체 경로 이름 (하나의 문자열)
  - 예: `s3://my-bucket/2024/summer/beach.jpg` → 키는 `2024/summer/beach.jpg`
- 💡 URL 형식:
  - Virtual-hosted style: `https://bucket-name.s3.region.amazonaws.com/key` (기본, 권장)
  - Path style: `https://s3.region.amazonaws.com/bucket-name/key` (레거시)
- 크기: 최소 0 byte ~ 최대 5TB
- 단일 PUT 최대 5GB, 그 이상은 Multipart Upload 필수 (100MB 이상 권장)
- 태그: 💡 객체에 `키=값` 형태의 라벨을 최대 10개 붙일 수 있음 (예: `Environment=Production`, `Department=Finance`). 태그를 붙여두면:
  - IAM 정책에서 "이 태그가 붙은 객체만 접근 허용/거부" 같은 조건을 걸 수 있음
  - Lifecycle 규칙에서 "이 태그가 붙은 객체만 30일 후 Glacier로 이동" 같은 조건을 걸 수 있음
  - 비용 할당: AWS 청구서에서 태그별로 비용을 분류해서 "Finance 팀이 S3에 얼마 썼는지" 추적 가능
- 메타데이터: 💡 객체에 대한 부가 정보. HTML의 `<meta>` 태그와 비슷한 개념으로, 데이터 자체가 아니라 데이터를 설명하는 정보
  - 시스템 메타데이터: S3가 자동으로 설정 (Content-Type, Content-Length, Last-Modified 등). 예를 들어 `.jpg` 파일을 올리면 Content-Type이 `image/jpeg`로 설정되어 브라우저가 이미지로 인식
  - 사용자 정의 메타데이터: `x-amz-meta-` 접두사를 붙여서 원하는 정보를 저장. 예: `x-amz-meta-author=홍길동`. 검색이나 필터링에 활용 가능

### 1.4 스토리지 클래스 7종

S3에 저장하는 모든 객체는 7가지 클래스 중 하나에 속한다. 핵심 원리는 간단하다: 자주 꺼내볼수록 저장 비용이 비싸고, 거의 안 꺼내볼수록 저장 비용이 싸다. 대신 저장 비용이 싼 클래스는 꺼낼 때 비용이 비싸거나 시간이 오래 걸린다.

> 💡 Glacier가 이름에 붙는 클래스(Glacier Instant / Glacier Flexible / Glacier Deep Archive)는 원래 "Amazon Glacier"라는 별도의 아카이브 저장 서비스였는데, 나중에 S3에 통합되면서 S3 스토리지 클래스의 일부가 되었다. Glacier = 빙하라는 뜻으로, "꽁꽁 얼려놓고 거의 안 꺼내는 데이터"라는 의미. 그래서 Glacier가 붙으면 "장기 보관용, 꺼내는 데 시간이 걸림"이라고 생각하면 됨.

💡 비용 구조 — S3 요금은 3가지로 구성됨:
- 저장 비용: 데이터를 보관하는 데 GB당 월 얼마 (클래스별로 다름)
- 요청 비용: PUT/GET 등 API를 호출할 때마다 건당 비용
- 검색 비용: IA/Glacier 클래스에서 데이터를 꺼낼 때 GB당 추가 비용 (Standard는 검색 비용 없음)

💡 클래스별 비용 비교 (서울 리전 기준, 단위: 달러):

| 클래스 | 저장 (GB/월) | GET (1,000건) | 검색 (GB당) | 최소 보관 | AZ |
|------|---------|----------|---------|--------|-----|
| Standard | 0.025 | 0.0004 | 없음 | 없음 | ≥3 |
| Intelligent-Tiering | 0.025~자동 | 0.0004 | 없음 | 없음 | ≥3 |
| Standard-IA | 0.0138 | 0.001 | 0.01 | 30일 | ≥3 |
| One Zone-IA | 0.011 | 0.001 | 0.01 | 30일 | 1 |
| Glacier Instant | 0.005 | 0.01 | 0.03 | 90일 | ≥3 |
| Glacier Flexible | 0.0045 | 0.05 | 0.01~0.03 | 90일 | ≥3 |
| Glacier Deep Archive | 0.002 | 0.05 | 0.02~0.05 | 180일 | ≥3 |

> 💡 비용 테이블 읽는 법: Standard는 1GB를 한 달 저장하는 데 약 0.025 달러(약 33원). Deep Archive는 0.002 달러(약 2.6원)로 Standard의 약 12분의 1. 대신 Deep Archive에서 데이터를 꺼내려면 검색 비용이 GB당 0.02~0.05 달러가 추가로 붙고, GET 요청 비용도 Standard의 125배. 저장만 싸고 꺼내는 건 비싸다는 게 핵심 트레이드오프.
>
> 시험에서는 정확한 금액을 묻지 않고, "어떤 클래스가 가장 비용 효율적인가"를 시나리오로 물어봄. 금액 자체보다 클래스 간 상대적 관계를 이해하는 게 중요.

> 💡 최소 보관 기간이란: Standard 외의 클래스에는 "최소 이 기간 동안은 저장한 것으로 간주하고 요금을 받겠다"는 규칙이 있다. Standard-IA의 최소 보관은 30일인데, 이 말은 객체를 넣고 3일 만에 삭제하더라도 "30일 동안 저장한 것"으로 계산해서 30일분 저장 요금이 청구된다는 뜻. 실제로 3일만 썼어도 30일분을 내야 함. Glacier Deep Archive는 180일이므로, 1주일 쓰고 삭제해도 180일분 요금이 나옴. 이 때문에 "싸다고 잠깐 넣었다 빼는" 용도에는 Standard가 낫다.

💡 어떤 상황에서 어떤 클래스를 쓰나:

| 상황 | 클래스 | 실무 사례 |
|----|------|--------|
| 매일 접근한다 | Standard | 웹 앱 이미지, API 응답 데이터, 활성 로그 |
| 접근 패턴을 예측할 수 없다 | Intelligent-Tiering | 신규 서비스 데이터, 시즌성 콘텐츠 |
| 월 1~2회, 즉시 접근 필요 | Standard-IA | 재해 복구용 백업, 월 1회 보는 보고서 |
| IA인데 손실돼도 괜찮다 | One Zone-IA | 썸네일 복제본, 재생성 가능한 캐시 |
| 분기별 접근, 즉시 필요 | Glacier Instant | 분기별 감사 자료, 의료 영상 아카이브 |
| 몇 분~시간 기다려도 됨 | Glacier Flexible | 연 1~2회 열어보는 과거 계약서, 법적 문서 |
| 거의 안 봄, 규정상 장기 보관 | Glacier Deep Archive | 7~10년 의무 보관 금융 기록, 방송 원본 |

---

**Intelligent-Tiering — 자동 비용 최적화 클래스**

💡 다른 6개 클래스는 사용자가 접근 패턴을 직접 판단해서 골라야 한다. 하지만 현실에서는 예측이 어려운 경우가 많다:
- 처음엔 매일 접근하다가 3개월 뒤엔 아무도 안 보는 데이터
- 언제 갑자기 다시 접근할지 모르는 데이터

이런 상황을 위해 만들어진 게 Intelligent-Tiering이다. S3가 접근 패턴을 모니터링하면서 자동으로 가장 싼 단계로 옮겨주고, 다시 접근하면 자동으로 복귀.

Intelligent-Tiering 안에는 5개 단계가 있다:

| 단계 | 조건 | 꺼내는 시간 | 활성화 |
|-----|-----|---------|------|
| Frequent Access | 기본 (활발히 접근 중) | 즉시 | 자동 |
| Infrequent Access | 30일 연속 미접근 | 즉시 | 자동 |
| Archive Instant Access | 90일 연속 미접근 | 즉시 | 자동 |
| Archive Access | 90~180일 미접근 | 3~5시간 | 수동 활성화 필요 |
| Deep Archive Access | 180일+ 미접근 | 12시간 | 수동 활성화 필요 |

장단점:
- 장점: 검색 비용 0원 (다른 IA/Glacier는 꺼낼 때마다 검색 비용 발생), 자동 최적화, 최소 보관 기간 없음
- 비용: 객체당 월 소액의 모니터링 비용 발생. 128KB 미만 객체는 항상 Frequent Access에 유지되어 모니터링 비용도 안 붙음
- 시험 키워드: "접근 패턴을 예측할 수 없다" → Intelligent-Tiering이 정답

---

**Glacier Flexible Retrieval 복원 옵션** (시험 빈출) — 💡 같은 Glacier 클래스라도 급할수록 비용이 비쌈:

| 옵션 | 소요 시간 | GB당 복원 비용 (대략, 달러) | 언제 쓰나 |
|-----|---------|----------------|--------|
| Expedited | 1~5분 | ~0.03 | 긴급하게 꺼내야 할 때 |
| Standard | 3~5시간 | ~0.01 | 일반적인 복원 |
| Bulk | 5~12시간 | ~0.0025 | 대량 데이터를 싸게 복원할 때 |

> 💡 Expedited는 Bulk 대비 약 12배 비쌈. 시험에서는 "가장 비용 효율적으로 복원"하라고 하면 Bulk, "즉시 필요"하면 Expedited가 답.
>
> 💡 Provisioned Capacity란: Expedited 복원은 피크 시간에 AWS 자원이 부족하면 거절될 수 있는데, 미리 "나는 Expedited를 쓸 거야"라고 예약(구매)해놓으면 거절 없이 확실하게 사용 가능. 월 정액제로 예약하는 것.

Glacier Deep Archive 복원 옵션:

| 옵션 | 소요 시간 | 비용 |
|-----|---------|-----|
| Standard | 12시간 | 저렴 |
| Bulk | 48시간 | 가장 저렴 |

---

**IA 클래스 주의사항**:

- 최소 128KB 과금 — 💡 Standard-IA / One Zone-IA는 실제 파일이 1KB든 50KB든 128KB로 간주하고 요금을 매김. 예를 들어 10KB 파일 1,000개를 Standard-IA에 넣으면, 실제로는 10MB인데 128MB분 요금이 청구됨 (12.8배 손해). 작은 파일이 많으면 Standard가 오히려 저렴
- 검색 비용 별도 발생 — 💡 객체를 꺼낼 때(GET)마다 GB당 0.01 달러 비용이 붙음. 접근 빈도가 높으면 저장 비용 절감분보다 검색 비용이 더 클 수 있어서, Standard보다 비싸질 수 있음
- One Zone-IA: 1개 AZ에만 저장 → AZ 장애 시 데이터 손실. 재생성 가능한 데이터(썸네일, 로그 복제본)에만 사용

### 1.5 Lifecycle 관리

위에서 정리한 스토리지 클래스를 시간 경과에 따라 자동으로 전환하거나, 오래된 객체를 자동 삭제하는 규칙이다. 수동으로 하나하나 옮길 필요 없이, 한 번 규칙을 설정해두면 S3가 알아서 처리한다.

**💡 어떻게 설정하나?**
- AWS 콘솔 → S3 → 버킷 선택 → **관리(Management)** 탭 → **수명 주기 규칙 생성**
- 또는 AWS CLI, CloudFormation, Terraform 등으로 설정 가능
- 규칙 설정 시 아래 3가지를 지정:
  1. 적용 범위: 버킷 전체, 특정 접두사(키가 `logs/`로 시작하는 객체만 등), 또는 특정 태그가 붙은 객체만
  2. 액션: 전환 또는 삭제
  3. 시점: 업로드 후 며칠 뒤에 실행할지

설정 예시 — "로그 파일은 30일 후 IA, 90일 후 Glacier, 1년 후 삭제":
```
규칙 이름: log-lifecycle
적용 범위: 접두사 = logs/
액션 1: 30일 후 → Standard-IA로 전환
액션 2: 90일 후 → Glacier Flexible로 전환
액션 3: 365일 후 → 영구 삭제
```
이렇게 하면 `logs/` 아래 객체들은 업로드 후 자동으로 30일 → IA → 90일 → Glacier → 365일 → 삭제가 진행됨

주요 액션 4가지:
- Transition (전환): 더 싼 클래스로 이동 (Standard → IA → Glacier → Deep Archive)
- Expiration (만료): 기간 지나면 영구 삭제
- NoncurrentVersionTransition/Expiration (이전 버전 전환/만료): Versioning 켠 버킷에서 이전 버전만 대상으로 전환/삭제 (현재 버전은 건드리지 않음). Versioning의 비용 폭증을 막는 수단
- AbortIncompleteMultipartUpload (미완료 멀티파트 업로드 중단): 💡 대용량 파일을 Multipart Upload(1.13절)로 올리다가 중간에 실패하거나 취소하면, 이미 올라간 파트 조각들이 S3에 남아서 저장 비용이 계속 발생한다. 이 규칙을 설정하면 "업로드 시작 후 N일이 지나도 완료되지 않은 멀티파트 업로드의 파트 조각을 자동 삭제"해줌. 설정 안 하면 모르는 사이에 비용이 쌓일 수 있음

제약 사항:
- 전환 방향 제약: "더 저렴한 방향"으로만 전환 가능, 역방향은 불가 → Glacier에서 Standard로 되돌리려면 객체를 복사해서 새 클래스에 넣어야 함
- Standard → IA 계열 전환: 최소 30일 경과 후 가능
- IA → Glacier 계열 전환: 추가 30일 경과 필요 (Standard → Glacier까지 최소 60일)

시험 단골 시나리오: "30일 → IA, 90일 → Glacier Flexible, 365일 → Deep Archive"

### 1.6 Versioning (버전 관리)

같은 키에 파일을 덮어쓰거나 삭제해도 이전 버전을 자동 보관하는 기능. 실수로 삭제하거나 덮어쓴 경우 이전 버전으로 복구할 수 있다.

💡 Versioning 자체는 무료인가?
- Versioning 기능을 켜는 것 자체에는 추가 비용이 없다
- 하지만 모든 이전 버전이 S3에 그대로 남아 있으므로, 각 버전이 독립적으로 저장 비용을 먹음
- 예: 10MB 파일을 5번 덮어쓰면 → 50MB분의 저장 비용 발생 (최신 10MB + 이전 4개 버전 40MB)
- 삭제한 객체도 Delete Marker만 붙을 뿐 이전 버전은 남아 있으므로 비용이 계속 발생
- 이 문제를 해결하는 방법: 1.5절의 Lifecycle 규칙에서 NoncurrentVersionExpiration을 설정해서 "이전 버전은 30일 후 자동 삭제" 같은 규칙을 걸어야 함. Versioning을 켜면 이 Lifecycle 규칙도 함께 설정하는 것이 사실상 필수

기본 동작:
- 한 번 켜면 끌 수 없음 (일시 중지만 가능, 일시 중지 시 새 객체의 버전 ID는 null)
- Versioning이 꺼져 있을 때 업로드된 객체의 버전 ID는 "null"
- DELETE 동작: 최신 버전 위에 Delete Marker(삭제 표시)를 추가할 뿐, 이전 버전은 그대로 남음. 겉보기에는 삭제된 것처럼 보이지만 실제로는 숨겨진 것
- 복구: Delete Marker를 삭제하면 이전 버전이 즉시 노출
- 영구 삭제: 특정 버전 ID를 지정해 DELETE — 되돌릴 수 없음

MFA (Multi-Factor Authentication) Delete: 영구 삭제 + Versioning 일시 중지 시 MFA 인증을 요구하는 추가 보호 장치. 루트 계정 CLI로만 설정 가능

> 참고 자료 정정 — Versioning 일시 중지 동작
> 기존 문서에서는 "일시 중지하면 새 객체의 버전만 생성 안 됨"으로 간략히 서술했으나, 정확하게는 일시 중지 상태에서 업로드된 객체는 버전 ID가 `null`로 설정되며, 기존 `null` 버전이 있으면 덮어씀. 이전 버전들(실제 버전 ID를 가진 것)은 그대로 보존됨.
> 참고: https://docs.aws.amazon.com/AmazonS3/latest/userguide/AddingObjectstoVersionSuspendedBuckets.html

### 1.7 암호화

💡 왜 S3에서 암호화가 필요한가?

S3에 저장된 데이터는 접근 제어(IAM, Bucket Policy)로 보호하지만, 암호화는 그것과 별개의 보호 계층이다. 접근 제어가 "문 잠그기"라면, 암호화는 "금고에 넣기"와 같다.

암호화가 필요한 이유:
- AWS 내부자 위험: AWS 직원이 물리 디스크에 접근하더라도 암호화된 데이터는 읽을 수 없음
- 규정 준수: 금융, 의료, 공공기관 등에서는 "저장 데이터 암호화" 자체가 법적 의무인 경우가 많음
- 디스크 폐기/교체 시: AWS가 물리 하드디스크를 교체·폐기할 때 데이터가 노출되는 것을 방지
- 계정 유출 시 피해 최소화: 계정이 탈취되면 접근 제어는 뚫리지만, SSE-KMS를 쓰면 KMS 키 권한이 별도로 필요하므로 추가 방어선이 됨

결국 "접근 권한이 뚫리더라도 데이터 자체는 암호화되어 있어서 내용을 볼 수 없게" 하는 것이 목적이다.

---

암호화는 크게 2가지로 나뉨:
- 저장 시 암호화 (Encryption at Rest) — S3 디스크에 저장될 때 암호화. SSE 또는 CSE를 사용
- 전송 중 암호화 (Encryption in Transit) — 네트워크를 통해 데이터가 이동할 때 암호화. HTTPS 사용. 버킷 정책에서 `aws:SecureTransport: false`를 DENY하면 HTTPS를 강제할 수 있음

저장 시 암호화 옵션 — SSE (Server-Side Encryption) 4가지 + CSE:

| 옵션 | 풀네임 | 누가 키를 관리하나 | 왜 이걸 쓰나 |
|-----|-------|------------|---------|
| SSE-S3 | SSE with S3-managed keys | AWS가 알아서 관리 | "그냥 암호화해줘, 신경 쓰기 싫다" — 가장 간단. 무료. 모든 새 객체에 자동 적용. 대부분 이걸로 충분 |
| SSE-KMS | SSE with KMS keys | 고객이 KMS에서 키를 생성/관리 | "누가 언제 이 키를 사용했는지 감사 기록이 필요하다" — CloudTrail에 기록이 남음. 키에 별도 권한을 걸 수 있어서 S3 접근 권한이 있어도 KMS 키 권한이 없으면 복호화 불가 |
| SSE-C | SSE with Customer-provided keys | 고객이 매 요청마다 키를 직접 제공 | "우리 키를 AWS에 맡기지 않겠다" — AWS는 키를 저장하지 않으며, 키 분실 시 데이터 복구 불가 |
| DSSE-KMS | Dual-layer SSE with KMS keys | 고객 KMS 키로 2중 암호화 | "암호화를 2번 해야 한다는 규정이 있다" — 군사, 정부 등 초고수준 규정. 거의 안 씀 |
| CSE | Client-Side Encryption | 고객이 직접 암호화 후 업로드 | "AWS에 평문을 보여주고 싶지 않다" — AWS는 암호화된 데이터만 저장. 가장 안전하지만 운영 부담이 큼 |

> 💡 왜 CSE만 쓰지 않고 SSE를 쓰나?
> CSE는 가장 안전하지만, 클라이언트가 직접 구현해야 하고 키 관리도 전부 자기 책임이라 운영 부담이 크다. SSE는 S3가 다 알아서 해주므로 편리하고, SSE-KMS를 쓰면 감사 기록까지 남아서 규정 준수도 가능. 대부분의 기업은 SSE-S3(기본) 또는 SSE-KMS(감사 필요 시)로 충분.

---

SSE-KMS 주의사항 (시험 빈출):
- KMS에는 초당 API 호출 한도가 있음 (리전별 5,500~30,000건). SSE-KMS를 쓰면 객체를 업로드/다운로드할 때마다 KMS API를 호출하므로, 대량 작업 시 한도에 걸릴 수 있음
- 이 문제를 해결하는 것이 S3 Bucket Keys: 버킷 레벨 키를 캐시해서 KMS API 호출 99% 절감. 원래는 객체마다 KMS에 요청을 보내야 하는데, Bucket Keys를 쓰면 버킷 단위로 한 번만 KMS를 호출하고 이후는 캐시된 키를 재사용
- SSE-KMS 객체를 다운로드할 때도 `kms:Decrypt` 권한 필요 — S3 접근 권한만으로는 부족, KMS 키 권한도 있어야 읽을 수 있음
- 교차 계정 접근 시 KMS Key Policy에서 상대 계정에 권한 부여 필요

SSE-C 주의사항:
- 반드시 HTTPS를 통해서만 사용 가능 (키를 HTTP로 보내면 네트워크에서 탈취 가능하므로)
- AWS는 키를 저장하지 않음 → 키 분실 시 데이터 복구 불가

시험 판별:
- 키 사용 감사 필요? → SSE-KMS
- 고객이 직접 키 관리, AWS에 맡기지 않음? → SSE-C
- 2중 암호화/초엄격 규정? → DSSE-KMS
- 그냥 기본 암호화? → SSE-S3 (기본 적용)
- AWS에 평문 보내기 싫음? → CSE

### 1.8 접근 제어

💡 S3에 누가 접근할 수 있는지를 결정하는 여러 겹의 보안 장치가 있다. 시험에서는 "이런 상황에서 접근이 되는가/안 되는가"를 자주 물어보므로 평가 순서를 이해해야 함.

권한 평가 순서 — 요청이 들어오면 위에서부터 순서대로 체크:
1. Block Public Access (퍼블릭 접근 차단) — 켜져 있으면 퍼블릭 허용 모두 차단 (최우선)
2. 명시 거부 — IAM (Identity and Access Management) 또는 Bucket Policy에 Deny가 하나라도 있으면 무조건 거부
3. 조직 SCP (Service Control Policy) — Organizations에서 계정 전체에 걸린 제한
4. IAM 정책 + Bucket Policy 중 어느 하나라도 Allow이면 통과
5. ACL (Access Control List) — 비권장, 기본적으로 비활성 상태

Bucket Policy (버킷 정책): 버킷에 직접 붙이는 JSON 형식의 접근 규칙. "누가, 무엇을, 허용/거부, 어떤 객체에, 어떤 조건에서" 할 수 있는지 정의
- 다른 AWS 계정의 사용자에게도 접근을 허용할 수 있음 (교차 계정 접근)
- 💡 Condition 활용 예: 특정 IP에서만 접근 허용, 특정 VPC Endpoint에서만 접근 허용, MFA 인증한 사용자만 허용, SSE-KMS 암호화를 강제, aws:PrincipalOrgID로 같은 Organizations 내 계정만 허용

Block Public Access: 계정/버킷 레벨에서 모든 퍼블릭 허용을 한 번에 차단, 기본 ON
- 💡 4가지 독립 설정 (앞 2개는 ACL 경로 차단, 뒤 2개는 Bucket Policy 경로 차단. 4개 모두 켜면 어떤 경로로든 퍼블릭 접근 불가):
  - BlockPublicAcls: 퍼블릭 접근을 허용하는 ACL을 새로 추가하는 것을 차단
  - IgnorePublicAcls: 이미 설정된 퍼블릭 ACL을 무시
  - BlockPublicPolicy: 퍼블릭 접근을 허용하는 버킷 정책 저장 자체를 차단
  - RestrictPublicBuckets: 퍼블릭 정책이 이미 적용된 버킷이라도 퍼블릭/교차 계정 접근을 차단
- 계정 레벨로 설정하면 모든 하위 버킷에 강제 적용

S3 Object Ownership — 💡 다른 AWS 계정이 내 버킷에 객체를 업로드하면, 기본적으로 업로드한 계정이 그 객체의 소유자가 된다. 그러면 버킷 주인이 자기 버킷 안의 객체를 제어하지 못하는 문제가 생김. 이를 해결하기 위한 설정:
- Bucket owner enforced (기본, 권장): ACL 비활성, 버킷 소유자가 모든 객체 소유. 누가 올리든 버킷 주인이 소유
- Bucket owner preferred: 교차 계정 업로드 시 `bucket-owner-full-control` ACL이면 버킷 소유자로 전환
- Object writer: 업로드한 계정이 객체 소유 (레거시)

S3 Access Points: 💡 버킷 정책은 버킷 하나에 하나만 붙일 수 있는데, 부서가 10개이고 각 부서마다 접근 권한이 다르면 하나의 정책이 매우 복잡해짐. Access Point를 쓰면 한 버킷 위에 용도별 엔드포인트를 여러 개 만들어서, 각각에 별도 정책을 부여할 수 있음
- VPC 전용 Access Point: 특정 VPC에서만 접근 허용

### 1.9 정적 웹호스팅 + CloudFront + ACM

💡 HTML, CSS, JS, 이미지 같은 정적 파일로만 구성된 웹사이트는 서버(EC2) 없이 S3만으로 호스팅할 수 있다. 서버 관리가 불필요하고 트래픽에 따라 자동 확장되므로 비용이 매우 저렴.

- S3 자체로 정적 웹 사이트 호스팅 가능 (HTTP만, 리전 종속)
- 엔드포인트: `http://<bucket>.s3-website-<region>.amazonaws.com`
- HTTPS/글로벌 지연 단축/커스텀 도메인 → **CloudFront + ACM + Route 53** 조합
- **OAC (Origin Access Control)** — CloudFront만 S3 버킷에 접근할 수 있도록 제한하는 설정. 이걸 쓰면 S3 버킷을 퍼블릭으로 열지 않고도 CloudFront를 통해서만 콘텐츠를 제공할 수 있음 (OAI라는 이전 방식의 후속 버전)
  - Block Public Access 켜둔 상태에서도 CloudFront는 통과
  - OAC 사용이 권장 (OAI는 레거시, SSE-KMS 미지원)
- **ACM (AWS Certificate Manager)**: SSL/TLS 인증서를 발급·관리하는 서비스. CloudFront에서 HTTPS를 쓰려면 ACM 인증서가 필요하며, 반드시 **us-east-1** 리전에서 발급해야 함 (CloudFront는 글로벌 서비스이므로)
- **Route 53**: AWS의 DNS 서비스. A Alias 레코드로 커스텀 도메인을 CloudFront 배포에 연결

### 1.10 복제 (CRR / SRR)

한 버킷에 쓴 객체를 다른 버킷으로 자동 복제하는 기능. 원본 버킷에 객체를 넣으면 설정된 대상 버킷으로 자동 복사됨.

💡 왜 복제가 필요한가?
- 재해 복구(DR): 서울 리전이 다운되더라도 도쿄 리전의 복제본에서 서비스를 계속할 수 있음
- 지연 최소화: 미국 사용자는 미국 리전 복제본에서, 한국 사용자는 서울 리전에서 접근하면 빠름
- 규정 준수: "데이터를 특정 리전에도 사본을 보관해야 한다"는 법적 요구

- **CRR (Cross-Region Replication, 교차 리전 복제)**: 다른 리전의 버킷으로 복제
- **SRR (Same-Region Replication, 동일 리전 복제)**: 같은 리전의 다른 버킷으로 복제 → 로그 집계, 데이터 거버넌스 분리, 프로덕션/테스트 환경 간 데이터 동기화
- 전제조건: 원본/대상 버킷 모두 Versioning ON, 복제를 수행할 **IAM Role** 필요
- 복제 대상: 새로 쓰거나 수정된 객체만, 과거 객체는 **S3 Batch Replication**으로 일괄 복제
- 삭제: Delete Marker 복제는 선택(기본 OFF), 영구 삭제(버전 ID 지정)는 복제되지 않음 — 실수로 원본을 삭제해도 복제본은 안전하게 남도록 하는 설계
- **RTC (Replication Time Control)**: 15분 내 99.99% 복제 완료를 보장하는 유료 옵션. RPO (Recovery Point Objective)가 엄격한 금융/의료에서 사용
- 교차 계정 복제 가능: 대상 버킷 정책에서 원본 계정의 IAM Role 허용 필요
- 💡 주의 — 복제는 "한 단계"만 됨: 버킷 A→B 복제, 버킷 B→C 복제를 각각 설정해도, A에 넣은 객체가 C까지 자동으로 전달되지 않음. 복제로 들어온 객체는 다시 복제 대상이 되지 않기 때문. A의 데이터를 C에도 두고 싶으면 A→C 복제를 별도로 설정해야 함

### 1.11 Object Lock / Glacier Vault Lock

💡 금융, 의료, 공공기관 등에서는 법적으로 "이 데이터는 N년 동안 절대 삭제하면 안 된다"는 규정이 있는 경우가 많다. 이를 위해 S3가 제공하는 잠금 기능이 Object Lock이다.

Object Lock — 한 번 쓰면 수정/삭제를 막는 WORM (Write Once Read Many) 잠금. S3 버킷의 개별 객체 단위로 잠금:
- Versioning 켠 버킷에서만 사용, 기존 버킷에도 나중에 활성화 가능 (한 번 활성화하면 해제 불가)
- 보관 모드 두 가지:
  - Governance: 특권 IAM이 해제 가능
  - Compliance: root 계정조차도 수정/삭제 불가, 보관 기간 단축도 불가 → 가장 엄격. 💡 단, 설정한 보관 기간(예: 7년)이 만료되면 정상적으로 삭제 가능. "영원히 못 지운다"가 아니라 "보관 기간 동안은 누구도 못 지운다"는 의미
- Legal Hold (법적 보류): 기간 설정 없이 "이 객체는 홀드" 상태 유지, 수사/소송 대비

> 참고 자료 정정 — Object Lock 활성화 시점
> 기존 문서에서 "버킷 생성 시 또는 이후에 걸 수 있음"이라고 서술했는데, 이것은 맞음. 과거에는 생성 시에만 가능했으나 AWS가 업데이트하여 기존 버킷에도 활성화 가능해짐. 단, 한 번 활성화하면 해제 불가.
> 참고: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock-configure.html

Glacier Vault Lock — 💡 Glacier는 원래 S3와 별개의 서비스였고, 그때부터 있던 자체 잠금 기능. Object Lock이 개별 객체를 잠그는 것과 달리, Vault Lock은 Vault(아카이브 저장소) 전체에 대한 정책을 한 번 설정하면 변경 불가로 만드는 것. 정책 자체를 잠그기 때문에, "이 Vault는 영구 보관"이라고 설정하면 진짜로 영원히 삭제 불가 — 설정 시 매우 신중해야 함

시험 판별:
- "S3 객체를 N년간 삭제 불가" → Object Lock Compliance
- "관리자만 예외적 삭제 가능" → Object Lock Governance
- "법적 보류" → Legal Hold
- "Glacier Vault 정책을 변경 불가능하게" → Glacier Vault Lock

### 1.12 Pre-signed URL (미리 서명된 URL)

- AWS 계정이 없는 외부 사용자에게 **특정 객체 1개에 대해 시간 제한이 있는 URL을 발급**하여 임시 접근을 허용하는 기능. 예를 들어 유료 콘텐츠 다운로드 링크, 고객이 파일을 업로드할 수 있는 임시 URL 등에 활용
- 서명 유효 기간: 최대 **7일** (IAM User), IAM Role/STS는 임시 자격 증명 만료까지
- 서명에 쓰인 IAM의 권한을 초과할 수 없음
- GET = 다운로드 링크, PUT = 업로드 링크

> **참고 자료 정정 — Pre-signed URL 유효 기간**
> 기존 문서에서 "최대 7일(IAM User) / 36시간(IAM Role)"이라고 서술했으나, IAM Role의 경우 고정 36시간이 아니라 **임시 자격 증명의 유효 기간까지**가 정확함. URL 만료 = min(설정한 만료 시간, 자격 증명 만료 시간).
> 참고: https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html

### 1.13 이벤트 / Transfer Acceleration / Multipart Upload

이벤트 알림 (Event Notifications):
- 💡 S3 버킷에 객체가 업로드/삭제/복원될 때 자동으로 다른 서비스에 알림을 보내는 기능. 예를 들어 "이미지가 업로드되면 자동으로 Lambda가 썸네일을 생성" 같은 자동화에 활용
- 이벤트 종류: ObjectCreated, ObjectRemoved, ObjectRestore 등
- 직접 대상: Lambda, SQS, SNS
- EventBridge 통합 (권장): S3 → EventBridge → 18+ AWS 서비스로 라우팅, 고급 필터링, 멀티 대상 지원

전송 가속화 (Transfer Acceleration):
- CloudFront 엣지 네트워크를 통해 업로드를 가속 — 사용자와 가까운 엣지로 보내고, 엣지에서 S3까지는 AWS 내부 고속망을 사용. 원거리 대용량에 특히 효과
- 버킷 단위로 켜고, 엔드포인트는 `<bucket>.s3-accelerate.amazonaws.com`
- Multipart Upload와 조합 가능
- 💡 무조건 켜면 좋은 건 아님:
  - 추가 비용 발생: GB당 약 0.04~0.08 달러가 일반 전송 요금 위에 추가됨
  - 가까운 거리에서는 효과 없음: 같은 리전 근처에서 업로드하면 속도 차이가 없는데 비용만 추가됨. AWS가 가속 효과가 없다고 판단하면 가속 요금은 안 매기지만, 기본 전송 요금은 나옴
  - 버킷 이름에 점(.)이 있으면 사용 불가 (예: `my.bucket.name`)
  - 적합한 경우: 업로드하는 사용자가 S3 버킷과 물리적으로 먼 거리에 있고, 대용량 파일을 자주 올릴 때

멀티파트 업로드 (Multipart Upload):
- 100MB 이상 권장, 5GB 초과는 필수
- 파트 단위 병렬 업로드 → 속도 향상, 실패한 파트만 재전송, 총 5TB까지
- AbortIncompleteMultipartUpload Lifecycle: 미완료 업로드는 파트가 S3에 남아서 스토리지 비용이 계속 발생 → 자동 정리 규칙 필수

바이트 범위 요청 (Byte-Range Fetches):
- GET 요청에 Range 헤더를 지정해 객체의 일부분만 다운로드
- 💡 언제 쓰나:
  - 대용량 파일 다운로드 가속: 예를 들어 10GB 파일을 처음부터 끝까지 한 번에 받는 대신, 0~1GB / 1~2GB / ... 식으로 10개 범위를 동시에 요청해서 병렬 다운로드
  - 파일의 일부만 필요할 때: 예를 들어 동영상 파일의 처음 몇 MB만 읽어서 메타데이터(해상도, 길이 등)를 추출하거나, 로그 파일의 마지막 부분만 가져올 때. 전체를 다운로드하지 않으므로 전송 비용 절감

### 1.14 S3 VPC 접근 (시험 매우 빈출)

💡 S3는 VPC 바깥에 있는 서비스이다. EC2는 VPC 안(AWS 사설 네트워크)에 있고, S3는 공인 인터넷에 노출된 서비스(공인 URL로 접근)이므로, 원래 EC2가 S3에 접근하려면 공인 인터넷 망을 거쳐야 한다. VPC Endpoint를 쓰면 공인 인터넷을 거치지 않고 AWS 내부 전용 네트워크로 S3에 접근할 수 있다.

게이트웨이 엔드포인트 (Gateway Endpoint):
- 💡 Gateway Endpoint 없이: EC2 → VPC → 인터넷 게이트웨이 → 공인 인터넷 → S3 (EC2에 퍼블릭 IP 또는 NAT Gateway 필요)
- 💡 Gateway Endpoint 사용 시: EC2 → VPC 내부 → Gateway Endpoint → AWS 내부 네트워크 → S3 (퍼블릭 IP도 NAT도 불필요)
- 무료
- 라우트 테이블에 항목을 추가하는 방식
- Endpoint Policy로 특정 버킷/액션만 허용 가능
- 버킷 정책에서 `aws:sourceVpce` 조건으로 특정 VPC Endpoint에서만 접근 허용

인터페이스 엔드포인트 (Interface Endpoint, PrivateLink):
- VPC 안에 프라이빗 IP를 가진 ENI (Elastic Network Interface)를 생성하여 S3에 접근
- Gateway Endpoint와 달리, 온프레미스에서 Direct Connect/VPN을 통해 이 프라이빗 IP로 접근 가능 → 온프레미스 ↔ S3 프라이빗 연결이 필요할 때 사용
- 비용 발생 (시간당 + 데이터 처리량)

시험 판별:
- "프라이빗 서브넷의 EC2가 S3에 접근" → Gateway Endpoint (무료, 대부분 정답)
- "온프레미스에서 S3에 프라이빗 접근" → Interface Endpoint
- "공인 인터넷을 거치지 않고 S3 접근" → VPC Endpoint

### 1.15 기타 S3 기능

**S3 Select**: S3에 저장된 CSV/JSON/Parquet 파일에 간단한 SQL을 실행하여 **필요한 행/열만 서버 측에서 걸러서** 가져오는 기능. 예를 들어 10GB 로그 파일에서 특정 조건에 맞는 행만 뽑으면, 10GB를 전부 다운로드하지 않고 결과만 받을 수 있어서 비용과 시간 절감

**S3 Batch Operations (일괄 작업)**: 수십억 개 객체에 대한 대량 작업을 한 번에 실행 (복사, 태깅, ACL 변경, Glacier 복원 등)
- **S3 Batch Replication (일괄 복제)**: 기존 객체를 다른 버킷으로 일괄 복제

**S3 Object Lambda**: S3에서 객체를 다운로드(GET)할 때 Lambda 함수를 자동으로 거쳐서 실시간으로 변환된 결과를 반환하는 기능. 원본 객체는 변경하지 않음. 예: 같은 고객 데이터 파일인데, A 앱에서 받을 때는 주민번호를 마스킹해서 주고, B 앱에서 받을 때는 원본 그대로 주는 식으로 활용

**요청자 지불 (Requester Pays)**: 보통 S3에서 데이터를 다운로드하면 버킷 소유자가 전송 비용을 부담하는데, 이 설정을 켜면 다운로드하는 사람(요청자)이 전송 비용을 부담. 대용량 공개 데이터셋을 제공할 때 유용. 익명 요청은 불가 (요청자가 누구인지 알아야 비용을 청구할 수 있으므로)

**CORS (Cross-Origin Resource Sharing)**: 웹 브라우저에서 다른 도메인의 리소스를 호출할 수 있도록 허용하는 설정. 예를 들어 `site-a.com`에서 호스팅하는 웹페이지가 `site-b.com`의 S3 버킷에서 이미지를 가져오려면, `site-b.com` 버킷에 CORS를 설정해서 `site-a.com`의 요청을 허용해줘야 함

서버 접근 로깅 (Server Access Logging): 버킷에 대한 모든 요청(누가, 언제, 어떤 객체에, 어떤 작업을 했는지)을 로그로 기록 → 다른 S3 버킷에 저장. 💡 로그 버킷과 원본 버킷은 반드시 다른 버킷이어야 함 — 같은 버킷에 저장하면 로그 저장 자체가 새로운 요청을 만들고, 그 요청이 또 로그를 만들고... 하는 무한 루프 발생

**Storage Class Analysis (스토리지 클래스 분석)**: S3가 버킷 내 객체들의 접근 패턴을 자동으로 분석해서 "이 객체들은 IA로 옮기면 비용이 절감됩니다" 같은 권장 리포트를 생성. Lifecycle 규칙을 만들기 전에 참고하면 유용

**S3 Inventory (인벤토리)**: 버킷에 있는 객체들의 목록과 메타데이터(크기, 스토리지 클래스, 암호화 상태 등)를 매일/매주 CSV 또는 Parquet 파일로 자동 생성. "우리 버킷에 뭐가 얼마나 있나" 파악하거나, Batch Operations의 입력으로 활용

---

## 2. Amazon EFS

### 2.1 EFS란?

💡 여러 EC2 인스턴스가 **같은 파일을 동시에 읽고 써야** 하는 상황(예: 웹 서버 여러 대가 같은 설정 파일이나 콘텐츠를 공유)에서 EBS는 한 EC2에만 붙일 수 있어서 불가능하다. 이때 쓰는 것이 EFS.

- **NFS (Network File System) v4.0/v4.1** 프로토콜을 사용하는 네트워크 파일 스토리지 — 여러 컴퓨터가 네트워크를 통해 하나의 파일 시스템을 공유할 수 있게 해주는 프로토콜
- 💡 **Linux 전용** (Windows 미지원 — Windows는 FSx for Windows File Server)
- **수천 개의 EC2에서 동시에 접근** 가능 (공유 파일 서버). EBS는 기본적으로 하나의 EC2에만 연결되지만, EFS는 여러 EC2가 같은 파일을 동시에 읽고 쓸 수 있음
- 파일 추가/삭제에 따라 자동 확장 → 미리 크기를 프로비저닝할 필요 없음, 페타바이트까지 확장 가능
- 보안 그룹으로 접근 제어
- POSIX 호환 파일 시스템 — Linux의 파일 권한 모델(소유자/그룹/기타, rwx)을 그대로 사용

### 2.2 가용성 및 네트워크

- **Mount Target (마운트 타겟)**: EFS에 접근하기 위한 연결 지점. 각 AZ에 하나씩 만들어야 하며, EC2는 자기가 속한 AZ의 Mount Target을 통해 EFS에 연결. Mount Target이 없는 AZ의 EC2는 EFS에 접근할 수 없음
- 여러 AZ에 중복 저장 → 하나의 AZ가 파괴되더라도 다른 AZ에서 서비스 가능
- **VPN 또는 Direct Connect**를 통해 **온프레미스에서 접속 가능**
- 💡 Lambda, ECS/Fargate에서도 마운트 가능

### 2.3 스토리지 클래스와 수명 주기 관리

- 4가지 스토리지 클래스:
  - **Standard**: 여러 AZ에 저장, 높은 가용성
  - **Standard-IA**: 여러 AZ, 자주 접근하지 않는 데이터용 (저장 비용 저렴, 접근 시 비용 발생)
  - **One Zone**: 한 개 AZ에만 저장 → 비용 절감, AZ 장애 시 데이터 손실 가능
  - 💡 **One Zone-IA**: One Zone + Infrequent Access 결합 (가장 저렴)
- **수명 주기 관리**: 지정 기간 동안 미접근 파일을 **IA 클래스로 자동 이동**, 다시 접근하면 Standard로 되돌리기도 가능

### 2.4 처리량 모드 / 성능 모드

**처리량 모드** — 파일 시스템이 사용할 수 있는 전체 처리량:
- **Bursting** (기본): 스토리지 용량에 따라 처리량이 확장. 용량이 적으면 기본 처리량도 낮고, 버스트 크레딧으로 단기간만 올릴 수 있음
- **Elastic** (권장): 워크로드에 따라 처리량을 자동으로 조절. 용량과 무관하게 필요한 만큼 처리량이 늘어나므로 가장 편리
- **Provisioned**: 처리량을 미리 지정. 일정한 고처리량이 항상 필요할 때

**성능 모드** — 개별 파일 접근의 응답 속도:
- **General Purpose** (기본): 저지연, 대부분의 워크로드에 적합
- **Max I/O**: 수천 EC2 동시 접근 시 최적화, 처리량 극대화하지만 개별 접근 지연시간이 다소 높음

### 2.5 암호화

- 💡 **전송 중 암호화**: TLS (Transport Layer Security)를 사용하여 마운트 (`mount -t efs -o tls`)
- 💡 **저장 시 암호화**: KMS 키를 사용, 파일 시스템 생성 시 설정
- 💡 기존 미암호화 EFS를 암호화로 변경 불가 → 새 EFS 생성 후 데이터 마이그레이션 (DataSync 활용)

### 2.6 EFS Access Points

- 💡 애플리케이션별로 **다른 루트 디렉토리와 사용자/그룹**을 강제 적용
- IAM 정책으로 Access Point별 접근 제어 가능
- 사용 예: 멀티 테넌트 환경에서 각 앱이 자기 디렉토리만 접근

### 2.7 EFS Replication

- 💡 EFS를 다른 리전 또는 같은 리전의 다른 EFS로 자동 복제 → DR 시나리오

### 💡 2.8 EFS vs EBS 비교

| 항목 | EFS | EBS |
|------|-----|-----|
| 프로토콜 | NFS v4 | 블록 스토리지 |
| 공유 | **다수 EC2 동시 접근** | 단일 EC2 (io1/io2 Multi-Attach 제외) |
| AZ | **다중 AZ** | 단일 AZ |
| 크기 | 자동 확장 | 미리 프로비저닝 |
| OS | **Linux 전용** | Linux/Windows |
| 사용 사례 | 공유 파일, CMS, 웹 서빙 | DB, 부트 볼륨 |

---

## 3. Amazon FSx

### 3.1 FSx란?

💡 EFS는 Linux의 NFS만 지원한다. 하지만 현실에서는 **Windows SMB 파일 공유**가 필요하거나, **초고성능 HPC용 파일 시스템**(Lustre)이 필요하거나, **온프레미스에서 쓰던 NetApp/ZFS를 그대로 AWS로 옮기고 싶은** 경우가 있다. 이런 다양한 파일 시스템 수요를 위해 AWS가 제공하는 것이 FSx.

- 서드파티 파일 시스템을 AWS에서 사용할 수 있도록 지원하는 완전 관리형 서비스
- 하드웨어 프로비저닝, 소프트웨어 구성, 패치, 백업 등을 자동으로 관리
- 유휴/전송 데이터를 자동 암호화, VPC 내에서 실행
- **4가지 파일 시스템 지원**

### 3.2 FSx for Windows File Server

- **SMB (Server Message Block)** 프로토콜을 지원하는 파일 서버 — 💡 Windows에서 파일 공유할 때 쓰는 표준 프로토콜 (탐색기에서 `\\서버\공유폴더`로 접근하는 것)
- **Active Directory (AD)**와 통합한 Windows 기반 환경 지원 — 💡 회사 Windows 도메인에 참여시켜서 기존 사용자/그룹 권한 그대로 사용 가능
- **DFS (분산 파일 시스템)** 네임스페이스로 여러 파일 공유를 하나의 구조로 그룹화 — 💡 여러 파일 서버를 하나의 경로처럼 보이게 해주는 기능
- 💡 다중 AZ 배포 지원 (고가용성)
- 💡 온프레미스에서 접근 가능 (VPN/Direct Connect 경유)
- 시험 키워드: **SMB, Windows, Active Directory** → FSx for Windows

### 3.3 FSx for Lustre

- **Linux에서 실행되는 고성능 컴퓨팅(HPC, High Performance Computing)** 워크로드에 적합 — 💡 과학 시뮬레이션, 금융 모델링, 머신러닝 학습 등 대량의 데이터를 빠르게 읽고 써야 하는 작업
- POSIX 호환, Linux 인스턴스 및 컨테이너에서 접근
- **S3와 원활한 통합**:
  - S3 버킷을 데이터 리포지토리로 연결
  - **Lazy Loading (지연 로딩)**: 처음 접근할 때만 S3에서 가져옴 (전체 복사 X)
  - 처리 결과를 S3에 **Write-back (쓰기 반영)** 가능
- 💡 **배포 유형**:
  - **Scratch**: 임시 스토리지, 데이터 복제 없음, 단기 고성능 처리
  - **Persistent**: 장기 스토리지, 동일 AZ 내 데이터 복제
- 시험 키워드: **HPC, Linux, 고성능, 머신러닝, S3 연동** → FSx for Lustre

### 3.4 FSx for NetApp ONTAP / OpenZFS

- 💡 **NetApp ONTAP**: NFS/SMB/iSCSI (Internet Small Computer Systems Interface) **모두 지원** (멀티프로토콜), 온프레미스 NetApp 마이그레이션에 적합, 모든 OS에서 접근 가능
- 💡 **OpenZFS**: NFS 프로토콜, Linux 기반 고성능, 스냅샷/클론 지원, ZFS 마이그레이션에 적합

### 3.5 FSx 선택 가이드

| 요구사항 | 선택 |
|--------|------|
| Windows + SMB + AD | FSx for Windows File Server |
| Linux + HPC + S3 연동 | FSx for Lustre |
| 멀티프로토콜 (NFS+SMB+iSCSI) | FSx for NetApp ONTAP |
| Linux + 고성능 + ZFS 마이그레이션 | FSx for OpenZFS |

---

## 4. Snow Family

### 4.1 Snow Family란?

- 대용량 데이터를 AWS로 마이그레이션하거나 엣지 컴퓨팅 처리를 위해 사용되는 **물리적 장비** 서비스
- 흐름: AWS에서 장비 배송 → 데이터 로드 → AWS로 반송 → S3에 업로드
- 💡 **OpsHub**: Snow 장비를 GUI로 관리할 수 있는 데스크톱 앱

### 4.2 Snow Family가 필요한 이유

- 네트워크로 전송하기엔 데이터 양이 너무 많을 때. 💡 예: 100TB를 1Gbps 인터넷으로 전송하면 약 **12일**이 걸리지만, Snowball Edge로 보내면 장비 배송 포함 약 **1주일**이면 완료
- 네트워크 연결이 불가능하거나 대역폭이 부족한 환경
- **엣지 컴퓨팅**: 데이터가 발생하는 현장(공장, 광산, 선박 등)에서 AWS 클라우드에 보내지 않고 **장비 내에서 직접 데이터를 처리**하는 것. 인터넷이 느리거나 없는 원격지에서 실시간 처리가 필요할 때 사용

### 4.3 Snowcone

- Snow Family 중 **가장 작은** 장비
- 💡 **HDD**: 8TB / **SSD**: 14TB
- **DataSync 에이전트 사전 설치**: DataSync는 AWS에서 제공하는 데이터 이동 서비스로, 네트워크를 통해 온프레미스 ↔ AWS 간 데이터를 전송하는 도구. Snowcone에는 이 에이전트가 미리 설치되어 있어서, 네트워크 연결이 가능하면 장비를 반송하지 않고 **온라인으로 전송도 가능** (Snow Family 중 유일)

### 4.4 Snowball Edge

- Snow Family의 대표적인 데이터 전송 장비
- **EC2와 Lambda** 실행 가능 (엣지 컴퓨팅)
- 두 가지 유형:
  - **Storage Optimized**: 💡 **80TB** — 대용량 데이터 전송에 최적
  - **Compute Optimized**: 💡 **42TB** + GPU 옵션 — 고성능 워크로드에 적합

### 4.5 Snowmobile

- **엑사바이트급** 대규모 데이터를 전송하는 서비스 (트레일러 컨테이너)
- 💡 대당 **100PB** 저장 가능
- 💡 **10PB 이상**의 데이터 전송 시 Snowball Edge보다 효율적

### 4.6 Snow Family 선택 가이드

| 데이터 규모 | 선택 |
|---------|------|
| ~14TB | **Snowcone** |
| 수십~수백 TB | **Snowball Edge** |
| 10PB 이상 | **Snowmobile** |

> **참고 자료 정정 — Snow Family 오타**
> 동료 정리 문서에서 "Glaicer"로 표기된 부분이 있으나 올바른 철자는 **"Glacier"**.

---

## 5. 핵심 요약 & 시험 포인트

### S3 핵심

- 버킷 = 리전 귀속, 이름 전 세계 유일. 객체 최대 5TB. 단일 PUT 5GB, 그 이상 Multipart 필수
- 클래스 7종: 자주 접근(Standard) → 가끔(IA) → 거의 안 봄(Glacier) → 보관만(Deep Archive)
- **Intelligent-Tiering**: 5개 자동 단계, 접근 패턴 예측 불가 시 최적, 검색 비용 없음
- **Glacier Flexible 복원**: Expedited(1-5분, 비쌈) / Standard(3-5시간) / Bulk(5-12시간, 저렴)
- Lifecycle: "30일 → IA, 90일 → Glacier, 365일 → Deep Archive"
- Versioning + MFA Delete = 영구 삭제 방지. Delete Marker 제거 = 즉시 복원
- 암호화: SSE-S3 기본, 감사 필요 SSE-KMS, 고객 키 SSE-C, 2중 DSSE-KMS, CSE
- **SSE-KMS throttling** → S3 Bucket Keys로 해결
- 접근 제어: Block Public Access → 명시 거부 → IAM+Bucket Policy → ACL(비권장)
- Bucket Policy Condition: **aws:PrincipalOrgID**
- 정적 웹: CloudFront + **OAC** + ACM(us-east-1) + Route 53 Alias
- 복제: CRR(교차 리전/DR), SRR(동일 리전), RTC(15분 SLA). Versioning 필수
- Object Lock Compliance = root도 못 지움. Governance = 특권만 해제. Legal Hold = 기간 없이 보류
- Pre-signed URL = IAM 없이 시간제 접근
- Transfer Acceleration = 원거리 대용량 업로드, 엣지 경유
- VPC → S3 프라이빗 접근 = **Gateway Endpoint (무료)**, 온프레미스 = **Interface Endpoint**
- S3 Event → Lambda/SQS/SNS/EventBridge

### EFS 핵심

- NFS v4 기반, **Linux 전용**, **다수 EC2 동시 접근**, 자동 확장, **다중 AZ**
- 각 AZ에 **Mount Target** 필요
- 성능 모드: General Purpose (기본, 저지연) vs Max I/O (고처리량)
- 처리량 모드: Bursting (기본) / **Elastic (권장)** / Provisioned
- **Windows → FSx for Windows**, **Linux 공유 → EFS** (시험 판별)

### FSx 핵심

- Windows + SMB + AD → **FSx for Windows File Server**
- Linux + HPC + S3 연동 → **FSx for Lustre** (Scratch: 임시, Persistent: 장기)
- 멀티프로토콜 → FSx for NetApp ONTAP
- Linux + ZFS → FSx for OpenZFS

### Snow Family 핵심

- 대용량 오프라인 마이그레이션: Snowcone(8-14TB) < Snowball Edge(42-80TB) < Snowmobile(100PB)
- Snowball Edge: **Storage Optimized (80TB)** vs **Compute Optimized (42TB + GPU)**
- Snowcone: **DataSync 온라인 전송 가능** (유일)
- Snowmobile: **10PB 이상** 시 효율적
- 엣지 컴퓨팅: EC2/Lambda를 Snow 장비에서 로컬 실행 가능
