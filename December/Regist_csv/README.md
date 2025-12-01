# TenTenTown DataTable CSV Sync 매뉴얼

## 1) 준비물(프로젝트에 이미 포함되어 있어야 함)

### 필수 파일 2개

| 파일                                 | 위치                                                         | 용도                                |
| ---------------------------------- | ---------------------------------------------------------- | --------------------------------- |
| `dt_manifest.json`                 | `TenTenTown/RawData/dt_manifest.json`                      | DataTable ↔ RowStruct ↔ CSV 매핑 목록 |
| `sync_datatables_from_manifest.py` | `TenTenTown/Tools/Python/sync_datatables_from_manifest.py` | manifest를 읽어 DataTable을 CSV로 업데이트 |

---

## 2) 업데이트(가장 자주 하는 작업) — CSV 수정 후 반영

### Step-by-step

1. 엑셀/시트에서 값을 수정한 뒤 **CSV로 Export**
2. Export한 CSV를 프로젝트의 **RawData 동일 경로에 덮어쓰기**

   * 예: `TenTenTown/RawData/Character/Fighter/....csv`
3. 언리얼 에디터 실행
4. 상단 메뉴에서 실행:

   * **Tools → Execute Python Script…**
   * `TenTenTown/Tools/Python/sync_datatables_from_manifest.py` 선택
5. 업데이트 확인

   * 해당 DataTable을 열었을 때 값이 바뀌어 있으면 완료

---

## 3) 신규 데이터 추가(새로운 CSV / 새로운 DataTable 추가 시)

### A. CSV 파일 추가

1. CSV를 RawData에 추가합니다.

   * 예: `TenTenTown/RawData/Enemy/...`, `TenTenTown/RawData/Structure/...`

### B. DataTable 에셋 준비

1. Content Browser에 DataTable 에셋이 있어야 합니다.

   * 예: `/Game/Enemy/DataTable/DT_...`

### C. manifest에 매핑 1줄 추가

파일: `TenTenTown/RawData/dt_manifest.json`

아래 형식으로 `entries` 배열에 객체 하나를 추가합니다:

```json
{
  "dt_name": "DT_데이터테이블이름",
  "search_root": "/Game",
  "row_struct": "/Script/TenTenTown.구조체이름",
  "csv_rel": "RawData/상대경로/파일.csv"
}
```

#### 항목 뜻

| 키             | 의미              | 작성 기준                     |
| ------------- | --------------- | ------------------------- |
| `dt_name`     | DataTable 에셋 이름 | Content의 DataTable 이름 그대로 |
| `search_root` | DT 검색 범위        | 기본 `/Game`                |
| `row_struct`  | RowStruct 경로    | C++ USTRUCT의 Script 경로    |
| `csv_rel`     | CSV 상대경로        | 프로젝트 루트 기준 `RawData/...`  |

### D. 업데이트 실행

* **Tools → Execute Python Script…**로 `sync_datatables_from_manifest.py` 실행

---

## 4) 성공 여부 확인 방법(로그)

스크립트 실행 후 Output Log에 아래가 나오면 정상입니다.

* `Imported DataTable 'DT_...' - 0 Problems`
* 마지막에 `OK`가 증가하고 `FAIL=0` 상태

그리고 실제로 DT를 열었을 때 값이 CSV대로 들어가 있으면 완료입니다.

---

## 5) 실수 방지 체크리스트

| 체크          | 내용                                      |
| ----------- | --------------------------------------- |
| CSV 저장 위치   | 반드시 `TenTenTown/RawData/...` 아래         |
| manifest 경로 | `TenTenTown/RawData/dt_manifest.json`   |
| csv_rel     | `RawData/...` 상대경로로 작성(절대경로 X)          |
| row_struct  | `/Script/TenTenTown.***` 형태로 정확히        |
| 실행 방법       | **Tools → Execute Python Script…**로만 실행 |

