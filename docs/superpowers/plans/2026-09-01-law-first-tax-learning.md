# 법령 중심 세법 학습 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 국세기본법·국세징수법·법인세법·소득세법·부가가치세법의 전체 조문을 찾을 수 있고, 핵심 조문은 법률 원문부터 시행령·초보자 해설·과세쟁점·판례까지 한 페이지에서 읽을 수 있는 정적 모바일 학습 메뉴를 추가한다.

**Architecture:** 법률별 JSON 원문과 공유 학습 연결 모델을 검증한 뒤, 법률 목록·조문 목록·조문 상세 HTML을 원자적으로 생성한다. 기존 기출·최신판례·대시보드 파이프라인은 그대로 사용하고, 조문·시행령·쟁점·판례·기출은 안정 ID로만 연결한다. 콘텐츠는 HTML에 포함하고 검색·필터는 자바스크립트가 없어도 본문을 읽을 수 있는 점진적 향상으로 구현한다.

**Tech Stack:** Python 3.12 표준 라이브러리, JSON Schema 계약, 기존 `tax_study.pages` 정적 생성기, 자체 포함형 HTML/CSS, `unittest`, GitHub Pages (`main:/docs`).

---

### Task 1: 실패하는 법령 데이터 계약 테스트 작성

**Files:**
- Create: `tests/test_laws.py`
- Create: `tests/fixtures/laws/minimal-five-law-catalog.json`
- Test: `tests/test_laws.py`

- [ ] **Step 1: 최소 5개 법률 fixture와 실패 테스트를 작성한다**

```python
def test_catalog_requires_exact_five_core_laws(self):
    payload = json.loads(FIXTURE.read_text(encoding="utf-8"))
    self.assertEqual(
        [row["law_code"] for row in payload["laws"]],
        ["NTBA", "NCTA", "CTA", "ITA", "VAT"],
    )
    with self.assertRaises(laws.LawDataError):
        laws.validate_law_catalog({"laws": payload["laws"][:4]})

def test_article_requires_version_and_learning_references(self):
    payload = json.loads(FIXTURE.read_text(encoding="utf-8"))
    article = payload["laws"][0]["articles"][0]
    self.assertEqual(article["article_id"], "NTBA-16")
    self.assertIn("versions", article)
    self.assertIn("issue_ids", article)
    broken = copy.deepcopy(payload)
    broken["laws"][0]["articles"][0].pop("versions")
    with self.assertRaises(laws.LawDataError):
        laws.validate_law_catalog(broken)
```

- [ ] **Step 2: 테스트가 아직 존재하지 않는 모듈 때문에 실패하는지 확인한다**

Run: `PYTHONPATH=src:tests python3 -m unittest tests.test_laws -v`

Expected: FAIL with `ModuleNotFoundError` or `cannot import name 'validate_law_catalog'`.

- [ ] **Step 3: 테스트 파일과 fixture만 커밋한다**

```bash
git add tests/test_laws.py tests/fixtures/laws/minimal-five-law-catalog.json
git commit -m "test: define five-law catalog contract"
```

### Task 2: 공식 법령 원문 수집·정제 CLI 추가

**Files:**
- Create: `tools/import_laws.py`
- Create: `tests/fixtures/laws/api/ntba-current.json`
- Create: `tests/test_law_import.py`
- Modify: `data/exam-law-source-manifest.json` only when an official version tuple is rechecked

- [ ] **Step 1: bounded importer의 실패 경계를 테스트한다**

```python
def test_importer_rejects_non_core_law(self):
    manifest = {"laws": [{"law_code": "ETAX", "law_name": "개별소비세법"}]}
    with self.assertRaises(ImportLawError):
        select_core_law_versions(manifest, as_of="2026-09-01")

def test_importer_requires_official_https_source_and_preserves_article_text(self):
    source = json.loads(API_FIXTURE.read_text(encoding="utf-8"))
    selected = normalize_official_response(source, law_code="NTBA")
    self.assertEqual(selected["law_code"], "NTBA")
    self.assertTrue(selected["articles"][0]["versions"][0]["official_url"].startswith("https://www.law.go.kr/"))
    self.assertIn("text", selected["articles"][0]["versions"][0])
```

- [ ] **Step 2: importer를 구현한다**

`tools/import_laws.py`는 기존 `data/exam-law-source-manifest.json`의 NTBA·NCTA·CTA·ITA·VAT만 선택하고, 공식 기준일 이하의 최신 버전을 고른다. HTTP 요청은 30초 timeout, 응답 8MB 제한, HTTPS 국가법령정보센터 도메인 allowlist, UTF-8 strict decoding을 사용한다. 원문 항목에는 `article_id`, `article_number`, `title`, `text`, `versions`, `decree_article_ids`를 기록하고, 결과를 임시 디렉터리에 쓴 뒤 `data/laws/`로 원자 교체한다.

- [ ] **Step 3: fixture와 실제 수집 명령을 실행한다**

Run: `PYTHONPATH=src python3 tools/import_laws.py --manifest data/exam-law-source-manifest.json --output data/laws --as-of 2026-09-01`

Expected: `catalog.json`, `ntba.json`, `ncta.json`, `cta.json`, `ita.json`, `vat.json`이 생성되고 선택된 버전·조문 수·SHA-256이 출력된다. 네 개 제외 법률 파일은 생성되지 않는다.

- [ ] **Step 4: importer를 커밋한다**

```bash
git add tools/import_laws.py tests/test_law_import.py tests/fixtures/laws/api data/laws
git commit -m "feat: import five core tax law sources"
```

### Task 3: 법령 스키마·로더·참조 검증 구현

**Files:**
- Create: `data/laws.schema.json`
- Create: `src/tax_study/laws.py`
- Modify: `src/tax_study/__init__.py`
- Modify: `tests/test_laws.py`

- [ ] **Step 1: 스키마와 Python API의 필수 필드를 고정한다**

`catalog.json`은 `schema_version`, `source_as_of`, `laws`를 갖고, 법률 코드는 정확히 `NTBA`, `NCTA`, `CTA`, `ITA`, `VAT`만 허용한다. 조문 버전은 `version_id`, `status`, `effective_date`, `text`, `official_url`, `source_sha256`, `checked_at`을 필수로 한다. `decree_article_ids`, `issue_ids`, `precedent_ids`, `exam_question_ids`는 중복 없는 문자열 배열이어야 한다.

- [ ] **Step 2: 로더·모델 함수의 실패 테스트를 추가한다**

```python
def test_build_law_index_model_is_deterministic_and_has_only_core_laws(self):
    first = laws.build_law_index_model(DATA_DIR)
    second = laws.build_law_index_model(DATA_DIR)
    self.assertEqual(first, second)
    self.assertEqual([row["law_code"] for row in first["laws"]], ["NTBA", "NCTA", "CTA", "ITA", "VAT"])

def test_missing_decree_reference_fails_closed(self):
    broken = copy.deepcopy(self.catalog)
    broken["laws"][0]["articles"][0]["decree_article_ids"] = ["NTBA-D-DOES-NOT-EXIST"]
    with self.assertRaises(laws.LawDataError):
        laws.validate_law_catalog(broken)
```

- [ ] **Step 3: API를 구현한다**

구현할 공개 함수는 `load_law_catalog(path)`, `validate_law_catalog(payload)`, `build_law_index_model(path)`, `build_law_article_model(path, law_code, article_id)`다. 모든 파일은 symlink·비정규 파일·8MB 초과·잘못된 UTF-8을 거부한다. 공식 링크는 `https://www.law.go.kr/`로 시작하는 URL만 허용하고, unknown ID·중복 ID·제외 법률·비어 있는 원문은 `LawDataError`로 중단한다.

- [ ] **Step 4: 패키지 lazy export와 계약 테스트를 통과시킨다**

Run: `PYTHONPATH=src:tests python3 -m unittest tests.test_laws -v`

Expected: 모든 법령 계약 테스트 PASS.

- [ ] **Step 5: 커밋한다**

```bash
git add data/laws.schema.json src/tax_study/laws.py src/tax_study/__init__.py tests/test_laws.py
git commit -m "feat: validate law and decree catalog"
```

### Task 4: 기존 쟁점·기출·판례를 조문 ID에 매핑

**Files:**
- Create: `data/laws/learning.json`
- Create: `tools/build_law_learning_map.py`
- Create: `tests/test_law_learning_map.py`
- Modify: `data/curriculum.json` only for explicit article references that are absent

- [ ] **Step 1: 기존 116개 topic의 법률·조문 매핑 실패 테스트를 작성한다**

```python
def test_all_116_topics_map_to_one_of_five_laws_and_an_explicit_article(self):
    model = build_law_learning_map(CURRICULUM, EXAM_ISSUES, PRECEDENTS)
    self.assertEqual(len(model["topics"]), 116)
    self.assertEqual({row["primary_law"] for row in model["topics"]}, CORE_LAWS)
    self.assertTrue(all(row["article_ids"] for row in model["topics"]))

def test_unknown_article_mapping_is_rejected(self):
    broken = copy.deepcopy(self.law_catalog)
    broken["topics"][0]["article_ids"] = ["CTA-9999"]
    with self.assertRaises(LawLearningMapError):
        validate_law_learning_map(broken, self.law_catalog)
```

- [ ] **Step 2: 매핑 생성기를 구현한다**

`tools/build_law_learning_map.py`는 `curriculum.json`의 `laws_and_articles`, 70개 기출 쟁점 overlay의 `law_map`, `data/exam-important-precedents.json`, `curriculum.json`의 cases를 읽어 `article_ids`, `issue_ids`, `precedent_ids`, `exam_question_ids`를 만든다. 조문 번호가 없는 topic은 자동 추론하지 않고 오류 목록으로 중단한다. 직접 연결과 related 연결을 별도 필드로 보존한다.

- [ ] **Step 3: 각 핵심 조문에 초보자 해설·쟁점·조사·회계 필드를 채운다**

`learning.json`의 article overlay는 `beginner_explanation`, `issues`, `audit_application`, `accounting_tax_adjustment`, `status`를 갖는다. 116개 핵심 topic은 `status: "core"`, 나머지 조문은 `status: "source_only"`로 표시한다. 해설이 공식 법령 문장인 것처럼 보이지 않도록 원문·학습용 텍스트를 분리한다.

- [ ] **Step 4: 매핑 검증과 생성 결과를 커밋한다**

Run: `PYTHONPATH=src python3 tools/build_law_learning_map.py --laws data/laws --curriculum data/curriculum.json --exam-issues data/exam-issue-learning --output data/laws/learning.json`

Expected: `core_topics=116`, `core_laws=5`, `unresolved=0` 출력.

```bash
git add data/laws/learning.json tools/build_law_learning_map.py tests/test_law_learning_map.py
git commit -m "feat: map tax issues and precedents to law articles"
```

### Task 5: 조문 상세 모델·렌더러와 A안 템플릿 구현

**Files:**
- Create: `src/tax_study/law_reader.py`
- Create: `templates/law-reader.html`
- Create: `tests/test_law_reader.py`

- [ ] **Step 1: 승인된 섹션 순서를 고정하는 실패 테스트를 작성한다**

```python
def test_reader_uses_continuous_law_first_order(self):
    rendered = render_law_article(self.model, TEMPLATE)
    order = [
        rendered.index("법률 원문"),
        rendered.index("시행령 원문"),
        rendered.index("초보자용 해설"),
        rendered.index("과세쟁점·세무조사"),
        rendered.index("관련 기출"),
        rendered.index("중요 판례"),
    ]
    self.assertEqual(order, sorted(order))

def test_reader_contains_no_completion_control_and_escapes_authored_text(self):
    rendered = render_law_article(self.model, TEMPLATE)
    self.assertNotIn("학습 완료", rendered)
    self.assertIn("&lt;script&gt;", rendered)
```

- [ ] **Step 2: 조문 모델과 안전한 렌더링을 구현한다**

`build_law_article_reader_model`은 현행 버전, 연결 시행령 버전, 시험 당시 연혁, beginner explanation, issue cards, related exams, precedent cards를 정렬된 immutable projection으로 만든다. `render_law_article`은 HTML escape, 공식 URL allowlist, unique template token, raw HTML 차단을 적용한다. 원문과 학습용 해설은 텍스트 라벨을 가진 별도 `section`으로 렌더링한다.

- [ ] **Step 3: 모바일 템플릿을 구현한다**

템플릿은 `viewport-fit=cover`, 최소 17px 본문, `max-width: 68ch`, 긴 조문 자동 줄바꿈, 앵커 목차, 44px 조작 영역, reduced-motion, print CSS를 포함한다. 상단 목차는 가로 넘침 없이 줄바꿈되고 하단에 이전·목록·다음 링크를 둔다. 본문 콘텐츠에는 자바스크립트가 필요하지 않다.

- [ ] **Step 4: 렌더러 계약 테스트를 통과시키고 커밋한다**

Run: `PYTHONPATH=src:tests python3 -m unittest tests.test_law_reader -v`

Expected: 법률→시행령→해설→쟁점→기출→판례 순서, 공식 출처, escaping, no-completion 테스트 PASS.

```bash
git add src/tax_study/law_reader.py templates/law-reader.html tests/test_law_reader.py
git commit -m "feat: render law-first article readers"
```

### Task 6: 5개 법률 목록·조문 목록 화면 구현

**Files:**
- Create: `src/tax_study/law_hub.py`
- Create: `templates/law-index.html`
- Create: `tests/test_law_hub.py`

- [ ] **Step 1: 목록 모델 계약 테스트를 작성한다**

```python
def test_law_hub_has_exactly_five_cards_and_article_counts(self):
    model = build_law_hub_model(DATA_ROOT)
    self.assertEqual([row["law_code"] for row in model["laws"]], ["NTBA", "NCTA", "CTA", "ITA", "VAT"])
    self.assertTrue(all(row["article_count"] > 0 for row in model["laws"]))

def test_article_index_orders_by_article_number_and_exposes_core_status(self):
    model = build_law_index_model(DATA_ROOT, "NTBA")
    numbers = [row["sort_key"] for row in model["articles"]]
    self.assertEqual(numbers, sorted(numbers))
    self.assertIn(model["articles"][0]["content_status"], {"core", "source_only"})
```

- [ ] **Step 2: 법률 카드와 조문 행을 렌더링한다**

`/laws/`는 5개 카드와 법률별 설명만 보여 준다. `/laws/{law_code}/`는 모든 조문을 번호순으로 렌더링하며, 조문 번호·제목·키워드 검색, 핵심 개념·신고납부·조사증빙·불복구제·회계세무조정 필터를 정적 data attribute와 보조 inline script로 제공한다. 자바스크립트가 없어도 모든 조문 행과 링크가 보인다.

- [ ] **Step 3: 목록 테스트와 모바일 CSS 검사를 통과시킨다**

Run: `PYTHONPATH=src:tests python3 -m unittest tests.test_law_hub -v`

Expected: 5개 법률, 전체 조문 수, 핵심 상태, safe relative href, 모바일 viewport 테스트 PASS.

- [ ] **Step 4: 커밋한다**

```bash
git add src/tax_study/law_hub.py templates/law-index.html tests/test_law_hub.py
git commit -m "feat: add five-law and article indexes"
```

### Task 7: 정적 Pages 생성기와 기존 메뉴 통합

**Files:**
- Modify: `src/tax_study/pages.py`
- Modify: `src/tax_study/__init__.py`
- Modify: `templates/dashboard.html`
- Modify: `templates/curriculum.html`
- Modify: `templates/exams.html`
- Modify: `templates/precedents.html`
- Modify: `tests/test_pages.py`

- [ ] **Step 1: 새 경로와 기존 회귀를 고정하는 테스트를 추가한다**

```python
def test_expanded_site_contains_law_index_every_core_law_and_core_readers(self):
    docs = build_expanded_pages_site(self.state, self.project_root, self.repository, self.publish_date)
    self.assertTrue((docs / "laws/index.html").is_file())
    self.assertTrue((docs / "laws/NTBA/index.html").is_file())
    self.assertTrue((docs / "laws/NTBA/NTBA-16/index.html").is_file())
    self.assertIn('href="../laws/index.html"', (docs / "dashboard/index.html").read_text())

def test_expanded_site_rejects_unknown_law_or_article_output(self):
    with self.assertRaises(PagesPublishError):
        render_law_article_path("ETAX", "ETAX-1")
```

- [ ] **Step 2: Pages builder에 법령 정적 출력을 추가한다**

`pages.py`의 expanded build는 `laws/index.html`, 5개 `laws/{code}/index.html`, 모든 조문 상세 파일을 스테이징에 만든다. 기존 503개 산출물은 유지하고 법령 파일 수를 생성 모델에서 계산하여 manifest에 넣는다. staging 검증은 5개 법률 외 경로, raw JSON·Markdown 노출, 비공식 URL, broken local target을 거부한다. 기존 원자 교체·lock·backup 보존 동작은 변경하지 않는다.

- [ ] **Step 3: 전체 네비게이션에 법령 학습 링크를 추가한다**

기존 대시보드·교육과정·기출·최신판례 네비게이션에 `법령 학습` 링크를 추가하되, 기존 href와 공개 데이터 payload는 보존한다. 조문 상세에서는 `대시보드`, `법령 목록`, `기출`, `최신판례`를 relative anchor로 제공한다.

- [ ] **Step 4: 통합 테스트를 실행한다**

Run: `PYTHONPATH=src:tests python3 -m unittest tests.test_pages tests.test_dashboard tests.test_law_hub tests.test_law_reader -q`

Expected: 신규 경로와 기존 공개 페이지의 상대 링크·원자성·privacy·mobile 계약 PASS.

- [ ] **Step 5: 커밋한다**

```bash
git add src/tax_study/pages.py src/tax_study/__init__.py templates/dashboard.html templates/curriculum.html templates/exams.html templates/precedents.html tests/test_pages.py
git commit -m "feat: publish law-first pages alongside existing menus"
```

### Task 8: 콘텐츠 품질·접근성·회귀 검증 강화

**Files:**
- Create: `tests/test_law_quality.py`
- Modify: `tests/test_laws.py`
- Modify: `README.md`
- Modify: `DESIGN.md`

- [ ] **Step 1: 인수 기준을 직접 검사하는 테스트를 추가한다**

검사 대상은 정확히 5개 법률, 모든 조문 목록, 116개 core status, 원문 순서, 시행령 ID 존재성, 판례 공식 URL, “학습용 해설” 분리, 완료 버튼 부재, 390px viewport, `h1`·landmark·skip link, 44px 조작 영역, 자바스크립트 없는 본문이다.

- [ ] **Step 2: 법률·판례·기출 연결의 품질 검사를 실행한다**

Run: `PYTHONPATH=src:tests python3 -m unittest tests.test_laws tests.test_law_import tests.test_law_learning_map tests.test_law_quality tests.test_law_reader tests.test_law_hub -q`

Expected: 신규 법령 범위와 콘텐츠 계약 테스트가 모두 `OK`.

- [ ] **Step 3: README와 DESIGN에 새 메뉴 계약을 기록한다**

README에는 5개 법률 범위, `/laws/` 경로, 전체 조문·핵심 조문 단계적 공개, 현행·시험 당시 버전 원칙, 완료·로그인 없음, 공식 출처 정책을 추가한다. DESIGN에는 법률 목록 카드와 A안 조문 상세 위계를 추가한다.

- [ ] **Step 4: 커밋한다**

```bash
git add tests/test_law_quality.py tests/test_laws.py README.md DESIGN.md
git commit -m "test: enforce law-first learning acceptance criteria"
```

### Task 9: 정적 번들 생성·모바일 시각 검증·GitHub Pages 게시

**Files:**
- Modify: public repository `docs/` generated output only
- Preserve: `.tax-study-pages-publish.lock` (never stage, delete, or overwrite)

- [ ] **Step 1: 소스와 임시 Pages 저장소를 검증한다**

Run:

```bash
build_root=$(mktemp -d /tmp/tax-study-pages-build-XXXXXX)
cp -a /mnt/d/workspace/세법/tax-study-pages/. "$build_root"/
PYTHONPATH=src python3 -m tax_study.pages --state data/study.json --project-root . --repository "$build_root" --date 2026-09-01 --expanded
```

Expected: staging output contains `.nojekyll`, existing public pages, 5 law indexes, all article pages, and no raw source workspace files. The real repository is not touched during the build.

- [ ] **Step 2: generated output counts, order, links, and mobile CSS를 검사한다**

Run: `find "$build_root/docs" -type f | sort`; then run the law-specific quality tests against `"$build_root/docs"`.

Expected: `laws/index.html` has exactly five cards; every article row has a valid relative detail link; every core detail has all seven sections; all official URLs are HTTPS allowlisted; 390px capture has no horizontal overflow.

- [ ] **Step 3: real public docs를 동기화하고 좁은 경로만 stage한다**

```bash
cp -a "$build_root/docs/." /mnt/d/workspace/세법/tax-study-pages/docs/
git -C /mnt/d/workspace/세법/tax-study-pages add -- .tax-study-pages-public docs
git -C /mnt/d/workspace/세법/tax-study-pages diff --cached --name-only --no-renames
git -C /mnt/d/workspace/세법/tax-study-pages diff --cached --check
```

Expected: staged paths are `.tax-study-pages-public` or `docs/**` only. The lock file remains untracked and untouched.

- [ ] **Step 4: commit·push·remote SHA를 확인한다**

```bash
git -C /mnt/d/workspace/세법/tax-study-pages commit -m "Publish law-first tax learning pages"
git -C /mnt/d/workspace/세법/tax-study-pages push origin main
git -C /mnt/d/workspace/세법/tax-study-pages ls-remote origin refs/heads/main
```

Expected: remote `main` SHA equals the local published HEAD.

- [ ] **Step 5: 실제 모바일 화면을 캡처한다**

Use the installed headless shell at 390×844 for `/laws/`, one article detail, and one source-only article. Confirm the law card list, anchor order, wrapped law text, and bottom navigation. Save captures under `/tmp/law-first-*.png` and inspect them visually.

### Task 10: 전체 검증과 인수 보고

**Files:**
- Test: all `tests/test_*.py`
- Verify: public URLs and generated manifest

- [ ] **Step 1: 변경 범위 테스트를 다시 실행한다**

Run: `PYTHONPATH=src:tests PYTHONDONTWRITEBYTECODE=1 python3 -B -m unittest tests.test_laws tests.test_law_import tests.test_law_learning_map tests.test_law_quality tests.test_law_reader tests.test_law_hub tests.test_pages tests.test_dashboard -q`

Expected: all selected tests pass; any unrelated long-running audit-anchor test is reported separately rather than hidden.

- [ ] **Step 2: 전체 테스트를 bounded command로 실행한다**

Run: `PYTHONPATH=src:tests PYTHONDONTWRITEBYTECODE=1 timeout 600s python3 -u -B -m unittest discover -s tests -p 'test_*.py' -q`

Expected: exit 0. If the command reaches the timeout, record the exact timed-out module and retain the successful scoped-test result; do not claim the entire suite passed.

- [ ] **Step 3: 공개 URL을 HTTP와 본문 marker로 확인한다**

```bash
curl -L --max-time 30 -sS -o /tmp/law-index-public.html -w '%{http_code} %{url_effective}\n' https://wttear.github.io/tax-study-reader/laws/
curl -L --max-time 30 -sS -o /tmp/law-article-public.html -w '%{http_code} %{url_effective}\n' https://wttear.github.io/tax-study-reader/laws/NTBA/NTBA-16/
rg -q '법령 중심 학습|법률 원문|시행령 원문|초보자용 해설|과세쟁점|중요 판례' /tmp/law-article-public.html
```

Expected: both URLs return HTTP 200 and the article contains the required markers.

- [ ] **Step 4: 인수 보고에 경로·범위·검증 결과를 기록한다**

Report the five-law scope, total article rows, number of core readers, public URLs, remote commit SHA, mobile capture paths, scoped test count, and any full-suite limitation. Do not describe an unverified law version or case as current.
