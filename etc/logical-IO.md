# SQL 트레이스 논리적 I/O(Logical I/O) 지표 정리

오라클 SQL 트레이스 결과에서 **논리적 I/O**는 `query`와 `current`라는 두 가지 항목의 합으로 계산됩니다.

> **공식: Logical I/O = query + current**

---

## 1. 주요 지표 설명

### ① query (Consistent Read)
* **의미**: 읽기 일관성(Consistent Read) 모드에서 읽은 블록 수입니다.
* **특징**: 
    * 주로 `SELECT` 문에서 데이터를 조회할 때 발생합니다.
    * 쿼리 시작 시점의 데이터를 보여주기 위해 **Undo 세그먼트**를 참조하여 과거 상태로 되돌린 블록을 읽는 과정이 포함됩니다.
    * 일반적인 데이터 조회 성능의 척도가 됩니다.

### ② current (Current Read)
* **의미**: 현재 시점(Current Mode)의 데이터를 읽은 블록 수입니다.
* **특징**:
    * 주로 `INSERT`, `UPDATE`, `DELETE` 같은 DML 작업이나 `SELECT FOR UPDATE` 시 발생합니다.
    * 과거 시점이 아닌, 지금 현재 메모리에 있는 최신 상태의 블록을 읽습니다.
    * 인덱스 변경, 세그먼트 헤더 수정 등 시스템 내부적인 변경 작업 시에도 발생합니다.

---

## 2. query vs current 차이점 비교

| 구분 | query (Consistent Read) | current (Current Mode) |
| :--- | :--- | :--- |
| **데이터 시점** | 쿼리 시작 시점 (SNC 기반) | 현재 실시간 시점 |
| **핵심 용도** | 데이터 조회 및 정합성 유지 | 데이터 변경 및 최신 데이터 반영 |
| **주요 SQL** | 일반적인 `SELECT` | `INSERT`, `UPDATE`, `DELETE` |
| **Undo 참조** | 필요 시 Undo 블록 참조함 | Undo를 참조하지 않고 최신본 사용 |

---

## 3. 요약 및 성능 팁

* **논리적 I/O가 높다는 것**은 CPU 사용량이 많다는 것을 의미하며, 결국 성능 저하의 원인이 됩니다.
* 트레이스 결과에서 `disk` 항목(Physical I/O)이 낮더라도 `query`나 `current` 수치가 과도하게 높다면, **인덱스 스캔 효율성**이나 **SQL 튜닝**을 검토해야 합니다.
* 보고서의 `Row Source Operation` 부분을 함께 살펴보면, 어떤 단계에서 논리적 I/O가 많이 발생했는지 정확히 파악할 수 있습니다.
