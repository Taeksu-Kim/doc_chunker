# doc_chunker — 프로젝트 규칙

공통 규칙은 상위 `../CLAUDE.md`를 따른다. 이 파일은 doc_chunker 전용.

- **여러 에이전트가 쓰는 컴포넌트다.** 공개 API(`chunk`, `parse_and_chunk`, `Unit` 필드, `count=`)를 바꾸면 쓰는 쪽(doc_writer)이
  깨진다. 바꿀 때는 doc_writer 테스트까지 돌린다.
- 의존은 doc_parser 한 방향뿐. 읽기 결함은 doc_parser에서 고친다(여기서 우회하지 않는다).
- LLM·네트워크를 넣지 않는다 (같은 입력 → 같은 단위).
- 분할 규칙을 바꾸면 긴 문서 회귀(`tools/README.md`)를 돌린다.
- 미리보기 화면·A2A(`serve`, `app.py`·`service.py`·`static/index.html`)는 선택 기능: 웹 의존성은 `[app]` extra에만.
  A2A 배관은 `doc_parser.a2a`를 쓴다(여기 복사하지 않는다). 화면을 바꾸면 `tools/browser_check.py`, 서버를 바꾸면 `tests/test_app.py`.
- 설치: `pip install -e . --no-deps` (doc_parser는 먼저 editable 설치)
