# doc_chunker — 구조 단위 청킹 컴포넌트

[`doc_parser`](../doc_parser/)가 읽은 문서를 **문서 자체의 구조(장·절·표)를 따라** 검색용 단위(Unit)로 자릅니다.
폴더 일괄 처리(읽기 + 자르기)도 여기 있습니다. LLM·네트워크를 쓰지 않습니다.

## 설치·사용

```bash
PY=/home/ubuntu/miniconda3/envs/stock-dataset/bin/python
$PY -m pip install -e /mnt/d/workspace/side_projects/doc_parser
$PY -m pip install -e /mnt/d/workspace/side_projects/doc_chunker --no-deps

$PY -m doc_chunker 자료폴더/                                    # 파일별 단위 수·크기·경고·건너뜀·중복
$PY -m doc_chunker 자료폴더/ --max-tokens 3000 --out units.jsonl --show 2
```

```python
from doc_chunker import chunk, parse_and_chunk
from doc_parser import parse_file

units = chunk(parse_file("매뉴얼.pdf"), max_tokens=3000)   # [Unit(id, heading_path, loc_start, loc_end, text, tokens, part)]
res = parse_and_chunk(["자료폴더/"])                        # res.units, res.docs, res.skipped, res.duplicates

# 다른 모델의 토크나이저로 크기를 재려면
units = chunk(doc, 2000, count=lambda s: len(tokenizer.encode(s)))
```

## 청크 미리보기 화면 + A2A (`serve`)

문서가 검색 단위로 어떻게 잘리는지 사람이 보고, 다른 에이전트는 A2A로 같은 일을 맡깁니다(같은 저장소 — 에이전트가 자른 문서도 목록에 보임).
구조: `../reference/AGENT_PRODUCT.md`.

```bash
$PY -m pip install -e ".[app]" --no-deps    # fastapi·uvicorn은 doc_parser[app]과 같음 (stock-dataset엔 이미 있음)
$PY -m doc_chunker serve                     # http://127.0.0.1:8769 (127.0.0.1 고정), 데이터 doc_chunker/data/
```

- **화면**: 파일을 놓으면 상한(기본 3,000 토큰)으로 바로 자름 → **단위 지도**(칸 너비 = 토큰 수, 진하기 = 상한 대비 채운 정도,
  누르면 그 단위로) + 단위 목록(절 이름·쪽/줄 범위·부분 n/m·토큰 막대·본문, 표는 표로). 상한을 바꿔 다시 자르면 새 미리보기로
  남아 비교 가능. JSONL 내려받기. 같은 파일·같은 상한은 이전 결과를 엶. 짧은 장 여러 개가 한 단위로 묶이면 `heading_path`는
  비고 본문 첫 제목으로 표시("제1장 … 외 2개 절").
- **A2A 1.0** (`doc_parser.a2a` 공용 모듈): `SendMessage` — 파일은 file part `{raw, filename}` · `{url: file:///…}` · data `{path}`,
  상한은 data `{"max_tokens": 1500}`. 결과 artifact `units`(data), `metadata.reviewUrl` = 미리보기 화면. 자르기는 요청 안에서 끝나서 취소 없음.

```bash
$PY -m pytest -q tests/test_app.py           # 화면 API·A2A·재시작 E2E
$PY tools/browser_check.py [--doc FILE]      # 실제 Chromium 23개 점검 (기본: 215쪽 매뉴얼), 폭 3종·다크
```

## 청킹 규칙

1. 파일 전체가 `max_tokens` 이하면 단위 1개 (대부분의 업무 파일)
2. 크면 제목 트리를 따라: 들어가는 절은 통째로, 작은 형제 절들은 묶어서 상한까지
3. 그래도 큰 절은 블록 경계 → 문장·줄 경계. 큰 표는 행 단위로 자르고 **조각마다 머리 행 반복**, 행은 절대 안 쪼갬
4. 단위마다 맨 위에 `파일 · 문서 › 장 › 절 · p.12–p.14`와 **문서 정보**(첫머리: 시행일·작성일·버전·참석자)를 붙임
   → 검색으로 단위 하나만 와도 어느 버전·날짜인지 보임. 상한에는 이 머리 줄까지 포함

폴더 일괄(`parse_and_chunk`): 미지원 형식·깨진 파일은 건너뛰고 이유를 남김(한 파일 때문에 전체가 멈추지 않음),
내용이 같은 파일은 해시로 중복 표시, Office 잠금 파일(`~$`)·숨김 파일 제외.

토큰 수 기본값은 추정(글자 × 0.52 — Qwen3.8 토크나이저로 한국어 md에서 2026-11-13 측정한 0.517). 정확히 재려면 `count=`.

## 검증

- `python -m pytest -q` — 6개: CLI 진입점 E2E 3 + 미리보기 서버(API·A2A) E2E 3. CLI E2E (상한·문서 정보·조항 보존·토큰 계산 교체 / 나쁜 파일 묶음·같은 입력 2회 / 긴 docx)
- 긴 문서 회귀: [`tools/README.md`](tools/README.md). 2026-11-13 결과 — 215쪽 매뉴얼(docx, Word PDF 책갈피 有/無)에서
  절 표식 160/160, 500행 부록 표 500/500(중복 0), 표 조각 25개 모두 머리 행. 읽기+자르기 docx 0.9초, PDF 약 38초
