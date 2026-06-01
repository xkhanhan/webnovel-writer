# Review Critical Fixes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 修复四份 review 报告交叉验证后的关键问题：CLI JSON 文件输入边界、rejected commit 投影、async 测试配置、KnowledgeQuery schema 漂移、StateManager 状态写入分叉、vector summary 覆盖、review 流程文档、安全边界。

**Architecture:** 先恢复写作主链和测试套件可信度，再扩展 RAG 投影覆盖，最后收紧本地 dashboard/backup 的安全默认值。每个任务都包含回归测试，避免只改实现不固定行为。

**Tech Stack:** Python 3.10+, pytest/pytest-asyncio, SQLite, FastAPI, Markdown skills

---

## File Structure

### Core Commit/Projection

- `webnovel-writer/scripts/chapter_commit.py`: CLI entry point; rejected commit must also run state projection.
- `webnovel-writer/scripts/data_modules/chapter_commit_service.py`: projection orchestration; non-accepted commits should still allow the state writer rejected branch.
- `webnovel-writer/scripts/data_modules/event_projection_router.py`: writer selection; rejected commits require `state`.
- `webnovel-writer/scripts/data_modules/tests/test_chapter_commit_service.py`: service-level projection behavior tests.
- `webnovel-writer/scripts/data_modules/tests/test_projection_writers.py`: end-to-end projection regression tests.

### Test Infrastructure

- `pytest.ini`: remove async plugin bans; keep coverage settings.
- `webnovel-writer/scripts/data_modules/tests/test_rag_adapter.py`: existing async tests should run normally after config fix.

### CLI JSON Input Boundary

- `webnovel-writer/scripts/data_modules/cli_args.py`: add optional base-dir containment for `@file` JSON arguments while keeping stdin/direct JSON behavior unchanged.
- `webnovel-writer/scripts/data_modules/index_manager.py`: pass project-root containment to `load_json_arg()` for write/input JSON commands.
- `webnovel-writer/scripts/data_modules/state_manager.py`: pass project-root containment to `load_json_arg()` for `process-chapter`.
- `webnovel-writer/scripts/data_modules/sql_state_manager.py`: pass project-root containment to `load_json_arg()` for chapter entity processing.
- `webnovel-writer/scripts/data_modules/memory/store.py`: pass project-root containment to `load_json_arg()` for memory upserts/imports.
- `webnovel-writer/scripts/data_modules/rag_adapter.py`: pass project-root containment to `load_json_arg()` for scene indexing.
- `webnovel-writer/scripts/data_modules/style_sampler.py`: pass project-root containment to `load_json_arg()` for scene extraction.
- `webnovel-writer/scripts/data_modules/tests/test_coverage_boost.py`: extend existing `cli_args` tests.

### Knowledge Query

- `webnovel-writer/scripts/data_modules/knowledge_query.py`: query production `relationship_events.type` while returning stable JSON for callers.
- `webnovel-writer/scripts/data_modules/tests/test_knowledge_query.py`: use production schema, not mock-only `relationship_type`.

### State Manager

- `webnovel-writer/scripts/data_modules/state_manager.py`: route `set_chapter_status()` through locked merge semantics.
- `webnovel-writer/scripts/data_modules/tests/test_chapter_status.py`: preserve status monotonicity.
- `webnovel-writer/scripts/data_modules/tests/test_state_manager_extra.py`: add merge regression for existing disk state.

### Vector Projection/RAG

- `webnovel-writer/scripts/data_modules/vector_projection_writer.py`: add summary and scene chunks to commit projection, and avoid `asyncio.run()` failure when projection is invoked from an active event loop.
- `webnovel-writer/scripts/data_modules/tests/test_vector_projection_writer.py`: cover summary/scene chunk generation, stable IDs, and active-event-loop storage bridge.

### Review Skill Flow

- `webnovel-writer/skills/webnovel-review/SKILL.md`: switch to `review-pipeline --save-metrics`, remove obsolete second save command.
- `webnovel-writer/skills/webnovel-write/SKILL.md`: verify already uses `--save-metrics`; no change unless wording drifts.

### Local Security Defaults

- `webnovel-writer/dashboard/app.py`: restrict CORS to localhost origins, add file size limit for read API.
- `webnovel-writer/scripts/backup_manager.py`: ensure generated `.gitignore` excludes `.env`.
- `webnovel-writer/scripts/data_modules/style_sampler.py`: tolerate corrupt JSON in `tags`.
- New or existing tests under `webnovel-writer/scripts/tests/` or `webnovel-writer/scripts/data_modules/tests/` for these fixes.

---

## Task 0: Bound `@file` JSON Arguments to Project Roots

**Problem:** `load_json_arg("@path")` currently reads any local path. This is a local CLI feature rather than a remote vulnerability, but in Agent-driven pipelines the project root is the expected trust boundary.

**Design:** Keep backward compatibility for direct callers by adding an optional `base_dir` argument. If `base_dir` is passed, `@file` must resolve inside it. `@-` stdin and direct JSON strings are unchanged.

**Files:**
- Modify: `webnovel-writer/scripts/data_modules/cli_args.py`
- Modify: `webnovel-writer/scripts/data_modules/index_manager.py`
- Modify: `webnovel-writer/scripts/data_modules/state_manager.py`
- Modify: `webnovel-writer/scripts/data_modules/sql_state_manager.py`
- Modify: `webnovel-writer/scripts/data_modules/memory/store.py`
- Modify: `webnovel-writer/scripts/data_modules/rag_adapter.py`
- Modify: `webnovel-writer/scripts/data_modules/style_sampler.py`
- Test: `webnovel-writer/scripts/data_modules/tests/test_coverage_boost.py`

- [ ] **Step 1: Add failing containment tests**

Append these tests to the `cli_args` section of `webnovel-writer/scripts/data_modules/tests/test_coverage_boost.py`:

```python
def test_load_json_arg_rejects_file_outside_base_dir(tmp_path):
    project = tmp_path / "project"
    project.mkdir()
    outside = tmp_path / "secret.json"
    outside.write_text('{"secret": true}', encoding="utf-8")

    with pytest.raises(ValueError, match="outside allowed directory"):
        load_json_arg(f"@{outside}", base_dir=project)


def test_load_json_arg_allows_file_inside_base_dir(tmp_path):
    project = tmp_path / "project"
    project.mkdir()
    payload = project / "payload.json"
    payload.write_text('{"ok": true}', encoding="utf-8")

    assert load_json_arg(f"@{payload}", base_dir=project) == {"ok": True}


def test_load_json_arg_stdin_ignores_base_dir(monkeypatch, tmp_path):
    monkeypatch.setattr(sys, "stdin", StringIO('{"stdin": true}'))

    assert load_json_arg("@-", base_dir=tmp_path) == {"stdin": True}
```

- [ ] **Step 2: Run failing cli_args tests**

Run:

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests/test_coverage_boost.py::test_load_json_arg_rejects_file_outside_base_dir webnovel-writer/scripts/data_modules/tests/test_coverage_boost.py::test_load_json_arg_allows_file_inside_base_dir webnovel-writer/scripts/data_modules/tests/test_coverage_boost.py::test_load_json_arg_stdin_ignores_base_dir -q --no-cov
```

Expected before implementation: fail with `TypeError: load_json_arg() got an unexpected keyword argument 'base_dir'`.

- [ ] **Step 3: Implement optional containment helper**

Update `webnovel-writer/scripts/data_modules/cli_args.py`:

```python
def _resolve_json_arg_file(target: str, *, base_dir: str | Path | None = None) -> Path:
    path = Path(target).expanduser()
    if not path.is_absolute() and base_dir is not None:
        path = Path(base_dir) / path
    resolved = path.resolve()
    if base_dir is not None:
        base = Path(base_dir).expanduser().resolve()
        try:
            resolved.relative_to(base)
        except ValueError as exc:
            raise ValueError(f"json arg file outside allowed directory: {resolved}") from exc
    return resolved
```

Change the signature and file read branch:

```python
def load_json_arg(raw: str, *, base_dir: str | Path | None = None) -> Any:
    """
    解析 CLI 传入的 JSON 参数，支持两种形式：
    - 直接 JSON 字符串：'{"a":1}'
    - @ 文件路径：'@data.json'（从文件读取 JSON，避免 shell 引号地狱）
      - 特例：'@-' 表示从 stdin 读取
      - 当传入 base_dir 时，@ 文件必须位于 base_dir 内
    """
    if raw is None:
        raise ValueError("missing json arg")
    text = str(raw).strip()
    if text.startswith("@"):
        target = text[1:].strip()
        if not target:
            raise ValueError("invalid json arg: '@' without path")
        if target == "-":
            content = sys.stdin.read()
        else:
            content = _resolve_json_arg_file(target, base_dir=base_dir).read_text(encoding="utf-8")
        return json.loads(content)
    return json.loads(text)
```

- [ ] **Step 4: Pass project root from CLI call sites**

For each call site that has `args.project_root`, pass it as `base_dir=args.project_root`.

In `webnovel-writer/scripts/data_modules/index_manager.py`, update all `load_json_arg(...)` calls in command handlers, for example:

```python
        entities = load_json_arg(args.entities, base_dir=args.project_root)
        scenes = load_json_arg(args.scenes, base_dir=args.project_root)
```

and:

```python
        data = load_json_arg(args.data, base_dir=args.project_root)
```

In `webnovel-writer/scripts/data_modules/state_manager.py`:

```python
        data = load_json_arg(args.data, base_dir=args.project_root)
```

In `webnovel-writer/scripts/data_modules/sql_state_manager.py`:

```python
        data = load_json_arg(args.data, base_dir=args.project_root)
```

In `webnovel-writer/scripts/data_modules/memory/store.py`:

```python
        payload = load_json_arg(args.data, base_dir=args.project_root)
```

In `webnovel-writer/scripts/data_modules/rag_adapter.py`:

```python
        scenes = load_json_arg(args.scenes, base_dir=args.project_root)
```

In `webnovel-writer/scripts/data_modules/style_sampler.py`:

```python
        scenes = load_json_arg(args.scenes, base_dir=args.project_root)
```

If a specific module stores the resolved project root in `config.project_root` rather than `args.project_root`, use `base_dir=config.project_root`.

- [ ] **Step 5: Run cli and unified CLI tests**

Run:

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests/test_coverage_boost.py webnovel-writer/scripts/data_modules/tests/test_webnovel_unified_cli.py -q --no-cov
```

Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git add webnovel-writer/scripts/data_modules/cli_args.py webnovel-writer/scripts/data_modules/index_manager.py webnovel-writer/scripts/data_modules/state_manager.py webnovel-writer/scripts/data_modules/sql_state_manager.py webnovel-writer/scripts/data_modules/memory/store.py webnovel-writer/scripts/data_modules/rag_adapter.py webnovel-writer/scripts/data_modules/style_sampler.py webnovel-writer/scripts/data_modules/tests/test_coverage_boost.py
git commit -m "fix: bound json file arguments to project root"
```

---

## Task 1: Restore Rejected Commit State Projection

**Problem:** `StateProjectionWriter` supports `rejected -> chapter_rejected`, but both CLI and service skip projections for non-accepted commits.

**Files:**
- Modify: `webnovel-writer/scripts/chapter_commit.py`
- Modify: `webnovel-writer/scripts/data_modules/chapter_commit_service.py`
- Modify: `webnovel-writer/scripts/data_modules/event_projection_router.py`
- Test: `webnovel-writer/scripts/data_modules/tests/test_chapter_commit_service.py`
- Test: `webnovel-writer/scripts/data_modules/tests/test_projection_writers.py`

- [ ] **Step 1: Add failing service test for rejected projection**

Append this test to `webnovel-writer/scripts/data_modules/tests/test_chapter_commit_service.py`:

```python
import json


def test_apply_projections_updates_state_for_rejected_commit(tmp_path):
    (tmp_path / ".webnovel").mkdir(parents=True, exist_ok=True)
    (tmp_path / ".webnovel" / "state.json").write_text("{}", encoding="utf-8")

    service = ChapterCommitService(tmp_path)
    payload = service.build_commit(
        chapter=7,
        review_result={"blocking_count": 1},
        fulfillment_result={
            "planned_nodes": ["进入坊市"],
            "covered_nodes": ["进入坊市"],
            "missed_nodes": [],
            "extra_nodes": [],
        },
        disambiguation_result={"pending": []},
        extraction_result={"state_deltas": [], "entity_deltas": [], "accepted_events": []},
    )

    projected = service.apply_projections(payload)

    state = json.loads((tmp_path / ".webnovel" / "state.json").read_text(encoding="utf-8"))
    assert projected["projection_status"]["state"] == "done"
    assert state["progress"]["chapter_status"]["7"] == "chapter_rejected"
```

- [ ] **Step 2: Run the failing test**

Run:

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests/test_chapter_commit_service.py::test_apply_projections_updates_state_for_rejected_commit -q --no-cov
```

Expected before implementation: fail because `projection_status["state"]` remains `pending` or `state.json` has no `chapter_rejected`.

- [ ] **Step 3: Route rejected commits to state writer**

Update `webnovel-writer/scripts/data_modules/event_projection_router.py`:

```python
    def required_writers(self, commit_payload: Dict) -> List[str]:
        writers: Set[str] = set()
        status = str((commit_payload.get("meta") or {}).get("status") or "")
        if status == "rejected":
            writers.add("state")
            return sorted(writers)
        if status == "accepted":
            writers.add("state")
            writers.add("index")
        if commit_payload.get("entity_deltas"):
            writers.add("index")
        if str(commit_payload.get("summary_text") or "").strip():
            writers.add("summary")
        for event in commit_payload.get("accepted_events") or []:
            if not isinstance(event, dict):
                continue
            writers.update(self.route(event))
        return sorted(writers)
```

- [ ] **Step 4: Allow service projection for rejected state only**

Update the start of `ChapterCommitService.apply_projections()` in `webnovel-writer/scripts/data_modules/chapter_commit_service.py`:

```python
    def apply_projections(self, payload: Dict[str, Any]) -> Dict[str, Any]:
        status = str((payload.get("meta") or {}).get("status") or "")
        if status not in {"accepted", "rejected"}:
            return payload

        if status == "accepted":
            chapter = int((payload.get("meta") or {}).get("chapter") or 0)
            event_store = EventLogStore(self.project_root)
            payload["accepted_events"] = event_store.normalize_events(
                chapter, payload.get("accepted_events", [])
            )
            event_store.write_events(chapter, payload["accepted_events"])

            proposals = AmendProposalTrigger().check(chapter, payload.get("accepted_events", []))
            if proposals:
                manager = IndexManager(DataModulesConfig.from_project_root(self.project_root))
                with manager._get_conn() as conn:
                    ensure_override_ledger_columns(conn)
                    persist_amend_proposals(conn, chapter, proposals)
                    conn.commit()
```

Keep the writer import block and writer loop after this block. This preserves event log writes and override proposals for accepted commits only, while letting rejected commits reach `StateProjectionWriter`.

- [ ] **Step 5: Make CLI always call `apply_projections()`**

Update `webnovel-writer/scripts/chapter_commit.py`:

```python
    service.persist_commit(payload)
    payload = service.apply_projections(payload)
    print(json.dumps(payload, ensure_ascii=False))
```

- [ ] **Step 6: Run targeted projection tests**

Run:

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests/test_chapter_commit_service.py webnovel-writer/scripts/data_modules/tests/test_projection_writers.py -q --no-cov
```

Expected: all tests pass.

- [ ] **Step 7: Commit**

```bash
git add webnovel-writer/scripts/chapter_commit.py webnovel-writer/scripts/data_modules/chapter_commit_service.py webnovel-writer/scripts/data_modules/event_projection_router.py webnovel-writer/scripts/data_modules/tests/test_chapter_commit_service.py
git commit -m "fix: project rejected chapter commits to state"
```

---

## Task 2: Restore Async Pytest Execution

**Problem:** `pytest.ini` disables `asyncio` and `anyio`, so `@pytest.mark.asyncio` tests fail or are not exercised correctly.

**Files:**
- Modify: `pytest.ini`
- Verify: existing async tests under `webnovel-writer/scripts/data_modules/tests/test_rag_adapter.py`

- [ ] **Step 1: Confirm current async failure**

Run:

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests/test_rag_adapter.py::test_store_and_search -q --no-cov
```

Expected before implementation: fail or warn due to disabled async plugin.

- [ ] **Step 2: Update pytest config**

Change `pytest.ini` to:

```ini
[pytest]
testpaths = webnovel-writer/scripts/data_modules/tests webnovel-writer/scripts/tests
pythonpath = webnovel-writer/scripts
asyncio_mode = auto
addopts = -p no:debugging -p pytest_cov -q --cov --cov-report=term-missing --cov-fail-under=90 -p no:cacheprovider
```

Do not disable `pytest_asyncio` or `anyio`.

- [ ] **Step 3: Run async-focused tests**

Run:

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests/test_rag_adapter.py -q --no-cov
```

Expected: async tests execute and pass.

- [ ] **Step 4: Run full suite once**

Run:

```bash
python -m pytest -q
```

Expected: tests run with coverage enforcement. If failures remain, record exact failing tests before touching unrelated code.

- [ ] **Step 5: Commit**

```bash
git add pytest.ini
git commit -m "test: enable pytest async plugins"
```

---

## Task 3: Fix KnowledgeQuery Relationship Schema Drift

**Problem:** Production table uses `relationship_events.type`; `KnowledgeQuery` and its tests use `relationship_type`.

**Files:**
- Modify: `webnovel-writer/scripts/data_modules/knowledge_query.py`
- Modify: `webnovel-writer/scripts/data_modules/tests/test_knowledge_query.py`

- [ ] **Step 1: Rewrite test fixture to production schema**

In `test_knowledge_query.py`, replace the `relationship_events` table definition with:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS relationship_events (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            from_entity TEXT,
            to_entity TEXT,
            type TEXT NOT NULL,
            action TEXT DEFAULT '',
            polarity TEXT DEFAULT '',
            strength REAL DEFAULT 0.0,
            description TEXT,
            chapter INTEGER,
            scene_index INTEGER DEFAULT 0,
            evidence TEXT DEFAULT '',
            confidence REAL DEFAULT 1.0,
            created_at TEXT
        )
    """)
```

Replace inserts with:

```python
    conn.execute(
        "INSERT INTO relationship_events (from_entity, to_entity, type, chapter) VALUES (?, ?, ?, ?)",
        ("hanli", "陈巧倩", "同门", 20),
    )
    conn.execute(
        "INSERT INTO relationship_events (from_entity, to_entity, type, chapter) VALUES (?, ?, ?, ?)",
        ("hanli", "陈巧倩", "合作", 45),
    )
```

Keep assertions against output key `relationship_type` for backward-compatible CLI JSON.

- [ ] **Step 2: Run the failing KnowledgeQuery tests**

Run:

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests/test_knowledge_query.py -q --no-cov
```

Expected before implementation: fail with `no such column: relationship_type`.

- [ ] **Step 3: Query production column with output compatibility**

Update `KnowledgeQuery.entity_relationships_at_chapter()` in `knowledge_query.py`:

```python
            rows = conn.execute(
                """
                SELECT from_entity, to_entity, type AS relationship_type, description, chapter
                FROM relationship_events
                WHERE (from_entity = ? OR to_entity = ?) AND chapter <= ?
                ORDER BY chapter ASC, id ASC
                """,
                (entity_id, entity_id, chapter),
            ).fetchall()
```

Leave returned JSON as:

```python
                    "relationship_type": str(row["relationship_type"] or "").strip(),
```

- [ ] **Step 4: Run KnowledgeQuery and CLI-adjacent tests**

Run:

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests/test_knowledge_query.py webnovel-writer/scripts/data_modules/tests/test_webnovel_unified_cli.py -q --no-cov
```

Expected: pass.

- [ ] **Step 5: Commit**

```bash
git add webnovel-writer/scripts/data_modules/knowledge_query.py webnovel-writer/scripts/data_modules/tests/test_knowledge_query.py
git commit -m "fix: query relationship events using production schema"
```

---

## Task 4: Route Chapter Status Writes Through Locked Merge Semantics

**Problem:** `set_chapter_status()` mutates in-memory state and calls `_save_state()`, bypassing `save_state()` locking and disk merge.

**Files:**
- Modify: `webnovel-writer/scripts/data_modules/state_manager.py`
- Modify: `webnovel-writer/scripts/data_modules/tests/test_state_manager_extra.py`
- Verify: `webnovel-writer/scripts/data_modules/tests/test_chapter_status.py`

- [ ] **Step 1: Add failing locked-save regression tests**

Append to `test_state_manager_extra.py`:

```python
def test_set_chapter_status_uses_locked_save_state(temp_project, monkeypatch):
    manager = StateManager(temp_project, enable_sqlite_sync=False)
    called = {}

    def fake_save_state():
        called["save_state"] = True

    def fail_direct_save():
        raise AssertionError("set_chapter_status must use save_state()")

    monkeypatch.setattr(manager, "save_state", fake_save_state)
    monkeypatch.setattr(manager, "_save_state", fail_direct_save)

    manager.set_chapter_status(5, "chapter_drafted")

    assert called["save_state"] is True
    assert manager._pending_chapter_status == {"5": "chapter_drafted"}


def test_set_chapter_status_preserves_existing_disk_state(temp_project):
    temp_project.state_file.write_text(
        json.dumps(
            {
                "progress": {"current_chapter": 4, "chapter_status": {"4": "chapter_committed"}},
                "disambiguation_warnings": [{"chapter": 4, "mention": "宗主"}],
            },
            ensure_ascii=False,
        ),
        encoding="utf-8",
    )

    manager = StateManager(temp_project, enable_sqlite_sync=False)
    manager.set_chapter_status(5, "chapter_drafted")

    saved = json.loads(temp_project.state_file.read_text(encoding="utf-8"))
    assert saved["progress"]["chapter_status"]["4"] == "chapter_committed"
    assert saved["progress"]["chapter_status"]["5"] == "chapter_drafted"
    assert saved["disambiguation_warnings"] == [{"chapter": 4, "mention": "宗主"}]
```

- [ ] **Step 2: Run status tests before implementation**

Run:

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests/test_chapter_status.py webnovel-writer/scripts/data_modules/tests/test_state_manager_extra.py::test_set_chapter_status_uses_locked_save_state webnovel-writer/scripts/data_modules/tests/test_state_manager_extra.py::test_set_chapter_status_preserves_existing_disk_state -q --no-cov
```

Expected before implementation: `test_set_chapter_status_uses_locked_save_state` fails because current code calls `_save_state()` directly and has no `_pending_chapter_status`.

- [ ] **Step 3: Add pending chapter status field**

In `StateManager.__init__`, after `_pending_progress_chapter`, add:

```python
        self._pending_chapter_status: Dict[str, str] = {}
```

Update `has_pending` in `save_state()` to include:

```python
                self._pending_chapter_status,
```

- [ ] **Step 4: Merge pending chapter statuses inside `save_state()`**

Inside the locked `with lock:` block, immediately after `progress` is normalized for progress updates or before disambiguation merge, add:

```python
                if self._pending_chapter_status:
                    progress = disk_state.get("progress", {})
                    if not isinstance(progress, dict):
                        progress = {}
                        disk_state["progress"] = progress
                    chapter_status = progress.get("chapter_status")
                    if not isinstance(chapter_status, dict):
                        chapter_status = {}
                        progress["chapter_status"] = chapter_status
                    chapter_status.update(self._pending_chapter_status)
                    progress["last_updated"] = self._now_progress_timestamp()
```

- [ ] **Step 5: Clear pending chapter statuses only after successful write**

Where `save_state()` clears other pending structures after `atomic_write_json`, add:

```python
                self._pending_chapter_status.clear()
```

Place it beside the existing pending clears, not before SQLite sync error handling.

- [ ] **Step 6: Update `set_chapter_status()`**

Replace the final mutation/write block with:

```python
        progress = self._state.setdefault("progress", {})
        chapter_status = progress.setdefault("chapter_status", {})
        chapter_status[str(chapter)] = status
        self._pending_chapter_status[str(chapter)] = status
        self.save_state()
```

Keep monotonicity checks unchanged.

- [ ] **Step 7: Run StateManager status tests**

Run:

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests/test_chapter_status.py webnovel-writer/scripts/data_modules/tests/test_state_manager_extra.py -q --no-cov
```

Expected: pass.

- [ ] **Step 8: Commit**

```bash
git add webnovel-writer/scripts/data_modules/state_manager.py webnovel-writer/scripts/data_modules/tests/test_state_manager_extra.py
git commit -m "fix: merge chapter status updates under state lock"
```

---

## Task 5: Add Summary/Scene Chunks and Safe Async Bridge to Vector Projection

**Problem:** Commit projection writes event/entity chunks only. RAG already supports `summary` and `scene`, but commit projection does not feed them. The writer also calls `asyncio.run()` directly, which fails if this synchronous projection path is ever invoked inside an active event loop.

**Files:**
- Modify: `webnovel-writer/scripts/data_modules/vector_projection_writer.py`
- Modify: `webnovel-writer/scripts/data_modules/tests/test_vector_projection_writer.py`

- [ ] **Step 1: Add failing test for summary and scene chunks**

Append to `test_vector_projection_writer.py`:

```python
def test_collect_chunks_includes_summary_and_scenes():
    writer = VectorProjectionWriter.__new__(VectorProjectionWriter)
    payload = {
        "meta": {"chapter": 47, "status": "accepted"},
        "summary_text": "韩立在坊市发现丹方线索。",
        "scenes": [
            {"index": 1, "summary": "韩立入坊市观察摊位", "location": "坊市"},
            {"scene_index": 2, "content": "陈巧倩暗中提醒韩立有人跟踪。"},
        ],
        "accepted_events": [],
        "entity_deltas": [],
    }

    chunks = writer._collect_chunks(payload)

    by_type = {chunk["chunk_type"]: chunk for chunk in chunks}
    assert by_type["summary"]["chunk_id"] == "ch0047_summary"
    assert by_type["summary"]["parent_chunk_id"] is None
    assert by_type["scene"]["parent_chunk_id"] == "ch0047_summary"
    assert any(chunk["scene_index"] == 2 for chunk in chunks if chunk["chunk_type"] == "scene")
```

- [ ] **Step 2: Run failing vector test**

Run:

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests/test_vector_projection_writer.py::test_collect_chunks_includes_summary_and_scenes -q --no-cov
```

Expected before implementation: fail because no summary/scene chunks exist.

- [ ] **Step 3: Add summary chunk collection**

At the start of `_collect_chunks()` after `chunk_counts`, add:

```python
        summary_text = str(commit_payload.get("summary_text") or "").strip()
        summary_chunk_id = f"ch{chapter:04d}_summary" if chapter > 0 else ""
        if chapter > 0 and summary_text:
            chunks.append({
                "chunk_id": summary_chunk_id,
                "chapter": chapter,
                "scene_index": 0,
                "content": summary_text,
                "chunk_type": "summary",
                "parent_chunk_id": None,
                "source_file": f"commit:chapter_{chapter:03d}",
            })
```

- [ ] **Step 4: Add scene chunk collection**

After the summary block, add:

```python
        for idx, scene in enumerate(commit_payload.get("scenes") or [], start=1):
            if not isinstance(scene, dict):
                continue
            scene_index = int(scene.get("scene_index") or scene.get("index") or idx)
            text = str(scene.get("summary") or scene.get("content") or "").strip()
            location = str(scene.get("location") or "").strip()
            if location and text:
                text = f"{location}：{text}"
            if not text:
                continue
            chunk_id = self._chunk_id("scene", chapter, scene_index)
            chunks.append({
                "chunk_id": chunk_id,
                "chapter": chapter,
                "scene_index": scene_index,
                "content": text,
                "chunk_type": "scene",
                "parent_chunk_id": summary_chunk_id or None,
                "source_file": f"commit:chapter_{chapter:03d}",
            })
```

- [ ] **Step 5: Run vector and RAG tests**

Run:

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests/test_vector_projection_writer.py webnovel-writer/scripts/data_modules/tests/test_rag_adapter.py -q --no-cov
```

Expected: pass before the async bridge change, except any failures introduced by summary/scene implementation should be fixed before continuing.

- [ ] **Step 6: Add active event loop regression test**

Append to `test_vector_projection_writer.py`:

```python
import pytest


@pytest.mark.asyncio
async def test_run_store_coro_works_inside_active_event_loop():
    writer = VectorProjectionWriter.__new__(VectorProjectionWriter)

    async def store():
        return 3

    assert writer._run_store_coro(store()) == 3
```

- [ ] **Step 7: Run failing active-loop test**

Run:

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests/test_vector_projection_writer.py::test_run_store_coro_works_inside_active_event_loop -q --no-cov
```

Expected before implementation: fail because `_run_store_coro` does not exist.

- [ ] **Step 8: Implement safe coroutine bridge**

In `webnovel-writer/scripts/data_modules/vector_projection_writer.py`, add imports:

```python
import threading
from collections.abc import Coroutine
```

Add this helper method to `VectorProjectionWriter`:

```python
    def _run_store_coro(self, coro: Coroutine[Any, Any, int]) -> int:
        try:
            asyncio.get_running_loop()
        except RuntimeError:
            return int(asyncio.run(coro) or 0)

        result: dict[str, Any] = {}

        def runner() -> None:
            try:
                result["value"] = asyncio.run(coro)
            except Exception as exc:
                result["error"] = exc

        thread = threading.Thread(target=runner, daemon=True)
        thread.start()
        thread.join()
        if "error" in result:
            raise result["error"]
        return int(result.get("value") or 0)
```

Then change `_store_chunks()` from:

```python
            stored = asyncio.run(adapter.store_chunks(chunks))
            return stored
```

to:

```python
            return self._run_store_coro(adapter.store_chunks(chunks))
```

This keeps the synchronous public API intact while moving the coroutine to a short-lived thread when the caller already owns an event loop.

- [ ] **Step 9: Run vector and RAG tests**

Run:

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests/test_vector_projection_writer.py webnovel-writer/scripts/data_modules/tests/test_rag_adapter.py -q --no-cov
```

Expected: pass.

- [ ] **Step 10: Commit**

```bash
git add webnovel-writer/scripts/data_modules/vector_projection_writer.py webnovel-writer/scripts/data_modules/tests/test_vector_projection_writer.py
git commit -m "feat: project summaries and scenes to vectors safely"
```

---

## Task 6: Unify Review Skill Metrics Flow

**Problem:** CLI supports `review-pipeline --save-metrics`; `webnovel-review` still documents the old two-step flow.

**Files:**
- Modify: `webnovel-writer/skills/webnovel-review/SKILL.md`
- Verify: `webnovel-writer/skills/webnovel-write/SKILL.md`

- [ ] **Step 1: Update common mistakes**

In `webnovel-review/SKILL.md`, replace:

```markdown
- ❌ 把 report 文件生成等同于已落库（`save-review-metrics` 未跑）
```

with:

```markdown
- ❌ 把 report 文件生成等同于已落库（`review-pipeline --save-metrics` 未跑）
```

- [ ] **Step 2: Replace Step 5 command block**

Replace the two-command standard flow with:

```bash
python -X utf8 "${SCRIPTS_DIR}/webnovel.py" --project-root "${PROJECT_ROOT}" review-pipeline \
  --chapter {chapter_num} \
  --review-results "${PROJECT_ROOT}/.webnovel/tmp/review_results.json" \
  --metrics-out "${PROJECT_ROOT}/.webnovel/tmp/review_metrics.json" \
  --report-file "审查报告/第{chapter_num}章审查报告.md" \
  --save-metrics
```

Replace the requirement:

```markdown
- `review-pipeline` 生成的 `review_metrics.json` 必须可直接写入 `review_metrics` 表
```

with:

```markdown
- `review-pipeline --save-metrics` 必须完成报告生成、metrics 文件输出、`review_metrics` 表写入
```

- [ ] **Step 3: Grep for obsolete instruction**

Run:

```bash
rg -n "save-review-metrics|--save-metrics" webnovel-writer/skills/webnovel-review/SKILL.md webnovel-writer/skills/webnovel-write/SKILL.md
```

Expected: `webnovel-review` no longer instructs a separate `index save-review-metrics` call; both skills mention `--save-metrics`.

- [ ] **Step 4: Commit**

```bash
git add webnovel-writer/skills/webnovel-review/SKILL.md
git commit -m "docs: unify review metrics persistence flow"
```

---

## Task 7: Tighten Local Dashboard and Backup Safety Defaults

**Problem:** Dashboard allows `*` CORS while exposing local project text. Backup-created `.gitignore` does not exclude `.env`.

**Files:**
- Modify: `webnovel-writer/dashboard/app.py`
- Modify: `webnovel-writer/scripts/backup_manager.py`
- Test: add `webnovel-writer/scripts/tests/test_dashboard_security.py`
- Test: add `webnovel-writer/scripts/tests/test_backup_manager.py`

- [ ] **Step 1: Add dashboard security tests**

Create `webnovel-writer/scripts/tests/test_dashboard_security.py`:

```python
from fastapi.testclient import TestClient

from dashboard.app import create_app


def test_dashboard_cors_allows_localhost_origin(tmp_path):
    (tmp_path / ".webnovel").mkdir(parents=True)
    (tmp_path / ".webnovel" / "state.json").write_text("{}", encoding="utf-8")
    app = create_app(tmp_path)
    client = TestClient(app)

    response = client.options(
        "/api/project/info",
        headers={
            "Origin": "http://localhost:5173",
            "Access-Control-Request-Method": "GET",
        },
    )

    assert response.headers["access-control-allow-origin"] == "http://localhost:5173"


def test_dashboard_cors_rejects_untrusted_origin(tmp_path):
    (tmp_path / ".webnovel").mkdir(parents=True)
    (tmp_path / ".webnovel" / "state.json").write_text("{}", encoding="utf-8")
    app = create_app(tmp_path)
    client = TestClient(app)

    response = client.options(
        "/api/project/info",
        headers={
            "Origin": "https://example.com",
            "Access-Control-Request-Method": "GET",
        },
    )

    assert "access-control-allow-origin" not in response.headers
```

- [ ] **Step 2: Restrict CORS origins**

In `dashboard/app.py`, add module-level constant:

```python
LOCAL_CORS_ORIGINS = [
    "http://localhost",
    "http://localhost:5173",
    "http://localhost:8000",
    "http://127.0.0.1",
    "http://127.0.0.1:5173",
    "http://127.0.0.1:8000",
]
```

Change middleware setup to:

```python
    app.add_middleware(
        CORSMiddleware,
        allow_origins=LOCAL_CORS_ORIGINS,
        allow_methods=["GET"],
        allow_headers=["*"],
    )
```

- [ ] **Step 3: Add file read size guard**

In `file_read()`, before `read_text`, add:

```python
        max_bytes = 2 * 1024 * 1024
        if resolved.stat().st_size > max_bytes:
            raise HTTPException(413, "文件过大，无法预览")
```

- [ ] **Step 4: Add backup gitignore test**

Create `webnovel-writer/scripts/tests/test_backup_manager.py`:

```python
import subprocess

from backup_manager import GitBackupManager


def test_backup_manager_gitignore_excludes_env(tmp_path, monkeypatch):
    calls = []

    def fake_run(args, cwd=None, check=False, capture_output=False, text=False):
        calls.append(args)
        if args == ["git", "init"]:
            (tmp_path / ".git").mkdir()
        return subprocess.CompletedProcess(args=args, returncode=0, stdout="", stderr="")

    monkeypatch.setattr("backup_manager.is_git_available", lambda: True)
    monkeypatch.setattr("backup_manager.subprocess.run", fake_run)

    GitBackupManager(str(tmp_path))

    gitignore = (tmp_path / ".gitignore").read_text(encoding="utf-8")
    assert ".env" in gitignore
    assert ".env.*" in gitignore
    assert "!.env.example" in gitignore
```

- [ ] **Step 5: Update backup `.gitignore` template**

In `backup_manager.py`, add this block to the generated `.gitignore`:

```gitignore
# Env (keep .env.example)
.env
.env.*
!.env.example
```

- [ ] **Step 6: Run security tests**

Run:

```bash
python -m pytest webnovel-writer/scripts/tests/test_dashboard_security.py webnovel-writer/scripts/tests/test_backup_manager.py -q --no-cov
```

Expected: pass.

- [ ] **Step 7: Commit**

```bash
git add webnovel-writer/dashboard/app.py webnovel-writer/scripts/backup_manager.py webnovel-writer/scripts/tests/test_dashboard_security.py webnovel-writer/scripts/tests/test_backup_manager.py
git commit -m "fix: tighten dashboard cors and backup gitignore defaults"
```

---

## Task 8: Harden StyleSampler Tag JSON Parsing

**Problem:** A corrupt `tags` JSON value in `style_samples.db` crashes listing.

**Files:**
- Modify: `webnovel-writer/scripts/data_modules/style_sampler.py`
- Modify: `webnovel-writer/scripts/data_modules/tests/test_style_sampler_cli.py`

- [ ] **Step 1: Add corrupt tag regression test**

Append to `test_style_sampler_cli.py`:

```python
def test_style_sampler_ignores_corrupt_tag_json(temp_project):
    sampler = StyleSampler(temp_project)
    with sampler._get_conn() as conn:
        conn.execute(
            """
            INSERT INTO style_samples
            (id, chapter, scene_type, content, score, tags, created_at)
            VALUES (?, ?, ?, ?, ?, ?, CURRENT_TIMESTAMP)
            """,
            ("bad-tags", 1, SceneType.BATTLE.value, "战斗描写" * 50, 0.8, "[bad-json"),
        )
        conn.commit()

    samples = sampler.get_best_samples(limit=5)

    assert samples[0].id == "bad-tags"
    assert samples[0].tags == []
```

If the schema column order differs, inspect `_ensure_db()` in `style_sampler.py` and adjust column names exactly.

- [ ] **Step 2: Run failing test**

Run:

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests/test_style_sampler_cli.py::test_style_sampler_ignores_corrupt_tag_json -q --no-cov
```

Expected before implementation: fail with `json.JSONDecodeError`.

- [ ] **Step 3: Add safe JSON helper**

In `style_sampler.py`, add:

```python
def _safe_json_list(raw) -> list:
    if not raw:
        return []
    try:
        value = json.loads(raw)
    except (TypeError, json.JSONDecodeError):
        return []
    return value if isinstance(value, list) else []
```

Change row mapping from:

```python
            tags=json.loads(row[5]) if row[5] else [],
```

to:

```python
            tags=_safe_json_list(row[5]),
```

- [ ] **Step 4: Run style sampler tests**

Run:

```bash
python -m pytest webnovel-writer/scripts/data_modules/tests/test_style_sampler_cli.py -q --no-cov
```

Expected: pass.

- [ ] **Step 5: Commit**

```bash
git add webnovel-writer/scripts/data_modules/style_sampler.py webnovel-writer/scripts/data_modules/tests/test_style_sampler_cli.py
git commit -m "fix: tolerate corrupt style sample tags"
```

---

## Task 9: Final Verification

**Files:** read-only verification, plus fixes if tests reveal regressions.

- [ ] **Step 1: Compile Python sources**

Run:

```bash
python -m compileall -q webnovel-writer/scripts webnovel-writer/dashboard
```

Expected: exit code 0.

- [ ] **Step 2: Run focused regression suite**

Run:

```bash
python -m pytest \
  webnovel-writer/scripts/data_modules/tests/test_chapter_commit_service.py \
  webnovel-writer/scripts/data_modules/tests/test_coverage_boost.py \
  webnovel-writer/scripts/data_modules/tests/test_projection_writers.py \
  webnovel-writer/scripts/data_modules/tests/test_knowledge_query.py \
  webnovel-writer/scripts/data_modules/tests/test_chapter_status.py \
  webnovel-writer/scripts/data_modules/tests/test_state_manager_extra.py \
  webnovel-writer/scripts/data_modules/tests/test_vector_projection_writer.py \
  webnovel-writer/scripts/data_modules/tests/test_style_sampler_cli.py \
  webnovel-writer/scripts/tests/test_dashboard_security.py \
  webnovel-writer/scripts/tests/test_backup_manager.py \
  -q --no-cov
```

Expected: all pass.

- [ ] **Step 3: Run full suite**

Run:

```bash
python -m pytest -q
```

Expected: all tests pass and coverage is at least 90%.

- [ ] **Step 4: Inspect review skill command consistency**

Run:

```bash
rg -n "save-review-metrics|review-pipeline|--save-metrics" webnovel-writer/skills/webnovel-review/SKILL.md webnovel-writer/skills/webnovel-write/SKILL.md
```

Expected: both review flows use `review-pipeline --save-metrics`; no active instruction requires separate `index save-review-metrics`.

- [ ] **Step 5: Commit final test adjustments if needed**

If verification required small test-only fixes:

```bash
git add <changed-files>
git commit -m "test: cover critical review fixes"
```

If no further changes were needed, do not create an empty commit.

---

## Out of Scope for This Plan

- Full `/webnovel-resync` implementation for manually edited chapters.
- Full automatic review-fix loop.
- Full SQLite migration framework.
- Large refactors of `index_manager.py`, `status_reporter.py`, or `DataModulesConfig`.
- Dashboard frontend performance work beyond security defaults.
- Anti-AI/style quality product line improvements.

These should be planned separately after the correctness and test-trust fixes land.
