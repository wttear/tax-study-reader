# Exam Question Learning Reader Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publish 362 mobile-first learning pages for the licensed 2014–2026 Korean tax-accountant second-stage tax-law subquestions, each showing the source question, a beginner reading guide, a learning model answer with paragraph explanations, and reasoned links to related issues.

**Architecture:** Keep the existing 429-record archive as the complete index and add a separate validated detail-content layer for the 362 licensed records. Build immutable public reader models from the existing exam model plus three reviewed content shards, render one static HTML page per eligible question, and expose those pages from both the archive and issue hub while preserving the 67 link-only records from 2011–2013.

**Tech Stack:** Python 3 standard library, `unittest`, existing static HTML/CSS/JavaScript templates, one-time `hwp5txt`/`pdftotext` source extraction, GitHub Pages.

**Repository constraint:** `tax-study` is a writable source tree without Git metadata in this workspace. Do not initialize or publish a new source repository without separate user authorization. Tasks 1–9 therefore end with hash/test checkpoints instead of source commits; Task 10 commits only the generated, already-versioned `tax-study-pages` public tree.

---

## File map

- `tax-study/src/tax_study/exam_question_sources.py`: bounded one-time downloader/extractor for official Q-Net HWP, PDF, and ZIP attachments.
- `tax-study/src/tax_study/exam_question_data.py`: loads and validates reviewed question-detail shards against the canonical 429-record exam model and 70-topic catalog.
- `tax-study/src/tax_study/exam_question_learning.py`: creates the 362 immutable public detail models, stable URLs, and previous/list/next navigation.
- `tax-study/src/tax_study/exam_question_reader.py`: safely renders one validated detail model.
- `tax-study/templates/exam-question.html`: mobile-first B-layout template with all learning sections visible.
- `tax-study/data/exam-question-learning.schema.json`: exact review-data contract.
- `tax-study/data/exam-question-learning/2014-2016.json`: 77 reviewed question details.
- `tax-study/data/exam-question-learning/2017-2021.json`: 135 reviewed question details.
- `tax-study/data/exam-question-learning/2022-2026.json`: 150 reviewed question details.
- `tax-study/tests/test_exam_question_sources.py`: source download, digest, archive-safety, and extraction tests.
- `tax-study/tests/test_exam_question_data.py`: schema, coverage, legal scope, and production completeness tests.
- `tax-study/tests/test_exam_question_learning.py`: canonical model, navigation, related-issue, and immutability tests.
- `tax-study/tests/test_exam_question_reader.py`: HTML order, attribution, accessibility, mobile, and safety tests.
- `tax-study/src/tax_study/exam_archive.py`: adds detail-page entry links only to 2014–2026 rows.
- `tax-study/src/tax_study/exam_learning.py`: projects optional detail links into the question map used by the issue hub.
- `tax-study/templates/exam-archive.html`: changes the archive call to action from download-only to learn-first for eligible rows.
- `tax-study/templates/exams.html`: links eligible direct-exam cards to their detail pages.
- `tax-study/src/tax_study/pages.py`: renders, validates, and atomically publishes 362 additional static files.
- `tax-study/tests/test_exam_archive.py`, `test_exam_template.py`, `test_pages.py`: integration and regression coverage.
- `tax-study/sources/exams/README.md`: documents 2014–2026 detail coverage and the retained 2011–2013 link-only boundary.

## Data contract used by every content shard

Each shard has exactly this top-level shape:

```json
{
  "schema_version": 1,
  "years": [2014, 2015, 2016],
  "records": []
}
```

Each `records` entry has this exact shape:

```json
{
  "exam_question_id": "EXAM-2014-P1-Q1-S1",
  "shared_context_blocks": [
    {"type": "paragraph", "text": "공통 사실관계 전문"}
  ],
  "prompt_blocks": [
    {"type": "paragraph", "text": "세부문항 원문"}
  ],
  "beginner_reading": {
    "what_is_asked": "이 문항이 요구하는 답안 결과물",
    "key_facts": ["법률요건과 연결할 사실"],
    "prerequisites": ["먼저 이해할 개념"],
    "traps": ["혼동하기 쉬운 지점"]
  },
  "answer_at_a_glance": ["쟁점", "규범", "포섭", "결론"],
  "model_answer": [
    {
      "paragraph_id": "p1",
      "role": "issue",
      "text": "시험에 작성할 수 있는 첫 답안 문단"
    }
  ],
  "paragraph_explanations": [
    {
      "paragraph_id": "p1",
      "why_it_matters": "이 문단이 필요한 이유",
      "plain_language": "초보자를 위한 일상어 풀이"
    }
  ],
  "precedent_connections": [
    {
      "case_number": "2010두19713",
      "holding": "이 문항에 필요한 판시사항",
      "reason": "문항 사실과 판례가 연결되는 이유",
      "official_url": "https://www.law.go.kr/LSW/precInfoP.do?precSeq=165914"
    }
  ],
  "related_issues": [
    {
      "topic_id": "NTBA-AUDIT-SELECTION-001",
      "relation": "direct",
      "reason": "이 쟁점을 먼저 또는 이어서 학습해야 하는 이유"
    }
  ],
  "common_mistakes": ["답안에서 자주 빠지는 내용"],
  "variant_question": "사실 하나를 바꾼 연습 문제"
}
```

`shared_context_blocks` may be empty only when the official source has no separate shared context. `prompt_blocks` is never empty. A block is either a paragraph or the following table shape:

```json
{
  "type": "table",
  "caption": "문제에 표시된 표 제목",
  "headers": ["열 1", "열 2"],
  "rows": [["값 1", "값 2"]]
}
```

Allowed model-answer roles are `issue`, `rule`, `precedent`, `application`, `calculation`, `accounting`, `tax_adjustment`, and `conclusion`. Allowed related-issue relations are `direct`, `related`, and `extension`: `direct` matches a base primary topic, `related` matches a base related topic, and `extension` adds a catalog topic with a question-specific reason without changing the archive's primary classification. Every question has at least one related issue. Every `paragraph_id` is unique within one question and appears exactly once in `paragraph_explanations`.

### Task 1: Lock the licensed coverage and detail-data validator

**Files:**
- Create: `tax-study/data/exam-question-learning.schema.json`
- Create: `tax-study/src/tax_study/exam_question_data.py`
- Create: `tax-study/tests/test_exam_question_data.py`

- [ ] **Step 1: Write the failing scope and schema tests**

```python
PROJECT = Path(__file__).resolve().parents[1]
DETAIL_DIR = PROJECT / "data/exam-question-learning"
CATALOG = load_catalog(PROJECT / "data/curriculum.json")
_, EXAM_RECORDS = load_exam_data(
    PROJECT / "data/exam-source-manifest.json",
    PROJECT / "data/exam-questions.json",
)


class ExamQuestionDetailScopeTests(unittest.TestCase):
    def test_valid_2014_record_is_accepted_in_fixture_mode(self):
        records = validate_exam_question_details(
            [synthetic_shard("EXAM-2014-P1-Q1-S1", 2014)],
            EXAM_RECORDS,
            CATALOG,
            require_complete=False,
        )
        self.assertEqual(records[0]["exam_question_id"], "EXAM-2014-P1-Q1-S1")

    def test_every_model_answer_paragraph_has_one_beginner_explanation(self):
        shard = synthetic_shard("EXAM-2014-P1-Q1-S1", 2014)
        shard["records"][0]["paragraph_explanations"][0]["paragraph_id"] = "wrong"
        with self.assertRaisesRegex(ExamQuestionDataError, "paragraph"):
            validate_exam_question_details(
                [shard], EXAM_RECORDS, CATALOG, require_complete=False
            )

    def test_2013_record_is_rejected_even_if_it_claims_publication_permission(self):
        fixture = [synthetic_shard("EXAM-2013-P1-Q1-S1", 2013)]
        with self.assertRaisesRegex(ExamQuestionDataError, "2014.*2026"):
            validate_exam_question_details(
                fixture, EXAM_RECORDS, CATALOG, require_complete=False
            )


def synthetic_shard(question_id, year):
    return {
        "schema_version": 1,
        "years": [year],
        "records": [{
            "exam_question_id": question_id,
            "shared_context_blocks": [],
            "prompt_blocks": [{"type": "paragraph", "text": "공식 문항 원문"}],
            "beginner_reading": {
                "what_is_asked": "납세의무 판단을 서술한다.",
                "key_facts": ["처분일과 납세자"],
                "prerequisites": ["과세요건"],
                "traps": ["현행법과 시험 당시 법 혼용"],
            },
            "answer_at_a_glance": ["쟁점", "규범", "포섭", "결론"],
            "model_answer": [{
                "paragraph_id": "p1", "role": "issue", "text": "쟁점을 특정한다."
            }],
            "paragraph_explanations": [{
                "paragraph_id": "p1",
                "why_it_matters": "검토 순서를 정한다.",
                "plain_language": "무엇을 판단할 문제인지 먼저 밝힌다.",
            }],
            "precedent_connections": [],
            "related_issues": [{
                "topic_id": "NTBA-SUBSTANCE-001",
                "relation": "extension",
                "reason": "사실의 명칭보다 실제 귀속을 함께 판단하기 때문이다.",
            }],
            "common_mistakes": ["시험 당시 법령 누락"],
            "variant_question": "처분일만 달라지면 적용법을 다시 판단한다.",
        }],
    }
```

- [ ] **Step 2: Run the tests and verify RED**

Run:

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest test_exam_question_data -v
```

Expected: import failure for `tax_study.exam_question_data`.

- [ ] **Step 3: Implement the exact loader and fail-closed validator**

```python
DETAIL_YEARS = frozenset(range(2014, 2027))
DETAIL_COUNTS = {(2014, 2016): 77, (2017, 2021): 135, (2022, 2026): 150}
DETAIL_TOTAL = 362
MODEL_ANSWER_ROLES = frozenset({
    "issue", "rule", "precedent", "application", "calculation",
    "accounting", "tax_adjustment", "conclusion",
})


def load_exam_question_details(directory, exam_records, catalog):
    directory = Path(directory)
    shards = [
        json.loads(path.read_text(encoding="utf-8"), object_pairs_hook=_unique_object)
        for path in sorted(directory.glob("*.json"))
    ]
    return validate_exam_question_details(
        shards, exam_records, catalog, require_complete=True
    )
```

Implement `validate_exam_question_details(shards, exam_records, catalog, *, require_complete=True)`. The validator must enforce the exact fields documented above; reject booleans as integers; reject duplicate JSON keys, record IDs, paragraph IDs, topic IDs, and cases; reject control characters and local paths; require 3–5 `answer_at_a_glance` entries; require at least one related issue; require every related topic to exist in the catalog; enforce `direct` against base primary topics, `related` against base related topics, and `extension` against other catalog topics; and compare the 362 IDs to the canonical set derived from `exam_records` where `2014 <= exam_year <= 2026`. `require_complete=False` is permitted only for inputs smaller than 362 and never permits a year outside 2014–2026.

- [ ] **Step 4: Add the JSON schema mirroring the Python contract**

The schema must set `additionalProperties: false` at every object level, use the role enumeration above, require HTTPS case-source URLs on approved `law.go.kr` or `scourt.go.kr` hosts, and allow only `paragraph` and `table` prompt blocks. Validate the schema itself with `json.loads` in the test suite; Python validation remains the runtime authority.

- [ ] **Step 5: Run the focused tests and verify GREEN**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest test_exam_question_data -v
```

Expected: all synthetic scope and contract tests pass without reading production detail shards.

- [ ] **Step 6: Record the verified source-tree checkpoint**

```bash
sha256sum tax-study/data/exam-question-learning.schema.json tax-study/src/tax_study/exam_question_data.py tax-study/tests/test_exam_question_data.py
```

Expected: three SHA-256 lines are retained in the execution log after the focused test passes.

### Task 2: Build the bounded official-source extraction tool

**Files:**
- Create: `tax-study/src/tax_study/exam_question_sources.py`
- Create: `tax-study/tests/test_exam_question_sources.py`
- Modify: `tax-study/sources/exams/README.md`

- [ ] **Step 1: Write failing source-integrity tests**

```python
PROJECT = Path(__file__).resolve().parents[1]


def load_manifest():
    return json.loads(
        (PROJECT / "data/exam-source-manifest.json").read_text(encoding="utf-8")
    )


def zip_bytes(members):
    output = io.BytesIO()
    with ZipFile(output, "w") as archive:
        for name, payload in members.items():
            archive.writestr(name, payload)
    return output.getvalue()


class ExamQuestionSourceTests(unittest.TestCase):
    def setUp(self):
        self.temporary = tempfile.TemporaryDirectory()
        self.addCleanup(self.temporary.cleanup)
        self.work = Path(self.temporary.name)

    def test_selects_only_kogl_type_one_sources_from_2014_forward(self):
        selected = select_detail_sources(load_manifest())
        self.assertEqual(len(selected), 26)
        self.assertEqual({row["exam_year"] for row in selected}, set(range(2014, 2027)))
        self.assertTrue(all(row["license_name"] == "공공누리 제1유형" for row in selected))

    def test_download_rejects_digest_mismatch_before_extraction(self):
        with self.assertRaisesRegex(ExamQuestionSourceError, "digest"):
            verify_download(b"changed", "0" * 64)

    def test_zip_members_cannot_escape_the_work_directory(self):
        payload = zip_bytes({"../outside.hwp": b"unsafe"})
        with self.assertRaisesRegex(ExamQuestionSourceError, "archive member"):
            extract_attachment(payload, "questions.zip", self.work)
```

- [ ] **Step 2: Run the tests and verify RED**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest test_exam_question_sources -v
```

Expected: import failure for `tax_study.exam_question_sources`.

- [ ] **Step 3: Implement manifest selection, download verification, and safe extraction**

Expose these functions:

```python
MAX_ATTACHMENT_BYTES = 120 * 1024 * 1024


def select_detail_sources(manifest):
    selected = [
        copy.deepcopy(row)
        for row in manifest["sources"]
        if 2014 <= row["exam_year"] <= 2026
        and row["status"] == "available"
        and row["license_name"] == "공공누리 제1유형"
    ]
    expected = {
        (year, part)
        for year in range(2014, 2027)
        for part in ("세법학 1부", "세법학 2부")
    }
    actual = {(row["exam_year"], row["subject_part"]) for row in selected}
    if actual != expected or len(selected) != 26:
        raise ExamQuestionSourceError("licensed source coverage must be 26 rows")
    return selected


def verify_download(payload: bytes, expected_sha256: str) -> None:
    if len(payload) > MAX_ATTACHMENT_BYTES:
        raise ExamQuestionSourceError("attachment exceeds size limit")
    if hashlib.sha256(payload).hexdigest() != expected_sha256:
        raise ExamQuestionSourceError("attachment digest mismatch")


def download_verified_source(source, work_dir):
    url = source["official_attachment_url"]
    parsed = urlsplit(url)
    if parsed.scheme != "https" or parsed.hostname != "www.q-net.or.kr" or parsed.path != "/cst003.do":
        raise ExamQuestionSourceError("attachment URL is not approved Q-Net HTTPS")
    filename_values = parse_qs(parsed.query).get("fileName", [])
    if len(filename_values) != 1:
        raise ExamQuestionSourceError("attachment URL has no unique filename")
    filename = Path(unquote(filename_values[0])).name
    request = Request(url, headers={"User-Agent": "tax-study-source-review/1"})
    with urlopen(request, timeout=30) as response:
        payload = response.read(MAX_ATTACHMENT_BYTES + 1)
    verify_download(payload, source["source_sha256"])
    target = Path(work_dir) / f'{source["exam_year"]}-{source["subject_part"]}-{filename}'
    target.write_bytes(payload)
    return target


def extract_attachment(payload: bytes, filename: str, work_dir: Path) -> list[Path]:
    work_dir = Path(work_dir)
    suffix = Path(filename).suffix.lower()
    if suffix == ".zip":
        outputs = []
        total = 0
        with ZipFile(BytesIO(payload)) as archive:
            infos = archive.infolist()
            if len(infos) > 32:
                raise ExamQuestionSourceError("archive member count exceeds limit")
            for info in infos:
                if info.is_dir():
                    continue
                member = PurePosixPath(info.filename)
                if member.is_absolute() or ".." in member.parts or "\\" in info.filename:
                    raise ExamQuestionSourceError("unsafe archive member")
                if stat.S_ISLNK(info.external_attr >> 16):
                    raise ExamQuestionSourceError("archive member must not be a symlink")
                total += info.file_size
                if info.file_size > 30 * 1024 * 1024 or total > MAX_ATTACHMENT_BYTES:
                    raise ExamQuestionSourceError("archive expanded size exceeds limit")
                if info.flag_bits & 1:
                    raise ExamQuestionSourceError("encrypted archive member")
                outputs.extend(extract_attachment(archive.read(info), member.name, work_dir))
        return outputs
    source = work_dir / Path(filename).name
    source.write_bytes(payload)
    output = source.with_suffix(source.suffix + ".txt")
    if suffix == ".hwp":
        completed = subprocess.run(
            ["hwp5txt", str(source)], check=True, capture_output=True, text=True
        )
        output.write_text(completed.stdout, encoding="utf-8", newline="\n")
    elif suffix == ".pdf":
        subprocess.run(["pdftotext", "-layout", str(source), str(output)], check=True)
    else:
        raise ExamQuestionSourceError("attachment type must be HWP, PDF, or ZIP")
    return [output]


def normalize_extracted_text(path: Path) -> str:
    value = Path(path).read_text(encoding="utf-8", errors="strict")
    value = unicodedata.normalize("NFC", value.replace("\r\n", "\n").replace("\r", "\n"))
    return "\n".join(line.rstrip() for line in value.split("\n")).strip() + "\n"
```

Use `urllib.request` only for `https://www.q-net.or.kr/cst003.do` attachment URLs. Store downloads under a caller-supplied temporary directory, never in the repository. Open ZIP members one by one after rejecting absolute names, `..`, backslashes, symlinks, encryption, more than 32 members, an individual size above 30 MiB, or a total uncompressed size above 120 MiB. Invoke `hwp5txt` for HWP and `pdftotext -layout` for PDF through argument arrays without a shell. Normalize only line endings, trailing whitespace, and Unicode NFC automatically; remove repeated headers or page artifacts only during human transcription review so question wording is not changed by a heuristic.

- [ ] **Step 4: Add the documented extraction command**

Add this exact workflow to `sources/exams/README.md`:

```bash
cd tax-study
UV_CACHE_DIR=/tmp/tax-study-uv-cache uvx --from pyhwp hwp5txt --help
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src python3 -B -m tax_study.exam_question_sources \
  --manifest data/exam-source-manifest.json \
  --work-dir /tmp/tax-study-exam-sources
```

The CLI writes normalized text and a SHA-256 review report only to `--work-dir`; it never updates production JSON automatically.
In the same README edit, correct the stale `428개` structural count to the canonical `429개`, state that 362 records from 2014–2026 receive detail pages, and state that the 67 records from 2011–2013 remain link-only.

- [ ] **Step 5: Run focused tests and one official dry run**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest test_exam_question_sources -v
```

Expected: all source tests pass. Then run the documented CLI for one 2014 source and confirm its downloaded digest equals the manifest digest before content review.

- [ ] **Step 6: Record the verified source-tree checkpoint**

```bash
sha256sum tax-study/src/tax_study/exam_question_sources.py tax-study/tests/test_exam_question_sources.py tax-study/sources/exams/README.md
```

Expected: three SHA-256 lines are retained in the execution log after extraction tests pass.

### Task 3: Author and verify the 2014–2016 content shard

**Files:**
- Create: `tax-study/data/exam-question-learning/2014-2016.json`
- Modify: `tax-study/tests/test_exam_question_data.py`

- [ ] **Step 1: Add the failing shard-count test**

```python
def read_shard(name):
    return json.loads((DETAIL_DIR / name).read_text(encoding="utf-8"))


def eligible_ids_for_years(start_year, end_year):
    return {
        row["exam_question_id"]
        for row in EXAM_RECORDS
        if start_year <= row["exam_year"] <= end_year
    }


class ExamQuestion2014To2016ProductionTests(unittest.TestCase):
    def test_2014_2016_shard_has_exact_reviewed_coverage(self):
        shard = read_shard("2014-2016.json")
        self.assertEqual(shard["years"], [2014, 2015, 2016])
        self.assertEqual(len(shard["records"]), 77)
        self.assertEqual(
            {row["exam_question_id"] for row in shard["records"]},
            eligible_ids_for_years(2014, 2016),
        )
```

- [ ] **Step 2: Run the test and verify RED**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest \
  test_exam_question_data.ExamQuestion2014To2016ProductionTests.test_2014_2016_shard_has_exact_reviewed_coverage -v
```

Expected: missing `2014-2016.json`.

- [ ] **Step 3: Transcribe and author all 77 records**

Use only the digest-verified Q-Net source text from Task 2 for `shared_context_blocks` and `prompt_blocks`. Reuse the existing `learning_sections.answer_elements`, `law_at_exam`, `current_law_difference`, `common_omissions`, and topic mappings as source material, but rewrite them into the exact detail contract: a complete exam-ready `model_answer`, one beginner explanation per answer paragraph, and a nonblank reason for every related issue. Preserve official question numbering and table values. Do not add a court case unless its official case number and URL are verified.

- [ ] **Step 4: Run shard validation and legal spot checks**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest \
  test_exam_question_data.ExamQuestion2014To2016ProductionTests.test_2014_2016_shard_has_exact_reviewed_coverage \
  test_exams.ExamProductionDataTests -v
```

Expected: 77 IDs, no schema errors, and existing exam-law validation remains green.

- [ ] **Step 5: Record the verified source-tree checkpoint**

```bash
sha256sum tax-study/data/exam-question-learning/2014-2016.json tax-study/tests/test_exam_question_data.py
```

Expected: two SHA-256 lines are retained with the 77-record validation output.

### Task 4: Author and verify the 2017–2021 content shard

**Files:**
- Create: `tax-study/data/exam-question-learning/2017-2021.json`
- Modify: `tax-study/tests/test_exam_question_data.py`

- [ ] **Step 1: Add the failing shard-count test**

```python
class ExamQuestion2017To2021ProductionTests(unittest.TestCase):
    def test_2017_2021_shard_has_exact_reviewed_coverage(self):
        shard = read_shard("2017-2021.json")
        self.assertEqual(shard["years"], [2017, 2018, 2019, 2020, 2021])
        self.assertEqual(len(shard["records"]), 135)
        self.assertEqual(
            {row["exam_question_id"] for row in shard["records"]},
            eligible_ids_for_years(2017, 2021),
        )
```

- [ ] **Step 2: Run the test and verify RED**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest \
  test_exam_question_data.ExamQuestion2017To2021ProductionTests.test_2017_2021_shard_has_exact_reviewed_coverage -v
```

Expected: missing `2017-2021.json`.

- [ ] **Step 3: Transcribe and author all 135 records**

Apply the same reviewed contract as Task 3. For 2019–2021 attachments that contain multiple subjects, include only the shared facts and prompt blocks belonging to 세법학 1부 or 세법학 2부. Keep 시험 당시 법령 and 현행 차이 separate. Where a calculation, accounting entry, or tax adjustment is not part of the question, omit that model-answer role instead of inventing one.

- [ ] **Step 4: Run shard and audit-procedure legal checks**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest \
  test_exam_question_data.ExamQuestionProductionDataTests.test_2017_2021_shard_has_exact_reviewed_coverage \
  test_exam_ntba_legal_accuracy -v
```

Expected: 135 IDs and all national-tax-basic-act accuracy checks pass.

- [ ] **Step 5: Record the verified source-tree checkpoint**

```bash
sha256sum tax-study/data/exam-question-learning/2017-2021.json tax-study/tests/test_exam_question_data.py
```

Expected: two SHA-256 lines are retained with the 135-record validation output.

### Task 5: Author and verify the 2022–2026 content shard

**Files:**
- Create: `tax-study/data/exam-question-learning/2022-2026.json`
- Modify: `tax-study/tests/test_exam_question_data.py`

- [ ] **Step 1: Add the failing shard-count and total-coverage tests**

```python
class ExamQuestion2022To2026ProductionTests(unittest.TestCase):
    def test_2022_2026_shard_has_exact_reviewed_coverage(self):
        shard = read_shard("2022-2026.json")
        self.assertEqual(shard["years"], [2022, 2023, 2024, 2025, 2026])
        self.assertEqual(len(shard["records"]), 150)
        self.assertEqual(
            {row["exam_question_id"] for row in shard["records"]},
            eligible_ids_for_years(2022, 2026),
        )


class ExamQuestionAllProductionTests(unittest.TestCase):
    def test_all_three_shards_cover_exactly_362_records(self):
        records = load_exam_question_details(DETAIL_DIR, EXAM_RECORDS, CATALOG)
        self.assertEqual(len(records), 362)
```

- [ ] **Step 2: Run the tests and verify RED**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest \
  test_exam_question_data.ExamQuestion2022To2026ProductionTests.test_2022_2026_shard_has_exact_reviewed_coverage \
  test_exam_question_data.ExamQuestionAllProductionTests.test_all_three_shards_cover_exactly_362_records -v
```

Expected: missing `2022-2026.json` and incomplete production coverage.

- [ ] **Step 3: Transcribe and author all 150 records**

Apply the same reviewed contract as Tasks 3 and 4. Preserve tables and calculation inputs from the ZIP-contained originals. For the 2026 questions, use the verified exam-date law already present in `exam-law-source-manifest.json`; do not replace it with the 2026-08-30 current-law version. Explicitly explain calculation inputs, intermediate amounts, accounting treatment, and tax adjustment when those roles occur.

- [ ] **Step 4: Run total validation and VAT legal checks**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest \
  test_exam_question_data test_exam_vat_legal_accuracy -v
```

Expected: all 362 records validate and VAT exam-date/current-law branches remain correct.

- [ ] **Step 5: Record the verified source-tree checkpoint**

```bash
sha256sum tax-study/data/exam-question-learning/2022-2026.json tax-study/tests/test_exam_question_data.py
```

Expected: two SHA-256 lines are retained with the 150-record and 362-total validation output.

### Task 6: Build immutable question-reader models and navigation

**Files:**
- Create: `tax-study/src/tax_study/exam_question_learning.py`
- Create: `tax-study/tests/test_exam_question_learning.py`

- [ ] **Step 1: Write failing model tests**

```python
class ExamQuestionLearningModelTests(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        inputs = canonical_model_inputs()
        cls.exam_model = inputs["exam_model"]
        cls.details = load_exam_question_details(
            PROJECT / "data/exam-question-learning",
            cls.exam_model["records"],
            inputs["catalog"],
        )
        cls.material_hrefs_by_path = {
            row["material_path"]: f'../../../lesson-reader/{Path(row["reader_path"]).name}'
            for row in inputs["content_manifest"]["materials"]
        }
        cls.models = build_exam_question_reader_models(
            cls.exam_model,
            cls.details,
            inputs["catalog"],
            cls.material_hrefs_by_path,
        )

    def test_builds_362_stable_detail_paths_and_excludes_2011_2013(self):
        models = self.models
        self.assertEqual(len(models), 362)
        self.assertEqual(models[0]["detail_path"], "exams/questions/EXAM-2014-P1-Q1-S1/index.html")
        self.assertTrue(models[-1]["detail_path"].startswith("exams/questions/EXAM-2026-"))
        self.assertFalse(any("EXAM-2013" in row["detail_path"] for row in models))

    def test_previous_and_next_cover_each_question_once(self):
        models = self.models
        self.assertIsNone(models[0]["navigation"]["previous_href"])
        self.assertIsNone(models[-1]["navigation"]["next_href"])
        self.assertEqual(models[1]["navigation"]["previous_href"], "../EXAM-2014-P1-Q1-S1/")

    def test_builder_does_not_mutate_inputs(self):
        inputs = canonical_model_inputs()
        details = copy.deepcopy(self.details)
        hrefs = copy.deepcopy(self.material_hrefs_by_path)
        before = copy.deepcopy((inputs["exam_model"], details, inputs["catalog"], hrefs))
        build_exam_question_reader_models(
            inputs["exam_model"], details, inputs["catalog"], hrefs
        )
        self.assertEqual((inputs["exam_model"], details, inputs["catalog"], hrefs), before)
```

- [ ] **Step 2: Run the tests and verify RED**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest test_exam_question_learning -v
```

Expected: import failure for `tax_study.exam_question_learning`.

- [ ] **Step 3: Implement model projection**

```python
def build_exam_question_reader_models(exam_model, details, catalog, material_hrefs_by_path):
    questions = exam_model["records"]
    base_by_id = {row["exam_question_id"]: row for row in questions}
    ordered = sorted(
        details,
        key=lambda row: questions.index(base_by_id[row["exam_question_id"]]),
    )
    return [
        _project_reader_model(
            index, ordered, base_by_id, catalog, material_hrefs_by_path
        )
        for index in range(len(ordered))
    ]
```

Use a precomputed canonical order index instead of repeated `list.index` in the final implementation. Merge official source/license fields and existing `law_at_exam`/`current_law_difference` into the public model. Resolve each related issue to a `../../../lesson-reader/{reader-file}.html` href and preserve the authored reason. Produce a separate `detail_href_by_id` map with `./questions/{question_id}/` for the exam hub and `../questions/{question_id}/` for the archive.

- [ ] **Step 4: Run tests and verify GREEN**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest test_exam_question_learning -v
```

Expected: all model, ordering, immutability, and link tests pass.

- [ ] **Step 5: Record the verified source-tree checkpoint**

```bash
sha256sum tax-study/src/tax_study/exam_question_learning.py tax-study/tests/test_exam_question_learning.py
```

Expected: two SHA-256 lines are retained after model tests pass.

### Task 7: Render the approved B-layout detail page

**Files:**
- Create: `tax-study/src/tax_study/exam_question_reader.py`
- Create: `tax-study/templates/exam-question.html`
- Create: `tax-study/tests/test_exam_question_reader.py`

- [ ] **Step 1: Write failing renderer tests**

```python
PROJECT = Path(__file__).resolve().parents[1]
TEMPLATE = PROJECT / "templates/exam-question.html"


def production_reader_model():
    inputs = canonical_model_inputs()
    details = load_exam_question_details(
        PROJECT / "data/exam-question-learning",
        inputs["exam_model"]["records"],
        inputs["catalog"],
    )
    hrefs = {
        row["material_path"]: f'../../../lesson-reader/{Path(row["reader_path"]).name}'
        for row in inputs["content_manifest"]["materials"]
    }
    return build_exam_question_reader_models(
        inputs["exam_model"], details, inputs["catalog"], hrefs
    )[0]


class ExamQuestionReaderTests(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        cls.model = production_reader_model()

    def test_sections_follow_the_approved_b_order(self):
        rendered = render_exam_question_reader(self.model, TEMPLATE)
        headings = [
            "기출문제 원문",
            "초보자용 문제 읽기",
            "한눈에 보는 정답",
            "완성형 학습용 모범답안",
            "초보자용 답안 해설",
            "관련 쟁점 연결",
        ]
        offsets = [rendered.index(text) for text in headings]
        self.assertEqual(offsets, sorted(offsets))

    def test_answer_is_visible_without_details_tabs_or_javascript(self):
        rendered = render_exam_question_reader(self.model, TEMPLATE)
        self.assertIn(self.model["model_answer"][0]["text"], rendered)
        self.assertNotIn("<details", rendered)
        self.assertNotIn('role="tab"', rendered)

    def test_source_and_nonofficial_answer_labels_are_visible(self):
        rendered = render_exam_question_reader(self.model, TEMPLATE)
        self.assertIn("공공누리 제1유형", rendered)
        self.assertIn("한국산업인력공단 Q-Net", rendered)
        self.assertIn("학습용 모범답안이며 공식답안이 아님", rendered)
```

- [ ] **Step 2: Run the tests and verify RED**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest test_exam_question_reader -v
```

Expected: import failure for `tax_study.exam_question_reader`.

- [ ] **Step 3: Implement the fail-closed renderer**

Expose:

```python
def render_exam_question_reader(model: dict, template_path) -> str:
    template = _read_strict_bounded_template(template_path)
    body = _render_reader_body(_validate_public_model(model))
    return _replace_once(template, "__EXAM_QUESTION_BODY__", body)
```

Reuse the existing bounded-template and canonical escaping patterns from `exam_archive.py`. Reject unknown model fields, unsafe URLs, control characters, malformed blocks, and leftover template tokens. Render table blocks as `<table>` with `<caption>`, `<thead>`, and `<tbody>`. Pair each answer paragraph with its beginner explanation by `paragraph_id` without hiding either section.

- [ ] **Step 4: Build the mobile template**

The template must use one content column, visible numbered section labels, 44px targets, `viewport-fit=cover`, `env(safe-area-inset-bottom)`, reduced-motion handling, print rules, and a bottom navigation containing only real anchor links. It must include direct links to dashboard, issue hub, archive, and precedents. Do not add completion controls, local storage, analytics, external scripts, or JavaScript-required content.

- [ ] **Step 5: Run renderer and template tests**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest test_exam_question_reader -v
```

Expected: all order, attribution, accessibility, safety, and no-JavaScript tests pass.

- [ ] **Step 6: Record the verified source-tree checkpoint**

```bash
sha256sum tax-study/src/tax_study/exam_question_reader.py tax-study/templates/exam-question.html tax-study/tests/test_exam_question_reader.py
```

Expected: three SHA-256 lines are retained after renderer tests pass.

### Task 8: Link the archive and issue hub to eligible detail pages

**Files:**
- Modify: `tax-study/src/tax_study/exam_learning.py`
- Modify: `tax-study/src/tax_study/exam_archive.py`
- Modify: `tax-study/templates/exam-archive.html`
- Modify: `tax-study/templates/exams.html`
- Modify: `tax-study/tests/test_exam_learning.py`
- Modify: `tax-study/tests/test_exam_archive.py`
- Modify: `tax-study/tests/test_exam_template.py`

- [ ] **Step 1: Write failing link-routing tests**

```python
def test_archive_has_362_learning_links_and_retains_429_official_links(self):
    rendered = render_fixture_archive()
    self.assertEqual(rendered.count('class="question-learning-link"'), 362)
    self.assertEqual(rendered.count('class="official-source-link"'), 429)
    marker = 'id="EXAM-2013-P1-Q1-S1"'
    start = rendered.index(marker)
    old = rendered[start:rendered.index("</li>", start)]
    self.assertNotIn("question-learning-link", old)
    self.assertIn("official-source-link", old)

def test_issue_hub_links_eligible_direct_questions_to_detail_pages(self):
    inputs = canonical_model_inputs()
    inputs["question_detail_hrefs"] = {
        row["exam_question_id"]: f'./questions/{row["exam_question_id"]}/'
        for row in inputs["exam_model"]["records"]
        if 2014 <= row["exam_year"] <= 2026
    }
    rendered = render_exam_learning(build_exam_learning_model(**inputs), TEMPLATE)
    self.assertIn('./questions/EXAM-2024-P1-Q1-S1/', rendered)
    self.assertNotIn('./questions/EXAM-2013-P1-Q1-S1/', rendered)
```

- [ ] **Step 2: Run focused tests and verify RED**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest \
  test_exam_learning test_exam_archive test_exam_template -v
```

Expected: missing detail-link projection and missing learning-link markup.

- [ ] **Step 3: Add the optional detail link to every public question projection**

Extend `build_exam_learning_model` with a keyword-only `question_detail_hrefs=None`. Validate it as an exact 362-entry map for production, require `./questions/{question_id}/`, and project `detail_href` as a string for eligible records and `None` for 2011–2013. Keep the existing 429-record accounting unchanged.

- [ ] **Step 4: Render learn-first archive and issue-hub calls to action**

For eligible archive rows, render `문제·답안·초보자 해설 보기` before the existing Q-Net link. For 2011–2013, keep the current summary, rights label, and official file link. In the issue hub's direct-exam area, link the question identity to `detail_href` when present and retain the current read-only summary when absent.

- [ ] **Step 5: Run focused tests and verify GREEN**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest \
  test_exam_learning test_exam_archive test_exam_template -v
```

Expected: 362 learning links, 429 official links, 67 link-only rows, and all existing filters remain green.

- [ ] **Step 6: Record the verified source-tree checkpoint**

```bash
sha256sum tax-study/src/tax_study/exam_learning.py tax-study/src/tax_study/exam_archive.py tax-study/templates/exam-archive.html tax-study/templates/exams.html tax-study/tests/test_exam_learning.py tax-study/tests/test_exam_archive.py tax-study/tests/test_exam_template.py
```

Expected: seven SHA-256 lines are retained after routing tests pass.

### Task 9: Publish and verify all 362 detail pages atomically

**Files:**
- Modify: `tax-study/src/tax_study/pages.py`
- Modify: `tax-study/src/tax_study/__init__.py`
- Modify: `tax-study/tests/test_pages.py`

- [ ] **Step 1: Write failing expanded-site tests**

```python
def test_expanded_site_contains_exactly_362_exam_question_readers(self):
    with tempfile.TemporaryDirectory() as temporary:
        repository = Path(temporary) / "public-repository"
        repository.mkdir()
        (repository / MARKER_NAME).write_text(MARKER_TEXT, encoding="utf-8")
        docs = pages_module().build_expanded_pages_site(
            copy.deepcopy(CANONICAL_STATE), SOURCE_PROJECT, repository, "2026-08-30"
        )
        readers = sorted(docs.glob("exams/questions/*/index.html"))
        self.assertEqual(len(readers), 362)
        self.assertTrue((docs / "exams/questions/EXAM-2014-P1-Q1-S1/index.html").is_file())
        self.assertFalse((docs / "exams/questions/EXAM-2013-P1-Q1-S1/index.html").exists())

def test_expanded_manifest_grows_from_141_to_503_files(self):
    with tempfile.TemporaryDirectory() as temporary:
        repository = Path(temporary) / "public-repository"
        repository.mkdir()
        (repository / MARKER_NAME).write_text(MARKER_TEXT, encoding="utf-8")
        docs = pages_module().build_expanded_pages_site(
            copy.deepcopy(CANONICAL_STATE), SOURCE_PROJECT, repository, "2026-08-30"
        )
        self.assertEqual(len({p for p in docs.rglob("*") if p.is_file()}), 503)
```

- [ ] **Step 2: Run the pages integration test and verify RED**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest \
  test_pages.ExpandedPagesIntegrationTests -v
```

Expected: no question-reader files and the old 141-file manifest.

- [ ] **Step 3: Load details and build reader models before staging**

In `build_expanded_pages_site`, load `data/exam-question-learning`, build the 362 reader models, pass the detail-href map into `build_exam_learning_model`, and verify the new template digest in the same trust boundary as the existing templates.

- [ ] **Step 4: Extend staging and approved-file validation**

Create `staging/exams/questions/{question_id}/`, render each model to `index.html`, and add every relative path to `expected_files`. Update the expected count from 141 to 503. Parse every generated HTML file through the existing output safety scanner; require one main element, one visible source label, one visible nonofficial-answer notice, valid internal links, no local paths, and no external resources other than approved anchor destinations. Include the new reader directory in fsync, staging identity, rollback, and prior-site validation.

- [ ] **Step 5: Export the new public APIs lazily**

Add `build_exam_question_reader_models`, `load_exam_question_details`, and `render_exam_question_reader` to focused lazy-export sets in `tax_study/__init__.py`; do not import the full pages module eagerly.

- [ ] **Step 6: Run pages integration tests and verify GREEN**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest \
  test_pages.ExpandedPagesIntegrationTests \
  test_pages.PagesInputSafetyTests -v
```

Expected: 503 files, 362 question readers, no 2011–2013 detail pages, and rollback/security tests remain green.

- [ ] **Step 7: Record the verified source-tree checkpoint**

```bash
sha256sum tax-study/src/tax_study/pages.py tax-study/src/tax_study/__init__.py tax-study/tests/test_pages.py
```

Expected: three SHA-256 lines are retained after Pages integration tests pass.

### Task 10: Run full verification, content review, and GitHub Pages deployment

**Files:**
- Verify: all files introduced or changed in Tasks 1–9
- Publish output: `tax-study-pages/docs/exams/questions/*/index.html`

- [ ] **Step 1: Run the complete exam and pages suite**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src:tests python3 -B -m unittest \
  test_exam_question_sources \
  test_exam_question_data \
  test_exam_question_learning \
  test_exam_question_reader \
  test_exam_issue_overlay \
  test_exam_learning \
  test_exam_learning_data \
  test_exam_ntba_legal_accuracy \
  test_exam_vat_legal_accuracy \
  test_exam_template \
  test_exam_archive \
  test_exams \
  test_precedents \
  test_pages.ExpandedPagesIntegrationTests \
  test_pages.PagesInputSafetyTests \
  test_project_docs -v
```

Expected: all tests pass with no warnings or skipped production-completeness checks.

- [ ] **Step 2: Run static checks**

```bash
cd tax-study
PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=src python3 -B -m py_compile \
  src/tax_study/exam_question_sources.py \
  src/tax_study/exam_question_data.py \
  src/tax_study/exam_question_learning.py \
  src/tax_study/exam_question_reader.py \
  src/tax_study/pages.py
git diff --check
```

Expected: both commands exit 0.

- [ ] **Step 3: Perform bounded legal/content review**

Review at least one question from every year, every subject part, every model-answer role, every table-bearing source, and every related-issue relation. Additionally review all records that reference tax-audit procedure, VAT zero-rating, deemed supply, non-deductible input tax, corporate entertainment expenses, wrongful calculation, income attribution, tax adjustments, or precedent connections. Compare prompt text to the digest-verified official source and compare legal conclusions to the exam-date law source.

- [ ] **Step 4: Build the public tree atomically**

Run the existing expanded Pages build command with the project root and `tax-study-pages` repository. Confirm the output tree contains 503 files, the archive contains 429 rows, and `exams/questions` contains 362 directories.

- [ ] **Step 5: Inspect representative mobile pages**

At widths 320px, 375px, and 430px, inspect the first 2014 question, a table-heavy calculation question, a tax-audit question, a VAT question, and the last 2026 question. Confirm no horizontal overflow, visible source/answer labels, readable tables, 44px navigation targets, and working previous/list/next links.

- [ ] **Step 6: Commit generated public output and push**

```bash
cd tax-study-pages
git add docs
git diff --cached --check
git commit -m "feat: publish exam question learning readers"
git push origin main
```

Stage only paths actually produced by the verified build; preserve `.tax-study-pages-publish.lock` as untracked.

- [ ] **Step 7: Verify the live site**

Check these live contracts after GitHub Pages reports a successful deployment:

```text
https://wttear.github.io/tax-study-reader/exams/archive/
https://wttear.github.io/tax-study-reader/exams/questions/EXAM-2014-P1-Q1-S1/
https://wttear.github.io/tax-study-reader/exams/questions/EXAM-2026-P2-Q4-S3/
```

Require HTTP 200, live/local SHA-256 equality for representative files, 362 learning links in the archive, no 2011–2013 detail URL, and an exact remote commit SHA match.
