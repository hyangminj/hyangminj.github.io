---
title: "데이터 엔지니어가 직접 만든 테스트 데이터 생성기 — loadforge"
date: 2026-04-01
categories: [projects]
tags: [python, data-engineering, cli, testing, open-source]
---

## 왜 만들었나

데이터 엔지니어로 일하면서 가장 반복적으로 마주치는 문제 중 하나가 **테스트 데이터**입니다.

파이프라인을 새로 만들거나, 스키마를 변경하거나, 스테이징 환경을 세팅할 때마다 매번 손으로 INSERT문을 짜거나, 프로덕션 데이터를 마스킹해서 가져오거나, 적당히 CSV를 만들어왔습니다. 이게 한두 번이면 괜찮은데, 테이블 10개에 FK 관계가 엮여 있고, CHECK 제약조건까지 있으면 수작업은 고통 그 자체입니다.

특히 이런 상황들이 문제였습니다:

- FK 순서를 지키면서 여러 테이블에 데이터를 넣어야 할 때
- 특정 컬럼에 현실적인 분포(정규분포, 포아송 등)를 적용해야 할 때
- 엣지 케이스를 의도적으로 섞어야 할 때 (빈 문자열, 특수문자, 경계값)
- BigQuery, PostgreSQL, MySQL 등 타겟에 맞는 SQL을 뽑아야 할 때

이런 도구가 있으면 좋겠다고 몇 년간 생각만 하다가, 결국 직접 만들었습니다.

## loadforge란

[loadforge](https://pypi.org/project/loadforge/)는 SQL DDL이나 라이브 DB 스키마를 읽어서 **관계를 유지하는 합성 데이터**를 생성하는 Python CLI 도구입니다.

핵심 동작은 단순합니다:

1. DDL 파싱 또는 DB 스키마 인트로스펙션
2. FK 의존 관계를 분석해서 생성 순서 결정 (토폴로지 정렬)
3. 타입별 기본값 + 분포 오버라이드로 데이터 생성
4. 생성된 데이터 검증
5. SQL, JSON, CSV, Parquet 등 원하는 포맷으로 출력

```bash
pipx install loadforge
datagen --ddl schema.sql --rows 1000 --out postgres --output-path seed.sql
```

이게 전부입니다.

## 실제로 어떻게 쓰나

### 기본: DDL에서 바로 데이터 생성

```sql
CREATE TABLE users (
  id INT PRIMARY KEY,
  email VARCHAR(100) UNIQUE NOT NULL,
  age INT,
  tier VARCHAR(10)
);

CREATE TABLE orders (
  id INT PRIMARY KEY,
  user_id INT NOT NULL,
  amount NUMERIC,
  FOREIGN KEY (user_id) REFERENCES users(id)
);
```

```bash
datagen --ddl schema.sql --rows 100 --out json --output-path data.json
```

`users`가 먼저 생성되고, `orders.user_id`는 실제 생성된 `users.id` 값에서 참조됩니다. FK 무결성은 자동으로 보장됩니다.

### 분포 오버라이드

실제 데이터는 균등 분포가 아닙니다. `--dist` 옵션으로 현실적인 분포를 적용할 수 있습니다:

```bash
# 나이: 평균 33세, 표준편차 7
datagen --ddl schema.sql --rows 1000 \
  --dist users.age:normal,mean=33,std=7 \
  --dist orders.amount:pareto,alpha=1.5,xm=1 \
  --dist tier:weighted,premium=10%,standard=70%,free=20% \
  --out postgres
```

지원하는 분포:
- **normal** — 연속형 수치 (나이, 점수 등)
- **poisson** — 카운트 데이터 (일일 주문 수 등)
- **pareto** — 헤비테일 (결제 금액, 파일 크기 등)
- **zipf** — 랭킹/카테고리 빈도
- **exponential** — 대기 시간, 간격
- **weighted** — 카테고리별 비율 직접 지정
- **peak** — 시간대별 몰림 (`created_at:peak,hours=9-11,18-20`)

### 라이브 DB에서 스키마 읽기

DDL 파일이 없어도 됩니다:

```bash
datagen \
  --schema-from-db \
  --db-url postgresql+psycopg://user:pass@localhost:5432/mydb \
  --rows 1000 \
  --out json
```

### 설정 파일로 관리

반복적인 생성은 TOML 설정 파일로 관리하면 편합니다:

```toml
ddl = "schema.sql"
rows = 500
out = "postgres"
seed = 42
strict_checks = true
table_rows = { users = 100, orders = 500, events = 2000 }
dist = [
  "users.age:normal,mean=33,std=7",
  "orders.amount:pareto,alpha=1.7,xm=1",
]
```

```bash
datagen --config datagen.toml
```

## 데이터 엔지니어를 위한 기능들

### 엣지 케이스 자동 주입

20행마다 하나씩 엣지 케이스가 자동으로 섞입니다:

- 문자열: 특수문자, 빈 문자열에 가까운 값, 최대 길이 근처 값
- 숫자: 0, 경계값
- 일반적으로 파이프라인에서 문제를 일으키는 패턴들

실제 프로덕션에서 터지는 문제의 상당수가 이런 엣지 케이스에서 옵니다. 스테이징에서 미리 잡을 수 있습니다.

### CHECK 제약조건 인식

```sql
CREATE TABLE products (
  id INT PRIMARY KEY,
  price NUMERIC CHECK (price BETWEEN 10 AND 10000),
  status VARCHAR(10) CHECK (status IN ('active', 'inactive', 'pending'))
);
```

이런 CHECK 조건을 인식해서 조건에 맞는 데이터를 생성합니다. `--strict-checks` 옵션으로 생성 후 검증까지 가능합니다.

### 검증 리포트

```bash
datagen --ddl schema.sql --rows 1000 \
  --strict-checks \
  --report-path report.json \
  --out json --output-path data.json
```

FK 위반, NOT NULL 위반, UNIQUE 충돌, CHECK 위반을 JSON 리포트로 뽑아줍니다. CI에서 자동화하기 좋습니다.

### 다양한 출력 포맷

```bash
datagen --ddl schema.sql --rows 1000 --out postgres   # PostgreSQL INSERT
datagen --ddl schema.sql --rows 1000 --out mysql       # MySQL INSERT
datagen --ddl schema.sql --rows 1000 --out bigquery    # BigQuery INSERT
datagen --ddl schema.sql --rows 1000 --out json        # JSON
datagen --ddl schema.sql --rows 1000 --out csv         # CSV (테이블별 파일)
datagen --ddl schema.sql --rows 1000 --out parquet     # Parquet
```

BigQuery는 `INSERT ALL` 모드와 dataset-qualified 테이블명도 지원합니다.

## AI 도구와 함께 쓰기

솔직히 CLI 옵션이 많습니다. `--dist`, `--table-rows`, `--strict-checks` 등등 조합이 다양합니다.

이건 의도적인 부분이 있습니다. **AI 코딩 도구(Claude Code 등)와 함께 쓰는 것**을 염두에 두고 설계했습니다.

예를 들어 Claude Code에 이렇게 요청할 수 있습니다:

> "이 schema.sql로 1만 건 데이터 생성해줘. users.age는 정규분포로, orders.amount는 파레토 분포로. PostgreSQL INSERT문으로 뽑아줘."

AI가 적절한 CLI 명령을 조합해주기 때문에, 옵션을 외울 필요 없이 자연어로 원하는 데이터를 만들 수 있습니다. CLI가 제공하는 세밀한 제어와 AI의 자연어 인터페이스가 결합되는 셈입니다.

## 앞으로의 계획

현재 loadforge는 관계형 데이터베이스 중심이지만, 로드맵이 있습니다:

### DynamoDB 지원
NoSQL 환경에서도 테스트 데이터가 필요합니다. 파티션 키와 소트 키 구조를 이해하고, DynamoDB에 맞는 데이터를 생성할 수 있도록 확장할 예정입니다.

### 더 다양한 데이터 분포
현재도 normal, poisson, pareto, zipf 등을 지원하지만, power law 분포를 포함해 더 다양한 분포를 추가할 계획입니다. 현실 데이터를 더 정확하게 모사할수록 테스트의 가치가 올라갑니다.

### 데이터 파이프라인 엣지 케이스 특화
데이터 엔지니어링에서 자주 마주치는 구체적인 문제 상황들을 타겟팅합니다:

- **핫스팟(데이터 몰림)** — 특정 파티션 키에 데이터가 집중되는 현상. 파티셔닝된 테이블이나 분산 시스템에서 skew를 일으키는 주범입니다.
- **타임스탬프 엣지 케이스** — 자정 경계, 타임존 전환, 윤초 근처 값
- **NULL/빈 값 패턴** — 실제로 파이프라인을 깨뜨리는 다양한 빈 값 조합
- **대량 중복** — 중복 제거 로직이 제대로 동작하는지 검증

파이프라인이 프로덕션에서 터지기 전에, 스테이징에서 미리 잡는 것이 목표입니다.

## 마치며

몇 년간 "이런 게 있으면 좋겠다"고만 생각했던 도구를 결국 만들었습니다. 늦었지만, 만들고 나니 왜 진작 안 했나 싶습니다.

데이터 엔지니어링에서 테스트 데이터는 늘 뒷전으로 밀리는 주제입니다. 하지만 좋은 테스트 데이터가 있으면 파이프라인의 신뢰도가 확연히 달라집니다.

관심 있으시면 한번 써보세요:

```bash
pipx install loadforge
```

- GitHub: [hyangminj/datagen](https://github.com/hyangminj/datagen)
- PyPI: [loadforge](https://pypi.org/project/loadforge/)

피드백이나 이슈는 GitHub Issues로 남겨주시면 감사하겠습니다.
