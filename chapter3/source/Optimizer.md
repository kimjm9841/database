# Optimizer

# 옵티마이저

## 1. 옵티마이저 종류

SQL 옵티마이저는 작업을 가장 효율적으로 수행할 수 있는 최적의 데이터 액세스 경로를 선택해주는 DBMS의 핵심 엔진이다. 옵티마이저는 크게 다음과 같은 2가지 종류로 나눌 수 있다.

### (1) 비용 기반 옵티마이저 (CBO)

비용 기반 옵티마이저는 사용자 쿼리를 위해 후보군이 될만한 실행계획들을 도출하고, 데이터 딕셔너리[**⁽¹⁾**](#주석)에 미리 수집해 둔 통계 정보를 이용해 각 실행계획의 예상비용을 산정하고, 그중 가장 낮은 비용의 실행계획 하나를 선택하는 옵티마이저다.

### (2) 규칙 기반 옵티마이저 (RBO)

규칙 기반 옵티마이저는 각 액세스 경로에 대한 우선순위 규칙에 따라 실행계획을 만드는 옵티마이저다. 데이터 특성을 나타내는 통계정보를 전혀 활용하지 않고 단순히 인덱스 구조, 연산자, 조건절 형태 등을 사용하는 규칙에만 의존하여 실행계획을 선택하기 때문에 여러 변수가 존재하는 대량의 데이터를 처리하는 데 부적합하며 최근에는 대부분의 RDBMS들이 CBO를 선택한다.

## 2. 통계 정보

### (1) 테이블 통계 (**mysql.innodb_table_stats)**

| 통계항목 | 설명 |
| --- | --- |
| DATABASE_NAME | 데이터베이스의 이름 |
| TABLE_NAME | 테이블의 이름 |
| LAST_UPDATE | 레코드가 마지막으로 수정된 시점의 타임스탬프 |
| N_ROWS | 테이블 전체 레코드 건수 |
| CLUSTERED_INDEX_SIZE | 기본 인덱스(클러스터 인덱스)가 차지하는 페이지 수 |
| SUM_OF_OTHER_INDEX_SIZES | 기본 인덱스를 제외한 인덱스들이 차지하는 페이지 수의 총합 |

### (2) 인덱스 통계 (**mysql.innodb_index_stats)**

| 통계항목 | 설명 |
| --- | --- |
| DATABASE_NAME | 데이터베이스의 이름 |
| TABLE_NAME | 테이블의 이름 |
| INDEX_NAME | 인덱스의 이름 |
| LAST_UPDATE | 레코드가 마지막으로 수정된 시간 |
| STAT_NAME | STAT_VALUE가 갖는 통계의 이름 |
| STAT_VALUE | STAT_NAME에서 명명된 통계의 값 |
| SAMPLE_SIZE | 제공된 추정치에 대해 샘플링된 페이지의 수 |
| STAT_DESCRIPTION | STAT_NAME에서 명명된 통계에 대한 설명 |

### (3) 통계 정보 갱신

innodb_stats_auto_recalc 시스템 설정 변수의 값을 사용해 통계 정보가 자동으로 갱신될지 여부를 설정할 수 있다. 기본값은 ON이며, OFF로 설정하면 영구적인 통계 정보를 이용해 통계 정보가 자동으로 갱신되는 것을 막을 수 있다.

테이블을 생성할 때 STATS_AUTO_RECALC 옵션을 이용하면 통계 정보를 자동으로 수집할지 여부를 테이블 단위로 조정할 수 있다. 값이 1이면 테이블의 통계 정보를 자동 수집하며, 0이면 ANALYZE TABLE 명령을 실행할 때만 테이블의 통계 정보가 수집된다. 또한 테이블을 생성할 때 별도로 옵션을 설정하지 않거나 DEFAULT로 설정하면, 테이블의 통계 정보 수집을 innodb_stats_auto_recalc 시스템 설정 변수의 값으로 결정한다.

innodb_stats_transient_sample_pages은 자동으로 통계 정보 수집이 실행될 때 몇 개의 페이지를 임의로 샘플링해서 분석하고 그 결과를 통계 정보로 활용할지를 의미하며, 기본값은 8이다. innodb_stats_persistent_sample_pages은 ANALYZE TABLE 명령이 실행되면 몇 개의 페이지를 임의로 샘플링해서 분석하고 그 결과를 영구적인 통계 정보 테이블에 저장하고 활용할지를 의미하며, 기본값은 20이다.

### (4) 히스토그램

기존의 통계 정보는 테이블의 전체 레코드 건수와 인덱스된 칼럼이 가지는 유니크한 값의 개수 정도였다. 하지만 실제 응용 프로그램의 데이터는 항상 균등한 분포도를 가지지 않는다. 이런 단점을 보완하기 위해 히스토그램이 도입되었다. 히스토그램을 사용하면 특정 칼럼의 데이터 분포도를 참조할 수 있다. 비록 모든 값에 대한 분포도 정보를 가지지는 않지만 각 범위별로 레코드의 건수와 유니크한 값의 개수 정보를 가지기 때문에 훨씬 정확한 예측을 할 수 있다.

히스토그램 정보가 없으면 옵티마이저는 데이터가 균등하게 분포돼 있을 것으로 예측한다. 하지만 히스토그램이 있으면 특정 범위의 데이터가 많고 적음을 식별할 수 있다. 이를 기반으로 어느 테이블을 먼저 읽어야 조인의 횟수를 줄일 수 있을지 옵티마이저가 더 정확히 판단할 수 있다.

히스토그램은 데이터 분포 파악을 위해 테이블의 모든 행을 분석하기 때문에 Table Full Scan이 발생할 수 밖에 없다. 분석 과정에서 CPU 사용량 증가, 디스크 I/O 부하, 쿼리 성능 저하 등이 발생할 수 있기 때문에 사용자가 적은 시간에 수동으로 관리해줘야 한다.

## 3. 옵티마이저 힌트

옵티마이저는 대부분 좋은 선택을 하지만, 완벽하진 않다. SQL 문이 복잡할수록 실수할 가능성도 늘어난다. 이럴 때 통계 정보에 담을 수 없는 데이터 또는 업무 특성을 활용해 개발자가 직접 더 효율적인 액세스 경로를 찾아낼 수 있으며, 이를 옵티마이저에게 알려주기 위해 사용하는 것이 옵티마이저 힌트이다. 옵티마이저 힌트는 영향 범위에 따라 다음 4개 그룹으로 나눌 수 있다.

- 인덱스: 특정 인덱스의 이름을 사용할 수 있는 옵티마이저 힌트
- 테이블: 특정 테이블의 이름을 사용할 수 있는 옵티마이저 힌트
- 쿼리 블록: 특정 쿼리 블록에 사용할 수 있는 옵티마이저 힌트로서, 특정 쿼리 블록의 이름을 명시하는 것이 아니라 힌트가 명시된 쿼리 블록에 대해서만 영향을 미치는 옵티마이저 힌트
- 글로벌: 전체 쿼리에 대해서 영향을 미치는 힌트

| 힌트 이름 | 설명 | 영향 범위 |
| --- | --- | --- |
| MAX_EXECUTION_TIME | 쿼리의 실행 시간 제한 | 글로벌 |
| BNL, NO_BNL | 블록 중첩 루프 조인의 사용 여부 제어 | 쿼리 블록, 테이블 |
| JOIN_FIXED_ORDER | FROM 절에 명시된 테이블 순서대로 조인 실행 | 쿼리 블록 |
| JOIN_ORDER | 힌트에 명시된 테이블 순서대로 조인 실행 | 쿼리 블록 |
| JOIN_PREFIX | 힌트에 명시된 테이블을 조인의 드라이빙 테이블로 조인 실행 | 쿼리 블록 |
| JOIN_SUFFIX | 힌트에 명시된 테이블을 조인의 드리븐 테이블로 조인 실행 | 쿼리 블록 |
| SKIP_SCAN, NO_SKIP_SCAN | 인덱스 스킵 스캔 사용 여부 제어 | 테이블, 인덱스 |
| INDEX, NO_INDEX | GROUP BY, ORDER BY, WHERE 절의 처리를 위한 인덱스 사용 여부 제어 | 인덱스 |

이외에도 다양한 옵티마이저 힌트가 있다. 옵티마이저 힌트는 SQL 문에서 주석 내에 + 기호와 함께 사용한다. 아래의 SQL 문은 WHERE 절의 처리를 위해 인덱스를 사용하도록 설정한다. 이때 인덱스 수준의 힌트를 사용하기 위해 인덱스 이름을 명시할 때는 반드시 그 인덱스를 가진 테이블명을 먼저 명시해야 한다. 

```sql
EXPLAIN
SELECT /*+ INDEX(employees ix_firstname) */ *
from employees
WHERE first_name = 'Matt';
```

## 4. 옵티마이저에 영향을 미치는 요소

통계정보와 옵티마이저 힌트 외에도 다음과 같은 요소가 옵티마이저에게 영향을 미친다.

### (1) SQL과 연산자 형태

결과가 같더라도 SQL을 어떤 형태로 작성했는지 또는 어떤 연산자를 사용했는지에 따라 옵티마이저가 다른 선택을 할 수 있으며 쿼리 성능에 영향을 미친다.

### (2) 인덱스, 클러스터, 파티션 등 옵티마이징 팩터[**⁽²⁾**](#주석)

쿼리를 똑같이 작성해도 인덱스, 클러스터[**⁽³⁾**](#주석), 파티션[**⁽⁴⁾**](#주석) 등을 구성했는지, 그리고 어떤 식으로 구성했는지에 따라 실행계획과 성능이 크게 달라진다.

### (3) 제약 설정

DBMS에 설정한 PK, FK, Check, Not Null 같은 제약들은 데이터 무결성을 보장해 줄뿐만 아니라 옵티마이저가 쿼리 성능을 최적화하는 데 매우 중요한 메타 정보로 활용된다.

### 주석

---

[**1.**](#1-옵티마이저-종류) 데이터베이스에 대한 정보를 저장하는 시스템의 중앙 저장소

[**2.**](#4-옵티마이저에-영향을-미치는-요소) 쿼리 성능을 향상시키기 위해 데이터베이스의 구조와 접근 방식을 조정하는 요소

[**3.**](#4-옵티마이저에-영향을-미치는-요소) 여러 개의 DB를 수평적인 구조로 구축하여 분산 처리하는 방식

[**4.**](#4-옵티마이저에-영향을-미치는-요소) 논리적으로 하나의 테이블이지만 실제로는 여러 개의 테이블로 분리해서 관리하는 기능