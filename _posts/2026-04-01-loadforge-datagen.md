---
title: "dev 환경에서 프로덕션급 테스트를 위한 fake 데이터 생성기 — loadforge"
date: 2026-04-01
categories: [projects]
tags: [python, data-engineering, cli, testing, open-source]
---

## 왜 만들었나

데이터 엔지니어로 일하면서 늘 부딪히는 문제가 있습니다. **dev 환경에서 프로덕션 수준의 테스트를 하고 싶은데, 적절한 테스트 데이터가 없다.**

프로덕션 데이터를 마스킹해서 가져오는 건 보안/규정 문제가 있고, 손으로 만드는 건 몇 행이 한계입니다. 그렇다고 랜덤 값만 채워 넣으면 현실과 동떨어진 데이터라 제대로 된 테스트가 안 됩니다.

결국 필요한 건 이런 겁니다:

- **현실적인 fake 데이터** — 실제 서비스 데이터와 비슷한 분포와 패턴
- **대량 생성** — 수백만 행도 빠르게
- **테이블 간 관계 유지** — 부모-자식 테이블의 참조가 깨지지 않는 데이터
- **엣지 케이스 포함** — 프로덕션에서 터지는 그런 값들

이런 도구가 있으면 좋겠다고 몇 년간 생각만 하다가, 결국 직접 만들었습니다.

## loadforge란

[loadforge](https://pypi.org/project/loadforge/)는 DB 스키마를 읽어서 **현실적인 fake 데이터를 대량 생성**하는 Python CLI 도구입니다.

dev 환경, 스테이징, CI 파이프라인에서 프로덕션 수준의 테스트를 돌리기 위한 데이터를 만들어줍니다. 서비스 기능 테스트든, 데이터 파이프라인 검증이든, 부하 테스트든 용도에 맞게 쓸 수 있습니다.

```bash
pipx install loadforge
datagen --ddl schema.sql --rows 10000 --out json --output-path test_data.json
```

스키마만 넣으면 테이블 간 관계를 분석해서, 알아서 순서대로 데이터를 채워줍니다.

## 어떤 상황에서 쓰나

### 서비스 기능 테스트

새로운 API를 개발했는데, 빈 DB로는 제대로 테스트가 안 됩니다. 유저 1만 명, 주문 5만 건, 이벤트 20만 건 — 이런 규모의 데이터가 들어있는 dev DB가 필요합니다.

```bash
datagen --ddl schema.sql \
  --table-rows users=10000,orders=50000,events=200000 \
  --out postgres --output-path seed.sql
```

테이블별로 원하는 행 수를 지정할 수 있습니다.

### 데이터 파이프라인 검증

ETL이나 ELT 파이프라인이 제대로 동작하는지 확인하려면, 다양한 패턴의 입력 데이터가 필요합니다. loadforge는 20행마다 엣지 케이스를 자동으로 섞어줍니다:

- 특수문자, 아주 긴 문자열, 경계값
- 0, 음수, 매우 큰 숫자
- 파이프라인에서 흔히 문제를 일으키는 패턴들

프로덕션에서 터지기 전에 dev에서 미리 잡을 수 있습니다.

### 부하 테스트

서비스가 대량 데이터를 잘 처리하는지 확인하고 싶을 때, Polars 엔진으로 빠르게 대량 생성할 수 있습니다:

```bash
datagen --ddl schema.sql --rows 1000000 \
  --engine polars --out csv --output-path ./load_test_data
```

### DB에 직접 넣기

생성한 데이터를 파일로 뽑지 않고 바로 dev DB에 넣을 수도 있습니다:

```bash
datagen --ddl schema.sql --rows 10000 \
  --insert --db-url postgresql+psycopg://user:pass@localhost:5432/devdb
```

## 현실적인 데이터 분포

테스트 데이터의 핵심은 **현실성**입니다. 모든 값이 균등 분포면 프로덕션과 전혀 다른 데이터입니다.

`--dist` 옵션으로 컬럼별 분포를 지정할 수 있습니다:

```bash
datagen --ddl schema.sql --rows 10000 \
  --dist users.age:normal,mean=33,std=7 \
  --dist orders.amount:pareto,alpha=1.5,xm=1 \
  --dist tier:weighted,premium=10%,standard=70%,free=20% \
  --dist created_at:peak,hours=9-11,18-20 \
  --out json
```

지원하는 분포들:

| 분포 | 용도 | 예시 |
|------|------|------|
| **normal** | 연속형 수치 | 나이, 점수, 키 |
| **poisson** | 카운트 데이터 | 일일 주문 수, 방문 횟수 |
| **pareto** | 헤비테일 | 결제 금액, 파일 크기 |
| **zipf** | 랭킹/빈도 | 인기 상품, 검색어 순위 |
| **exponential** | 간격/대기시간 | 요청 간격, 세션 길이 |
| **weighted** | 카테고리 비율 | 등급, 상태값 |
| **peak** | 시간대 몰림 | 출퇴근 시간 트래픽 |

예를 들어 `peak`을 쓰면 출퇴근 시간(9-11시, 18-20시)에 데이터가 몰리는 타임스탬프를 생성할 수 있습니다. 실제 서비스 트래픽 패턴과 비슷한 데이터로 테스트할 수 있습니다.

## 스키마 제약조건 인식

loadforge는 스키마에 정의된 제약조건을 읽고, 그에 맞는 데이터를 생성합니다.

### 테이블 간 참조 관계

```sql
CREATE TABLE users (id INT PRIMARY KEY, name VARCHAR(50));
CREATE TABLE orders (
  id INT PRIMARY KEY,
  user_id INT NOT NULL,
  FOREIGN KEY (user_id) REFERENCES users(id)
);
```

`users`를 먼저 생성하고, `orders.user_id`는 실제 생성된 `users.id` 값에서 가져옵니다. 테이블이 10개, 20개여도 의존 관계를 분석해서 올바른 순서로 생성합니다.

### CHECK 조건

```sql
CREATE TABLE products (
  id INT PRIMARY KEY,
  price NUMERIC CHECK (price BETWEEN 10 AND 10000),
  status VARCHAR(10) CHECK (status IN ('active', 'inactive', 'pending'))
);
```

`price`는 10~10000 사이, `status`는 세 값 중 하나로만 생성됩니다.

### UNIQUE / NOT NULL

UNIQUE 컬럼은 중복 없이, NOT NULL 컬럼은 빈 값 없이 생성됩니다.

## 라이브 DB에서 스키마 읽기

DDL 파일이 없어도 됩니다. 이미 돌아가고 있는 DB에서 스키마를 직접 읽어올 수 있습니다:

```bash
datagen \
  --schema-from-db \
  --db-url postgresql+psycopg://user:pass@localhost:5432/mydb \
  --tables users,orders,events \
  --rows 10000 \
  --out json
```

운영 DB의 스키마를 그대로 읽어서, 동일한 구조의 fake 데이터를 dev DB에 채울 수 있습니다.

## 다양한 출력 포맷

생성된 데이터를 어디서 쓸지에 따라 포맷을 선택할 수 있습니다:

```bash
datagen --ddl schema.sql --rows 10000 --out postgres   # PostgreSQL INSERT
datagen --ddl schema.sql --rows 10000 --out mysql       # MySQL INSERT
datagen --ddl schema.sql --rows 10000 --out sqlite      # SQLite INSERT
datagen --ddl schema.sql --rows 10000 --out bigquery    # BigQuery INSERT
datagen --ddl schema.sql --rows 10000 --out json        # JSON
datagen --ddl schema.sql --rows 10000 --out csv         # CSV (테이블별 파일)
datagen --ddl schema.sql --rows 10000 --out parquet     # Parquet
```

## 검증 리포트

생성한 데이터가 실제로 제약조건을 만족하는지 검증하고 리포트를 뽑을 수 있습니다:

```bash
datagen --ddl schema.sql --rows 10000 \
  --strict-checks \
  --report-path report.json \
  --out json --output-path data.json
```

리포트에는 참조 무결성 위반, NOT NULL 위반, UNIQUE 충돌, CHECK 위반이 포함됩니다. CI 파이프라인에 넣어서 자동화하기 좋습니다.

## 설정 파일

매번 긴 CLI 명령을 치는 대신, TOML 파일로 설정을 관리할 수 있습니다:

```toml
ddl = "schema.sql"
rows = 10000
out = "postgres"
seed = 42
strict_checks = true
table_rows = { users = 1000, orders = 5000, events = 20000 }
dist = [
  "users.age:normal,mean=33,std=7",
  "orders.amount:pareto,alpha=1.7,xm=1",
]
```

```bash
datagen --config datagen.toml
```

`--seed` 옵션으로 동일한 데이터를 반복 생성할 수 있어서, 재현 가능한 테스트에 유용합니다.

## AI 도구와 함께 쓰기

CLI 옵션이 다양한 건 의도적입니다. **AI 코딩 도구와 함께 쓸 것**을 염두에 두고 설계했습니다.

예를 들어 Claude Code에 이렇게 요청할 수 있습니다:

> "이 schema.sql로 유저 1만 명, 주문 5만 건 생성해줘. 나이는 정규분포, 결제 금액은 파레토 분포로. dev DB에 바로 넣어줘."

AI가 적절한 옵션 조합을 만들어주기 때문에, 옵션을 외울 필요가 없습니다. CLI의 세밀한 제어력과 자연어 인터페이스가 결합되는 셈입니다.

## 앞으로의 계획

- **DynamoDB 지원** — NoSQL 환경의 fake 데이터 생성
- **분포 확장** — power law 등 추가 분포 지원
- **파이프라인 엣지 케이스 특화** — 핫스팟(데이터 몰림), 타임스탬프 경계값, 대량 중복 등 파이프라인을 깨뜨리는 패턴 생성

## 마치며

몇 년간 생각만 했던 도구를 결국 만들었습니다. 늦었지만, 만들고 나니 왜 진작 안 했나 싶습니다.

dev 환경에서 프로덕션급 테스트를 하려면 프로덕션급 데이터가 필요합니다. loadforge가 그 간극을 채워줄 수 있으면 좋겠습니다.

```bash
pipx install loadforge
```

- GitHub: [hyangminj/datagen](https://github.com/hyangminj/datagen)
- PyPI: [loadforge](https://pypi.org/project/loadforge/)

피드백이나 이슈는 GitHub Issues로 남겨주시면 감사하겠습니다.
