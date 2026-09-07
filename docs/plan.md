# persona-ingestion — 기획 문서

## 목적

사용자가 로컬에서 적법하게 확보한 structured transcript/script 파일을 private corpus로 변환한다. v1의 입력은 Gintama transcript TXT와 BBC Sherlock shooting script PDF이며, 결과는 화자 기준으로 정제된 canonical utterance다.

## 데이터 경계

- 코드와 synthetic fixture만 Git에 둔다.
- 실제 raw PDF/TXT, parsed JSONL/Parquet, vector dump, backup은 Git 밖 private PV에 둔다.
- URL crawler, transcript scraper/indexer, subtitle downloader는 구현하지 않는다.
- `rights_status: private_unverified` data는 public demo, benchmark report, screenshot으로 승격하지 않는다.

## 입력과 출력

| Persona | Input adapter | target speaker alias |
| --- | --- | --- |
| `gintoki` | timestamp + speaker transcript TXT | `Gintoki`, `Gin`, `Gintoki Sakata` |
| `sherlock` | shooting script PDF | `SHERLOCK` |

Canonical record:

```json
{
  "utterance_id": "sherlock-s01e01-p003-0007",
  "character_id": "sherlock",
  "series": "sherlock",
  "episode": "S01E01",
  "source_id": "sherlock-s01e01",
  "source_locator": {"page": 3},
  "speaker_raw": "SHERLOCK",
  "speaker_normalized": "sherlock",
  "language": "en",
  "text": "…",
  "parser_version": "sherlock-pdf-v1",
  "source_sha256": "…"
}
```

`source_manifest.yaml`에는 source URL/file hash, acquisition date, rights status, allowed scope, retention/delete date를 남긴다.

## 책임

- intake manifest와 SHA-256 생성
- Gintama TXT adapter, Sherlock PDF adapter
- line merge, whitespace/Unicode normalization, speaker alias mapping
- stage direction/sign/translator note 분리와 quarantine record
- canonical JSONL/Parquet 및 parse report 생성
- 이후 phase에서 chunk/embed/Qdrant upsert Job 제공

## 제외

- ASR, diarization, timestamp-subtitle alignment, 자동 번역
- raw corpus 공개, crawler, web download
- online chat API나 GPU model serving

## 구현 루프

### Loop 1 — synthetic transcript parser

- Gintama와 같은 timestamp/speaker fixture를 입력으로 target utterance만 추출한다.
- 완료 조건: multiline merge, alias mapping, non-target exclusion, malformed line quarantine unit test.

### Loop 2 — canonicalization/report

- JSONL/Parquet, source manifest, source hash, parse stats를 만든다.
- 완료 조건: 같은 입력을 다시 실행하면 output hash가 같고 failure reason이 남음.

### Loop 3 — Sherlock PDF adapter

- layout-preserving text extraction 뒤 speaker heading state machine을 구현한다.
- 완료 조건: page header/footer와 scene direction이 dialogue에 섞이지 않는 synthetic PDF test.

### Loop 4 — private Kubernetes Job

- private PV를 read-only mount해 pipeline image가 result PV에만 쓴다.
- 완료 조건: raw data가 container image/log/Git에 남지 않음.

### Loop 5 — indexing handoff

- target utterance를 token-aware window로 chunk하고 embedding/Qdrant upsert를 수행한다.
- 완료 조건: persona/workspace filter와 source locator가 Qdrant payload에 유지됨.

## 품질 게이트

- target speaker leakage 0건
- locator 누락 0건
- parser failure는 quarantine에 원인과 함께 기록
- episode/source별 20개 표본을 원문과 대조
- 공개 test와 benchmark에는 synthetic data만 사용
