# 오라클 다이렉트 패스 I/O (Direct Path I/O) 정리

다이렉트 패스 I/O는 버퍼 캐시(SGA)를 거치지 않고, **프로세스의 PGA와 디스크 간에 데이터를 직접 주고받는 방식**입니다. 대용량 데이터 처리 시 성능을 최적화하기 위해 사용됩니다.

---

## 1. Direct Path I/O vs Buffered I/O 비교

| 구분 | Buffered I/O (일반 방식) | Direct Path I/O |
| :--- | :--- | :--- |
| **데이터 경로** | 디스크 ↔ **SGA(Buffer Cache)** ↔ PGA | 디스크 ↔ **PGA (직접 전송)** |
| **장점** | 한 번 읽은 데이터의 재사용성 높음 | 대량 처리 시 캐시 경합 및 오염 방지 |
| **주요 용도** | 일반적인 트랜잭션 (OLTP) | 대량 데이터 로딩, 배치 작업 (DW/OLAP) |
| **단점** | 대량 데이터 읽기 시 기존 캐시 밀려남 | 매번 디스크에서 읽어야 함 (재사용 불가) |

---

## 2. Direct Path Read (읽기)

SGA를 거치지 않고 디스크에서 PGA로 데이터를 직접 읽어오는 경우입니다.

* **주요 발생 상황**
    * **병렬 쿼리(Parallel Query)**: 여러 슬레이브 프로세스가 동시에 데이터를 읽을 때.
    * **대규모 Full Table Scan**: 테이블 크기가 버퍼 캐시에 담기 너무 클 때 (Adaptive Direct Path Read).
    * **Temp Segment 읽기**: 정렬(Sort)이나 해시 조인 중 메모리가 부족해 Temp 영역에 썼던 데이터를 다시 읽을 때.
* **관련 대기 이벤트**: `direct path read`, `direct path read temp`

---

## 3. Direct Path Write (쓰기)

SGA에 기록하지 않고 데이터를 디스크에 직접 쓰는 방식입니다.

* **주요 발생 상황**
    * **Direct Load**: `INSERT /*+ APPEND */` 힌트를 사용한 대용량 입력.
    * **CTAS / IAS**: `CREATE TABLE AS SELECT` 또는 `INSERT AS SELECT` 작업.
    * **Parallel DML**: 병렬로 수행되는 데이터 변경 작업.
    * **Temp Segment 쓰기**: 정렬 작업 중 PGA 공간 부족으로 임시 영역(Temp)에 기록할 때.
* **관련 대기 이벤트**: `direct path write`, `direct path write temp`

---

## 4. 특징 및 장점

1.  **버퍼 캐시 오염(Pollution) 방지**: 대형 테이블을 읽을 때 기존에 캐싱된 자주 쓰는 데이터(Hot Block)들이 밀려나는 것을 막습니다.
2.  **래치(Latch) 경합 감소**: 버퍼 캐시 탐색 과정이 생략되므로 `cache buffers chains` 래치나 `buffer busy waits` 현상을 피할 수 있습니다.
3.  **고속 기록**: Direct Path Write는 HWM(High Water Mark) 이후 영역에 데이터를 직접 밀어 넣으므로 로그 생성량이 적고 속도가 매우 빠릅니다.

---

## 5. 주의사항

* **체크포인트(Checkpoint) 발생**: Direct Path Read 수행 전, 메모리에 있는 변경된 데이터(Dirty Buffer)를 디스크에 기록해야 하므로 일시적인 지연이 발생할 수 있습니다.
* **배타적 잠금(Locking)**: `APPEND` 모드 작업 시 해당 테이블에 강력한 락(Exclusive)이 걸려 다른 세션의 DML이 차단될 수 있습니다.
