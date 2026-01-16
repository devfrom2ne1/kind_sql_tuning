# Redo 로그와 Undo 로그의 차이점
쉽게 비유하자면, Redo는 **"다시 하기"** 이고 Undo는 **"되돌리기"** 입니다.


## 1. Redo 로그 (다시 하기)
Redo 로그는 **"이미 완료된 변경 사항을 유실하지 않기 위해"** 기록하는 로그입니다.

* **핵심 역할:** 시스템 장애 발생 시, 마지막 성공 지점까지 데이터를 **재현(Replication)** 합니다.
* **보관 데이터:** 변경 **후**의 값 (New Value)
* **보장 원칙:** ACID 원칙 중 **영속성(Durability)**
* **작동 방식:** 
	1. 데이터 변경 시 로그를 먼저 기록합니다.
	2. 시스템이 비정상 종료 후 재시작되면, Redo 로그를 읽어 데이터 파일에 반영되지 않은 변경 건을 다시 실행합니다.
* **사용 목적:**
	1. Database Recovery( Media Recovery)
	2. Cache Recover( Instance Recovery 시 roll forward 단계)
	3. Fast Commit
* **매커니즘**:
	1. Log Force At Commit
	2. Fast Commit
	3. Write Ahead Logging


## 2. Undo 로그 (되돌리기)
Undo 로그는 **"변경 중인 데이터를 이전 상태로 되돌리기 위해"** 기록하는 로그입니다.

* **핵심 역할:** 작업 취소(Rollback) 시 데이터를 **원래대로 복구**하거나, 읽기 일관성을 유지합니다.
* **보관 데이터:** 변경 **전**의 값 (Old Value)
* **보장 원칙:** ACID 원칙 중 **원자성(Atomicity)** 및 **격리성(Isolation)**
* **작동 방식:**
    1. 데이터를 변경하기 직전, 이전 값을 Undo 영역에 복사해둡니다.
    2. 사용자가 `ROLLBACK`을 수행하면 이 로그를 사용해 데이터를 원복합니다.
    3. 다른 사용자가 수정 중인 데이터를 조회할 때, 일관성을 위해 수정 전 값을 보여주는 용도(MVCC)로도 사용됩니다.
* **사용 목적:**
	1. Transaction Rollback
	2. Transaction Recovery (Instance Recovery 시 rollback 단계)
	3. Read Consistency

---

## 3. 한눈에 비교하기

| 구분 | Redo 로그 | Undo 로그 |
| :--- | :--- | :--- |
| **목적** | 데이터 유실 방지 (Recovery) | 데이터 원복 (Rollback) |
| **기록 시점** | 변경 후의 상태 저장 | 변경 전의 상태 저장 |
| **핵심 키워드** | 다시 하기 (Re-do) | 취소 하기 (Un-do) |
| **복구 대상** | Commit 되었으나 디스크에 안 적힌 데이터 | Commit 되지 않은 채 중단된 데이터 |

---
## 💡 참고: Undo와 Redo의 위치
- **Undo 데이터**: SGA의 **Database Buffer Cache** 내 Undo 블록에 저장됨 (롤백/읽기 일관성용)
- **Redo 로그**: SGA의 **Redo Log Buffer**에 저장됨 (장애 복구용)
*Undo 블록의 변경사항 또한 Redo 로그 버퍼에 기록됩니다.*



> **요약**
> * **Redo:** "내가 한 일은 무슨 일이 있어도 끝까지 적용한다!"
> * **Undo:** "실수하거나 취소하면 예전 모습으로 돌아간다!"
