# Agent Lab P1-A Core Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 交付可安装的离线实验规格与证据内核，支持校验、冻结、持久事件、导出及验真。

**Architecture:** CLI 调用拥有 SQLite 与对象存储的应用服务。冻结规格使用规范 JSON 和 SHA-256，事件保存稳定序号和内容引用。这个子项目不执行模型请求；它是后续运行器、插件和看台共同依赖的内核。

**Tech Stack:** Python 3.11+、Pydantic 2、SQLite、标准库 argparse、pytest、uv。

---

## 执行范围与准备

规格：[P1-A](../specs/2026-10-07-agent-lab-core-design.md)。公共契约：[contracts](../specs/2026-10-07-agent-lab-contracts.md)。完整 P1 的后续交付：[路线图](./2026-10-07-agent-lab-delivery-roadmap.md)。本计划的完成声明只覆盖 P1-A，不覆盖实际模型实验、后台调度、插件安装或看台。

所有文件路径以 `shiyanpingtai` 仓库根目录为准。实现使用单独 worktree 和 `codex/` 分支。开始前检查工作区干净，并按实际环境取得 Python 3.11+ 与 uv。若缺少 uv，使用受控 Python 环境安装 uv；不要改写系统 Python。包依赖以 `uv.lock` 固定，实现时记录实际解析版本。

每个任务先增加有意义的行为测试，观察该测试失败，再实现对应行为，运行该任务测试并提交。首次失败必须来自当前行为未实现；依赖安装失败或测试未收集不算有效的红灯。以下代码块是具体文件内容或向已定义类追加的方法，均为待实施代码，不表示本轮已经创建运行代码。

文件职责与规格中的表一致。测试目录只覆盖核心不变量；包装、文档等低影响工作用安装与静态检查验收。

## Task 1：可安装包与基础错误

**Files:** Create `pyproject.toml`, `src/agentlab/__init__.py`, `src/agentlab/errors.py`。Modify `.gitignore`。

- [ ] 创建包配置，使用以下完整内容：

```toml
[build-system]
requires = ["hatchling>=1.27,<2"]
build-backend = "hatchling.build"

[project]
name = "agentlab-experiments"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = ["pydantic>=2.10,<3"]

[project.scripts]
agentlab = "agentlab.cli:main"

[dependency-groups]
dev = ["pytest>=8,<9"]

[tool.hatch.build.targets.wheel]
packages = ["src/agentlab"]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

- [ ] 创建 `src/agentlab/__init__.py`，内容为 `__version__ = "0.1.0"`。创建 `errors.py`：

```python
EXIT_CODES = {
    "INVALID_SPEC": 2, "BUDGET_EXCEEDED": 2,
    "IDEMPOTENCY_CONFLICT": 3, "NOT_FOUND": 4,
    "WORKSPACE_MISMATCH": 5, "INTEGRITY_ERROR": 6,
    "CAPABILITY_UNSUPPORTED": 7, "CONNECTION_UNAVAILABLE": 8,
    "STALE_ATTEMPT": 9,
}

class CoreError(ValueError):
    def __init__(self, code: str, message: str):
        self.code = code
        self.message = message
        super().__init__(message)
```

- [ ] `.gitignore` 增加 `*.egg-info/`、`.coverage`；保留 `.agentlab/` 和 `.worktrees/`。运行 `uv lock`、`uv sync --group dev`。预期依赖解析和包安装成功；此时命令入口的运行验收留在 Task 8，不宣称 CLI 已可用。
- [ ] 运行 `uv run python -c "import agentlab; print(agentlab.__version__)"`，预期 `0.1.0`。运行 `git diff --check`。
- [ ] 提交：`git add pyproject.toml uv.lock src/agentlab/__init__.py src/agentlab/errors.py .gitignore`，`git commit -m "build: prepare Agent Lab core package"`。

## Task 2：规格、哈希与有界矩阵

**Files:** Create `src/agentlab/specs.py`, `experiments/temporal-baseline.json`, `tests/core/conftest.py`, `tests/core/test_specs.py`。

- [ ] 创建样例规格，内容使用公共契约第 2 节的完整 JSON。两变体、两场景、两重复，应展开 8 项。创建测试夹具：

```python
# tests/core/conftest.py
import json
from pathlib import Path
import pytest

@pytest.fixture
def document():
    path = Path(__file__).resolve().parents[2] / "experiments" / "temporal-baseline.json"
    return json.loads(path.read_text(encoding="utf-8"))
```

```python
# tests/core/test_specs.py
import pytest
from agentlab.errors import CoreError
from agentlab.specs import compile_spec

def test_stable_hash_and_matrix(document):
    first = compile_spec(document)
    second = compile_spec(dict(reversed(list(document.items()))))
    assert first.sha256 == second.sha256
    assert len(first.jobs) == 8
    assert len({(j.variant_id, j.scenario_id, j.repeat) for j in first.jobs}) == 8

def test_budget_prevents_large_expansion(document):
    document["repeats"] = 100_000_000
    with pytest.raises(CoreError) as error:
        compile_spec(document)
    assert error.value.code == "BUDGET_EXCEEDED"

def test_duplicate_variant_and_nonfinite_config_rejected(document):
    document["variants"][1]["id"] = document["variants"][0]["id"]
    with pytest.raises(CoreError):
        compile_spec(document)
    document["variants"][1]["id"] = "summary"
    document["variants"][0]["config"] = {"threshold": float("nan")}
    with pytest.raises(CoreError):
        compile_spec(document)
```

- [ ] 运行 `uv run pytest tests/core/test_specs.py -q`，预期因 `compile_spec` 尚不存在失败。依赖安装应已成功。
- [ ] 创建 `specs.py`：

```python
import hashlib
import json
from dataclasses import dataclass
from typing import Any, Literal
from pydantic import BaseModel, ConfigDict, Field, ValidationError
from .errors import CoreError

class StrictModel(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True, frozen=True)

class Limits(StrictModel):
    max_runs: int = Field(gt=0)
    max_model_calls: int = Field(gt=0)
    max_seconds: int = Field(gt=0)
    max_concurrency: int = Field(gt=0)

class Variant(StrictModel):
    id: str = Field(min_length=1)
    connection_id: str = Field(min_length=1)
    connection_version: str = Field(min_length=1)
    model: str = Field(min_length=1)
    parameters: dict[str, Any] = Field(default_factory=dict)
    strategy: str = Field(min_length=1)
    strategy_version: str = Field(min_length=1)
    config: dict[str, Any]

class ExperimentSpec(StrictModel):
    schema_version: Literal[1]
    question: str = Field(min_length=1)
    dataset_id: str = Field(min_length=1)
    dataset_version: str = Field(min_length=1)
    scenario_ids: list[str] = Field(min_length=1)
    repeats: int = Field(gt=0)
    variants: list[Variant] = Field(min_length=1)
    limits: Limits
    evaluator_id: str = Field(min_length=1)
    evaluator_version: str = Field(min_length=1)

def canonical_json(value: Any) -> str:
    try:
        text = json.dumps(value, ensure_ascii=False, sort_keys=True,
                          separators=(",", ":"), allow_nan=False)
        text.encode("utf-8")
        return text
    except (TypeError, ValueError) as error:
        raise CoreError("INVALID_SPEC", "Expected finite JSON data") from error

def digest(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest()

@dataclass(frozen=True)
class Job:
    variant_id: str
    scenario_id: str
    repeat: int

@dataclass(frozen=True)
class CompiledSpec:
    json_text: str
    sha256: str
    jobs: tuple[Job, ...]

    def data(self) -> dict:
        return json.loads(self.json_text)

def compile_spec(raw: dict) -> CompiledSpec:
    canonical_json(raw)
    try:
        spec = ExperimentSpec.model_validate(raw)
    except ValidationError as error:
        raise CoreError("INVALID_SPEC", "Invalid experiment specification") from error
    variant_ids = [variant.id for variant in spec.variants]
    scenario_ids = spec.scenario_ids
    if (len(set(variant_ids)) != len(variant_ids)
            or len(set(scenario_ids)) != len(scenario_ids)
            or any(not item.strip() for item in [*variant_ids, *scenario_ids])):
        raise CoreError("INVALID_SPEC", "IDs must be nonempty and unique")
    count = len(variant_ids) * len(scenario_ids) * spec.repeats
    if count > spec.limits.max_runs:
        raise CoreError("BUDGET_EXCEEDED", "Matrix exceeds max_runs")
    text = canonical_json(spec.model_dump(mode="json"))
    jobs = tuple(Job(v, s, r) for v in variant_ids
                 for s in scenario_ids for r in range(spec.repeats))
    return CompiledSpec(text, digest(text.encode("utf-8")), jobs)
```

- [ ] 重跑本任务测试，预期 3 项通过；预算检查在展开之前执行，哈希由规范 JSON 产生。
- [ ] 提交 `src/agentlab/specs.py experiments/temporal-baseline.json tests/core/conftest.py tests/core/test_specs.py`，消息 `feat: compile immutable experiment specifications`。

## Task 3：工作区身份与初始化

**Files:** Create `src/agentlab/workspace.py`, `tests/core/test_workspace.py`。

- [ ] 写入以下测试，运行 `uv run pytest tests/core/test_workspace.py -q`，预期尚无 Workspace 时失败：

```python
from uuid import UUID
import pytest
from agentlab.errors import CoreError
from agentlab.workspace import Workspace

def test_identity_survives_reopen(tmp_path):
    first = Workspace.open(tmp_path, create=True)
    second = Workspace.open(tmp_path)
    assert first.workspace_id == second.workspace_id
    assert str(UUID(first.workspace_id)) == first.workspace_id

def test_read_does_not_initialize_missing_workspace(tmp_path):
    with pytest.raises(CoreError) as error:
        Workspace.open(tmp_path)
    assert error.value.code == "NOT_FOUND"
    assert not (tmp_path / ".agentlab").exists()

def test_two_processes_initialize_one_identity(tmp_path):
    import subprocess
    import sys
    code = "from pathlib import Path; from agentlab.workspace import Workspace; import sys; print(Workspace.open(Path(sys.argv[1]), create=True).workspace_id)"
    workers = [subprocess.Popen([sys.executable, "-c", code, str(tmp_path)],
               stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True) for _ in range(2)]
    results = [worker.communicate(timeout=10) for worker in workers]
    assert all(worker.returncode == 0 for worker in workers), results
    assert results[0][0].strip() == results[1][0].strip()

@pytest.mark.parametrize("value", ["not-json", '{"schema_version":1,"workspace_id":0}'])
def test_corrupt_identity_is_not_reinitialized(tmp_path, value):
    workspace = Workspace.open(tmp_path, create=True)
    marker = workspace.data / "workspace.json"
    marker.write_text(value)
    with pytest.raises(CoreError) as error:
        Workspace.open(tmp_path, create=True)
    assert error.value.code == "INTEGRITY_ERROR"
    assert marker.read_text() == value
```

- [ ] 创建模块；本地首版针对 macOS/Linux 使用进程锁，Windows 支持单独评估，不静默省略锁：

```python
import fcntl
import json
import os
import tempfile
from dataclasses import dataclass
from pathlib import Path
from uuid import UUID, uuid4
from .errors import CoreError
from .specs import canonical_json

@dataclass(frozen=True)
class Workspace:
    root: Path
    data: Path
    workspace_id: str

    @classmethod
    def open(cls, root: Path, create: bool = False):
        root = Path(root).expanduser().resolve()
        data = root / ".agentlab"
        marker = data / "workspace.json"
        if not create and not marker.is_file():
            raise CoreError("NOT_FOUND", "Workspace is not initialized")
        if data.is_symlink() or marker.is_symlink():
            raise CoreError("INTEGRITY_ERROR", "Workspace metadata cannot be a symlink")
        data.mkdir(parents=True, exist_ok=True, mode=0o700)
        with (data / "init.lock").open("a+b") as lock:
            fcntl.flock(lock, fcntl.LOCK_EX)
            if not marker.exists():
                value = {"schema_version": 1, "workspace_id": str(uuid4())}
                fd, name = tempfile.mkstemp(dir=data)
                try:
                    with os.fdopen(fd, "w", encoding="utf-8") as stream:
                        stream.write(canonical_json(value))
                        stream.flush()
                        os.fsync(stream.fileno())
                    os.replace(name, marker)
                    directory_fd = os.open(data, os.O_RDONLY)
                    try:
                        os.fsync(directory_fd)
                    finally:
                        os.close(directory_fd)
                finally:
                    Path(name).unlink(missing_ok=True)
            try:
                value = json.loads(marker.read_text(encoding="utf-8"))
                if (not isinstance(value, dict)
                        or type(value.get("schema_version")) is not int
                        or value["schema_version"] != 1
                        or not isinstance(value.get("workspace_id"), str)):
                    raise ValueError("Invalid identity format")
                identifier = str(UUID(value["workspace_id"]))
            except (KeyError, ValueError, TypeError) as error:
                raise CoreError("INTEGRITY_ERROR", "Invalid workspace identity") from error
        return cls(root, data, identifier)
```

- [ ] 运行本任务测试，两个同时初始化的 subprocess 必须得到同一 ID。再运行 `uv run pytest tests/core -q`。
- [ ] 提交模块和测试，消息 `feat: persist local workspace identity`。

## Task 4：内容寻址的证据对象

**Files:** Create `src/agentlab/artifacts.py`, `tests/core/test_artifacts.py`。

- [ ] 写入测试，运行 `uv run pytest tests/core/test_artifacts.py -q`，预期缺少 ArtifactStore 时失败：

```python
import pytest
from agentlab.artifacts import ArtifactStore
from agentlab.errors import CoreError

def test_content_deduplicates_and_tampering_fails(tmp_path):
    store = ArtifactStore(tmp_path / "objects")
    identifier = store.put(b"original evidence")
    assert store.put(b"original evidence") == identifier
    assert store.get(identifier) == b"original evidence"
    (store.root / identifier).write_bytes(b"changed evidence")
    with pytest.raises(CoreError) as error:
        store.get(identifier)
    assert error.value.code == "INTEGRITY_ERROR"

def test_rejects_path_as_identifier(tmp_path):
    store = ArtifactStore(tmp_path / "objects")
    with pytest.raises(CoreError):
        store.get("../outside")

def test_failed_replace_leaves_no_valid_object(tmp_path, monkeypatch):
    store = ArtifactStore(tmp_path / "objects")
    def fail_replace(source, target):
        raise OSError("simulated disk failure")
    monkeypatch.setattr("agentlab.artifacts.os.replace", fail_replace)
    with pytest.raises(OSError):
        store.put(b"uncommitted")
    assert list(store.root.iterdir()) == []
```

- [ ] 创建模块：

```python
import os
import re
import tempfile
from pathlib import Path
from .errors import CoreError
from .specs import digest

class ArtifactStore:
    def __init__(self, root: Path):
        self.root = Path(root)
        if self.root.is_symlink():
            raise CoreError("INTEGRITY_ERROR", "Object directory cannot be a symlink")
        self.root.mkdir(parents=True, exist_ok=True, mode=0o700)

    def _path(self, identifier: str) -> Path:
        if not re.fullmatch(r"[a-f0-9]{64}", identifier):
            raise CoreError("INVALID_SPEC", "Expected an artifact ID")
        return self.root / identifier

    def put(self, content: bytes) -> str:
        identifier = digest(content)
        target = self._path(identifier)
        if target.exists():
            self.get(identifier)
            return identifier
        fd, name = tempfile.mkstemp(dir=self.root)
        try:
            with os.fdopen(fd, "wb") as stream:
                stream.write(content)
                stream.flush()
                os.fsync(stream.fileno())
            os.replace(name, target)
            directory_fd = os.open(self.root, os.O_RDONLY)
            try:
                os.fsync(directory_fd)
            finally:
                os.close(directory_fd)
        finally:
            Path(name).unlink(missing_ok=True)
        return identifier

    def get(self, identifier: str) -> bytes:
        target = self._path(identifier)
        if target.is_symlink():
            raise CoreError("INTEGRITY_ERROR", "Artifact cannot be a symlink")
        if not target.is_file():
            raise CoreError("NOT_FOUND", "Artifact does not exist")
        content = target.read_bytes()
        if digest(content) != identifier:
            raise CoreError("INTEGRITY_ERROR", "Artifact hash mismatch")
        return content
```

- [ ] 运行本任务与核心测试，预期全部通过；检查临时文件不会在失败写入后遗留为有效对象。该对象存储不被宣传为签名或防恶意管理员篡改。
- [ ] 提交模块与测试，消息 `feat: store evidence by content hash`。

## Task 5：事务冻结与幂等键

**Files:** Create `src/agentlab/store.py`, `tests/core/test_freeze.py`。

- [ ] 写入测试，运行 `uv run pytest tests/core/test_freeze.py -q`，预期缺少 Store 时失败：

```python
import pytest
from agentlab.errors import CoreError
from agentlab.store import Store
from agentlab.workspace import Workspace

def test_freeze_replay_conflict_and_reopen(tmp_path, document):
    workspace = Workspace.open(tmp_path, create=True)
    store = Store(workspace)
    first = store.freeze(document, "request-1")
    assert store.freeze(document, "request-1")["id"] == first["id"]
    document["question"] = "different question"
    with pytest.raises(CoreError) as error:
        store.freeze(document, "request-1")
    assert error.value.code == "IDEMPOTENCY_CONFLICT"
    assert store.db.execute("SELECT count(*) FROM frozen_specs").fetchone()[0] == 1
    store.close()
    reopened = Store(Workspace.open(tmp_path))
    result = reopened.inspect(first["id"])
    result["spec"]["question"] = "changed local copy"
    assert reopened.inspect(first["id"])["spec"]["question"] != "changed local copy"
    reopened.close()

def test_two_connections_replay_one_freeze(tmp_path, document):
    from concurrent.futures import ThreadPoolExecutor
    workspace = Workspace.open(tmp_path, create=True)
    def freeze_in_connection(_):
        local = Store(workspace)
        try:
            return local.freeze(document, "same-key")["id"]
        finally:
            local.close()
    with ThreadPoolExecutor(max_workers=2) as executor:
        identifiers = list(executor.map(freeze_in_connection, range(2)))
    assert len(set(identifiers)) == 1
    check = Store(workspace)
    assert check.db.execute("SELECT count(*) FROM frozen_specs").fetchone()[0] == 1
    check.close()
```

- [ ] 创建 Store，数据 schema 与冻结事务如下。Task 6 在同一类中追加事件操作：

```python
import json
import sqlite3
from contextlib import contextmanager
from dataclasses import asdict
from datetime import datetime, timezone
from uuid import uuid4
from .artifacts import ArtifactStore
from .errors import CoreError
from .specs import compile_spec, canonical_json, digest

SCHEMA = """
BEGIN IMMEDIATE;
CREATE TABLE IF NOT EXISTS frozen_specs (
 id TEXT PRIMARY KEY, sha256 TEXT NOT NULL UNIQUE,
 json_text TEXT NOT NULL, created_at TEXT NOT NULL
);
CREATE TABLE IF NOT EXISTS freeze_keys (
 key TEXT PRIMARY KEY, frozen_id TEXT NOT NULL REFERENCES frozen_specs(id)
);
CREATE TABLE IF NOT EXISTS events (
 sequence INTEGER PRIMARY KEY AUTOINCREMENT,
 event_id TEXT NOT NULL UNIQUE,
 frozen_id TEXT NOT NULL REFERENCES frozen_specs(id),
 kind TEXT NOT NULL, observed_at TEXT NOT NULL, artifact_id TEXT NOT NULL
);
PRAGMA user_version=1;
COMMIT;
"""

class Store:
    def __init__(self, workspace):
        self.workspace = workspace
        self.artifacts = ArtifactStore(workspace.data / "objects")
        self.db = sqlite3.connect(workspace.data / "metadata.sqlite3", isolation_level=None)
        self.db.row_factory = sqlite3.Row
        self.db.execute("PRAGMA foreign_keys=ON")
        self.db.execute("PRAGMA journal_mode=WAL")
        self.db.execute("PRAGMA busy_timeout=5000")
        version = self.db.execute("PRAGMA user_version").fetchone()[0]
        if version not in (0, 1):
            self.db.close()
            raise CoreError("INTEGRITY_ERROR", "Unsupported database version")
        if version == 0:
            self.db.executescript(SCHEMA)

    def close(self):
        self.db.close()

    @contextmanager
    def transaction(self):
        self.db.execute("BEGIN IMMEDIATE")
        try:
            yield
            self.db.execute("COMMIT")
        except BaseException:
            if self.db.in_transaction:
                self.db.execute("ROLLBACK")
            raise

    def _view(self, row):
        compiled = compile_spec(json.loads(row["json_text"]))
        if compiled.sha256 != row["sha256"]:
            raise CoreError("INTEGRITY_ERROR", "Frozen specification hash mismatch")
        return {"id": row["id"], "sha256": row["sha256"],
                "workspace_id": self.workspace.workspace_id,
                "created_at": row["created_at"], "spec": compiled.data(),
                "matrix": [asdict(job) for job in compiled.jobs]}

    def inspect(self, identifier):
        row = self.db.execute("SELECT * FROM frozen_specs WHERE id=?", (identifier,)).fetchone()
        if row is None:
            raise CoreError("NOT_FOUND", "Frozen specification does not exist")
        return self._view(row)

    def freeze(self, document, key):
        if not isinstance(key, str) or not key.strip():
            raise CoreError("INVALID_SPEC", "A nonempty idempotency key is required")
        compiled = compile_spec(document)
        with self.transaction():
            previous = self.db.execute(
                "SELECT f.* FROM freeze_keys k JOIN frozen_specs f ON f.id=k.frozen_id WHERE k.key=?",
                (key,)).fetchone()
            if previous is not None:
                if previous["sha256"] != compiled.sha256:
                    raise CoreError("IDEMPOTENCY_CONFLICT", "Key already names different content")
                identifier = previous["id"]
            else:
                existing = self.db.execute("SELECT id FROM frozen_specs WHERE sha256=?",
                                           (compiled.sha256,)).fetchone()
                identifier = existing["id"] if existing else str(uuid4())
                if existing is None:
                    self.db.execute("INSERT INTO frozen_specs VALUES (?,?,?,?)",
                        (identifier, compiled.sha256, compiled.json_text,
                         datetime.now(timezone.utc).isoformat()))
                self.db.execute("INSERT INTO freeze_keys VALUES (?,?)", (key, identifier))
        return self.inspect(identifier)
```

- [ ] 运行本任务测试与 `uv run pytest tests/core -q`，预期冲突后没有新增冻结对象，重开读取正常，两个独立 Store 连接的同键提交只产生一行。
- [ ] 提交模块与测试，消息 `feat: freeze specifications transactionally`。

## Task 6：稳定序号的证据事件

**Files:** Modify `src/agentlab/store.py`；Create `tests/core/test_events.py`。

- [ ] 写入测试并运行 `uv run pytest tests/core/test_events.py -q`，预期 Store 缺少 append/list_events 时失败：

```python
import pytest
from agentlab.store import Store
from agentlab.workspace import Workspace
from agentlab.errors import CoreError

def test_events_reopen_and_cursor(tmp_path, document):
    workspace = Workspace.open(tmp_path, create=True)
    store = Store(workspace)
    frozen = store.freeze(document, "freeze")
    first = store.append(frozen["id"], "note", {"text": "first"})
    second = store.append(frozen["id"], "note", {"text": "second"})
    store.close()
    reopened = Store(workspace)
    page = reopened.list_events(frozen["id"], after=first["sequence"])
    assert [item["event_id"] for item in page] == [second["event_id"]]
    assert page[0]["payload"] == {"text": "second"}
    with pytest.raises(CoreError):
        reopened.append(frozen["id"], "model.response", {"made_up": True})
    reopened.close()
```

- [ ] 向 Store 追加以下完整方法：

```python
    def _event(self, row):
        payload = json.loads(self.artifacts.get(row["artifact_id"]).decode("utf-8"))
        return {"event_id": row["event_id"], "sequence": row["sequence"],
                "workspace_id": self.workspace.workspace_id,
                "frozen_spec_id": row["frozen_id"], "submission_id": None,
                "run_id": None, "attempt_id": None, "kind": row["kind"],
                "observed_at": row["observed_at"], "payload": payload,
                "artifact_ids": [row["artifact_id"]]}

    def append(self, frozen_id, kind, payload):
        if kind not in ("note", "test.observation"):
            raise CoreError("INVALID_SPEC", "Public append accepts notes and test observations only")
        self.inspect(frozen_id)
        artifact_id = self.artifacts.put(canonical_json(payload).encode("utf-8"))
        event_id = str(uuid4())
        with self.transaction():
            self.db.execute(
                "INSERT INTO events(event_id,frozen_id,kind,observed_at,artifact_id) VALUES (?,?,?,?,?)",
                (event_id, frozen_id, kind, datetime.now(timezone.utc).isoformat(), artifact_id))
        row = self.db.execute("SELECT * FROM events WHERE event_id=?", (event_id,)).fetchone()
        return self._event(row)

    def list_events(self, frozen_id, after=0, limit=100):
        self.inspect(frozen_id)
        if after < 0 or not 1 <= limit <= 1000:
            raise CoreError("INVALID_SPEC", "Invalid event cursor or page limit")
        rows = self.db.execute(
            "SELECT * FROM events WHERE frozen_id=? AND sequence>? ORDER BY sequence LIMIT ?",
            (frozen_id, after, limit)).fetchall()
        return [self._event(row) for row in rows]

    def event_head(self, frozen_id):
        self.inspect(frozen_id)
        return self.db.execute(
            "SELECT COALESCE(MAX(sequence),0) FROM events WHERE frozen_id=?",
            (frozen_id,)).fetchone()[0]
```

- [ ] 运行本任务与核心测试。核对页面顺序和各 artifact_ids 可读；事件关联不存在规格时报 NOT_FOUND，不能写入悬空事件。
- [ ] 提交文件，消息 `feat: journal inspectable evidence events`。

## Task 7：导出和证据包验真

**Files:** Create `src/agentlab/bundles.py`, `tests/core/test_bundles.py`。

- [ ] 写入以下测试，运行 `uv run pytest tests/core/test_bundles.py -q`，预期导出函数不存在时失败：

```python
import json
import pytest
from agentlab.bundles import export_bundle, verify_bundle
from agentlab.errors import CoreError
from agentlab.store import Store
from agentlab.workspace import Workspace

def test_bundle_roundtrip_tamper_and_undeclared_file(tmp_path, document):
    store = Store(Workspace.open(tmp_path / "project", create=True))
    frozen = store.freeze(document, "freeze")
    store.append(frozen["id"], "note", {"source": "human"})
    destination = tmp_path / "bundle"
    exported = export_bundle(store, frozen["id"], destination)
    assert verify_bundle(destination)["manifest_sha256"] == exported["manifest_sha256"]
    (destination / "extra.txt").write_text("not declared")
    with pytest.raises(CoreError):
        verify_bundle(destination)
    (destination / "extra.txt").unlink()
    (destination / "frozen.json").write_text("{}")
    with pytest.raises(CoreError) as error:
        verify_bundle(destination)
    assert error.value.code == "INTEGRITY_ERROR"
    store.close()

def test_manifest_path_traversal_is_rejected(tmp_path):
    root = tmp_path / "bundle"
    root.mkdir()
    (root / "manifest.json").write_text(json.dumps({"schema_version": 1,
        "files": [{"path": "../outside", "sha256": "a" * 64}]}))
    with pytest.raises(CoreError):
        verify_bundle(root)

def test_export_rejects_existing_directory_and_nested_undeclared_file(tmp_path, document):
    store = Store(Workspace.open(tmp_path / "project", create=True))
    frozen = store.freeze(document, "freeze")
    root = tmp_path / "bundle"
    export_bundle(store, frozen["id"], root)
    with pytest.raises(CoreError):
        export_bundle(store, frozen["id"], root)
    nested = root / "other"
    nested.mkdir()
    (nested / "manifest.json").write_text("{}")
    with pytest.raises(CoreError):
        verify_bundle(root)
    store.close()

def test_symlink_and_duplicate_manifest_entries_fail(tmp_path, document):
    store = Store(Workspace.open(tmp_path / "project", create=True))
    frozen = store.freeze(document, "freeze")
    root = tmp_path / "bundle"
    export_bundle(store, frozen["id"], root)
    manifest_path = root / "manifest.json"
    manifest = json.loads(manifest_path.read_text())
    original = manifest_path.read_text()
    manifest["files"].append(manifest["files"][0])
    manifest_path.write_text(json.dumps(manifest))
    with pytest.raises(CoreError):
        verify_bundle(root)
    manifest_path.write_text(original)
    (root / "alias").symlink_to(root / "frozen.json")
    with pytest.raises(CoreError):
        verify_bundle(root)
    store.close()
```

- [ ] 创建完整模块；manifest 自身不列入其文件哈希清单，返回的 manifest hash 可由调用者另行保存。验真验证内部完整性，不提供签名来源证明：

```python
import json
import os
import shutil
from dataclasses import asdict
from pathlib import Path, PurePosixPath
from .errors import CoreError
from .specs import canonical_json, compile_spec, digest

def _write(path, content):
    path.parent.mkdir(parents=True, exist_ok=True, mode=0o700)
    with path.open("xb") as stream:
        os.chmod(path, 0o600)
        stream.write(content)

def export_bundle(store, frozen_id, destination):
    destination = Path(destination)
    destination.parent.mkdir(parents=True, exist_ok=True)
    try:
        destination.mkdir(mode=0o700)
    except FileExistsError as error:
        raise CoreError("INVALID_SPEC", "Export destination must be new") from error
    try:
        frozen = store.inspect(frozen_id)
        cutoff = store.event_head(frozen_id)
        events = []
        cursor = 0
        while True:
            page = store.list_events(frozen_id, after=cursor)
            page = [event for event in page if event["sequence"] <= cutoff]
            if not page:
                break
            events.extend(page)
            cursor = page[-1]["sequence"]
        _write(destination / "frozen.json", canonical_json(frozen).encode("utf-8"))
        _write(destination / "events.json", canonical_json(events).encode("utf-8"))
        objects = {identifier for event in events for identifier in event["artifact_ids"]}
        for identifier in sorted(objects):
            _write(destination / "objects" / identifier, store.artifacts.get(identifier))
        entries = []
        for path in sorted(destination.rglob("*")):
            if path.is_file():
                content = path.read_bytes()
                entries.append({"path": path.relative_to(destination).as_posix(),
                                "sha256": digest(content), "bytes": len(content)})
        manifest = {"schema_version": 1, "workspace_id": store.workspace.workspace_id,
                    "frozen_spec_id": frozen_id, "event_cutoff": cutoff, "files": entries}
        content = canonical_json(manifest).encode("utf-8")
        _write(destination / "manifest.json", content)
        return {"path": str(destination.resolve()), "manifest_sha256": digest(content)}
    except BaseException:
        shutil.rmtree(destination)
        raise

def _entry_path(root, value):
    pure = PurePosixPath(value)
    if (not value or "\\" in value or pure.is_absolute()
            or ".." in pure.parts or pure.as_posix() != value):
        raise CoreError("INTEGRITY_ERROR", "Unsafe manifest path")
    target = root.joinpath(*pure.parts)
    current = root
    for part in pure.parts:
        current = current / part
        if current.is_symlink():
            raise CoreError("INTEGRITY_ERROR", "Bundle cannot contain symlinks")
    return target

def verify_bundle(root):
    root = Path(root)
    if root.is_symlink() or (root / "manifest.json").is_symlink():
        raise CoreError("INTEGRITY_ERROR", "Bundle cannot be a symlink")
    try:
        content = (root / "manifest.json").read_bytes()
        manifest = json.loads(content)
        if manifest["schema_version"] != 1:
            raise CoreError("INTEGRITY_ERROR", "Unsupported bundle version")
        declared = set()
        for entry in manifest["files"]:
            path = _entry_path(root, entry["path"])
            if entry["path"] in declared or entry["path"] == "manifest.json":
                raise CoreError("INTEGRITY_ERROR", "Duplicate manifest target")
            declared.add(entry["path"])
            data = path.read_bytes()
            if digest(data) != entry["sha256"] or len(data) != entry["bytes"]:
                raise CoreError("INTEGRITY_ERROR", "Bundle content mismatch")
        present = set()
        for path in root.rglob("*"):
            if path.is_symlink():
                raise CoreError("INTEGRITY_ERROR", "Bundle cannot contain symlinks")
            if path.is_file() and path.relative_to(root).as_posix() != "manifest.json":
                present.add(path.relative_to(root).as_posix())
        if declared != present:
            raise CoreError("INTEGRITY_ERROR", "Undeclared or missing bundle files")
        frozen = json.loads((root / "frozen.json").read_bytes())
        events = json.loads((root / "events.json").read_bytes())
        compiled = compile_spec(frozen["spec"])
        if (frozen["id"] != manifest["frozen_spec_id"]
                or frozen["workspace_id"] != manifest["workspace_id"]
                or compiled.sha256 != frozen["sha256"]
                or [asdict(job) for job in compiled.jobs] != frozen["matrix"]):
            raise CoreError("INTEGRITY_ERROR", "Bundle identity mismatch")
        previous = 0
        for event in events:
            if (event["sequence"] <= previous
                    or event["sequence"] > manifest["event_cutoff"]
                    or event["frozen_spec_id"] != frozen["id"]
                    or event["workspace_id"] != frozen["workspace_id"]):
                raise CoreError("INTEGRITY_ERROR", "Event identity or ordering mismatch")
            previous = event["sequence"]
            for identifier in event["artifact_ids"]:
                path = _entry_path(root, "objects/" + identifier)
                if digest(path.read_bytes()) != identifier:
                    raise CoreError("INTEGRITY_ERROR", "Artifact identity mismatch")
            if len(event["artifact_ids"]) != 1:
                raise CoreError("INTEGRITY_ERROR", "Unexpected core event artifact count")
            payload_path = _entry_path(root, "objects/" + event["artifact_ids"][0])
            if json.loads(payload_path.read_bytes()) != event["payload"]:
                raise CoreError("INTEGRITY_ERROR", "Event payload mismatch")
        if previous != manifest["event_cutoff"]:
            raise CoreError("INTEGRITY_ERROR", "Incomplete event snapshot")
        return {"valid": True, "workspace_id": manifest["workspace_id"],
                "frozen_spec_id": frozen["id"], "manifest_sha256": digest(content)}
    except CoreError:
        raise
    except (OSError, ValueError, KeyError, TypeError) as error:
        raise CoreError("INTEGRITY_ERROR", "Malformed or incomplete bundle") from error
```

- [ ] 运行本任务与核心测试，符号链接、重复清单路径、未声明嵌套文件必须失败。验证失败后不删改用户提供的验真目录。导出失败只能清理本调用新建的目录，不删除既有目录。
- [ ] 提交模块与测试，消息 `feat: export and verify evidence bundles`。

## Task 8：CLI 命令与结构化错误

**Files:** Create `src/agentlab/cli.py`, `tests/core/test_cli.py`。

- [ ] 创建命令测试；第一次运行 `uv run pytest tests/core/test_cli.py -q` 应因 CLI 模块不存在失败：

```python
import json
import subprocess
import sys
from pathlib import Path

def call_cli(workspace, *arguments):
    return subprocess.run([sys.executable, "-m", "agentlab.cli", "--workspace",
        str(workspace), *arguments, "--json"], capture_output=True, text=True)

def test_cli_roundtrip_and_domain_error(tmp_path):
    initialized = call_cli(tmp_path, "init")
    assert initialized.returncode == 0
    assert json.loads(initialized.stdout)["workspace_id"]
    source = Path(__file__).resolve().parents[2] / "experiments" / "temporal-baseline.json"
    frozen = call_cli(tmp_path, "spec", "freeze", str(source), "--key", "cli-1")
    assert frozen.returncode == 0
    identifier = json.loads(frozen.stdout)["result"]["id"]
    replay = call_cli(tmp_path, "spec", "freeze", str(source), "--key", "cli-1")
    assert json.loads(replay.stdout)["result"]["id"] == identifier
    missing = call_cli(tmp_path, "spec", "inspect", "missing")
    assert missing.returncode == 4
    assert json.loads(missing.stdout)["error"]["code"] == "NOT_FOUND"
    assert frozen.stderr == ""
```

- [ ] 创建 CLI。响应 request_id 来自本次命令；JSON 失败不回显原始输入：

```python
import argparse
import json
import sqlite3
import sys
from contextlib import closing
from dataclasses import asdict
from pathlib import Path
from uuid import uuid4
from .bundles import export_bundle, verify_bundle
from .errors import CoreError, EXIT_CODES
from .specs import compile_spec
from .store import Store
from .workspace import Workspace

class Parser(argparse.ArgumentParser):
    def error(self, message):
        raise CoreError("INVALID_SPEC", "Invalid command arguments")

def parser():
    common = Parser(add_help=False)
    common.add_argument("--json", action="store_true")
    root = Parser(prog="agentlab")
    root.add_argument("--workspace", type=Path, default=Path.cwd())
    commands = root.add_subparsers(dest="command", required=True)
    commands.add_parser("init", parents=[common])
    spec = commands.add_parser("spec").add_subparsers(dest="action", required=True)
    for action in ("validate", "freeze", "inspect"):
        child = spec.add_parser(action, parents=[common])
        child.add_argument("source")
        if action == "freeze":
            child.add_argument("--key", required=True)
    evidence = commands.add_parser("evidence").add_subparsers(dest="action", required=True)
    append = evidence.add_parser("append", parents=[common])
    append.add_argument("frozen_id")
    append.add_argument("--kind", required=True)
    append.add_argument("--file", type=Path, required=True)
    listing = evidence.add_parser("list", parents=[common])
    listing.add_argument("frozen_id")
    listing.add_argument("--after", type=int, default=0)
    exporting = commands.add_parser("export", parents=[common])
    exporting.add_argument("frozen_id")
    exporting.add_argument("destination", type=Path)
    bundle = commands.add_parser("bundle").add_subparsers(dest="action", required=True)
    verifying = bundle.add_parser("verify", parents=[common])
    verifying.add_argument("source", type=Path)
    return root

def load_document(path):
    try:
        return json.loads(Path(path).read_text(encoding="utf-8"))
    except FileNotFoundError as error:
        raise CoreError("NOT_FOUND", "Input file does not exist") from error
    except (ValueError, UnicodeError) as error:
        raise CoreError("INVALID_SPEC", "Input file must contain JSON") from error

def main(argv=None):
    argv = list(sys.argv[1:] if argv is None else argv)
    json_mode = "--json" in argv
    request_id = str(uuid4())
    workspace_id = None
    try:
        args = parser().parse_args(argv)
        if args.command == "bundle":
            result = verify_bundle(args.source)
            workspace_id = result["workspace_id"]
        else:
            workspace = Workspace.open(args.workspace, create=args.command == "init")
            workspace_id = workspace.workspace_id
            with closing(Store(workspace)) as store:
                if args.command == "init":
                    result = {"data_dir": str(workspace.data)}
                elif args.command == "spec" and args.action == "validate":
                    compiled = compile_spec(load_document(args.source))
                    result = {"valid": True, "sha256": compiled.sha256,
                        "matrix": [asdict(job) for job in compiled.jobs],
                        "connection_checks": "not_performed"}
                elif args.command == "spec" and args.action == "freeze":
                    result = store.freeze(load_document(args.source), args.key)
                elif args.command == "spec":
                    result = store.inspect(args.source)
                elif args.command == "evidence" and args.action == "append":
                    result = store.append(args.frozen_id, args.kind, load_document(args.file))
                elif args.command == "evidence":
                    items = store.list_events(args.frozen_id, args.after)
                    result = {"items": items,
                        "next_cursor": items[-1]["sequence"] if items else args.after}
                else:
                    result = export_bundle(store, args.frozen_id, args.destination)
        print(json.dumps({"schema_version": 1, "workspace_id": workspace_id,
            "request_id": request_id, "result": result}, ensure_ascii=False))
        return 0
    except (CoreError, OSError, sqlite3.Error, ValueError) as exception:
        error = exception if isinstance(exception, CoreError) else CoreError(
            "INTEGRITY_ERROR", "Local storage or input is invalid")
        value = {"schema_version": 1, "workspace_id": workspace_id,
                 "request_id": request_id,
                 "error": {"code": error.code, "message": error.message, "details": {}}}
        print(json.dumps(value, ensure_ascii=False), file=sys.stdout if json_mode else sys.stderr)
        return EXIT_CODES[error.code]

if __name__ == "__main__":
    raise SystemExit(main())
```

- [ ] 运行本任务与核心测试；运行 `uv run agentlab --help`，预期列出 init/spec/evidence/export/bundle。记录 JSON 参数错误的退出码，不在 stdout 加日志。
- [ ] 提交模块与测试，消息 `feat: expose core workflows through CLI`。

## Task 9：完整安装验收与文档

**Files:** Create `tests/core/test_roundtrip.py`, `docs/core-quickstart.md`。Modify `README.md`。

- [ ] 写入下面的跨命令验收，再运行 `uv run pytest tests/core/test_roundtrip.py -q`。若现有实现失败，按原失败修复，不放宽断言：

```python
import json
import subprocess
import sys
from pathlib import Path

def test_complete_offline_evidence_roundtrip(tmp_path):
    workspace = tmp_path / "workspace"
    bundle = tmp_path / "bundle"
    source = Path(__file__).resolve().parents[2] / "experiments" / "temporal-baseline.json"

    def call(*args, expected=0):
        process = subprocess.run([sys.executable, "-m", "agentlab.cli",
            "--workspace", str(workspace), *map(str, args), "--json"],
            capture_output=True, text=True)
        assert process.returncode == expected, process.stderr + process.stdout
        return json.loads(process.stdout)

    call("init")
    assert len(call("spec", "validate", source)["result"]["matrix"]) == 8
    frozen_id = call("spec", "freeze", source, "--key", "roundtrip")["result"]["id"]
    note = tmp_path / "note.json"
    note.write_text(json.dumps({"note": "human review", "test_only": True}))
    call("evidence", "append", frozen_id, "--kind", "note", "--file", note)
    assert len(call("evidence", "list", frozen_id)["result"]["items"]) == 1
    call("export", frozen_id, bundle)
    assert call("bundle", "verify", bundle)["result"]["valid"] is True
    (bundle / "frozen.json").write_text("{}")
    failure = call("bundle", "verify", bundle, expected=6)
    assert failure["error"]["code"] == "INTEGRITY_ERROR"
```

- [ ] `docs/core-quickstart.md` 说明 Python/uv 前提、`uv sync --group dev`、Task 9 的命令次序、工作区与 Git 的区别、FrozenSpec 不等于运行提交，以及导出完整性不等于来源签名。README 链接该文件并准确标明只完成 A。
- [ ] 完整验证：`uv run pytest tests/core -q`、`uv build`、`git diff --check`。用新的 Python 环境安装构建 wheel，运行安装后的 `agentlab --help` 和临时工作区的 init/validate/freeze/export/verify；不只运行源目录模块。记录执行命令与实际结果。
- [ ] 检查 `git status --short`，确保 `.agentlab/`、证据包和凭据不被加入。提交任务文件与验证说明，消息 `docs: verify and document offline core workflow`。

## 覆盖与交接

| 子项目不变量 | 任务 |
| --- | --- |
| I1 规范 JSON 与 hash | 2 |
| I2 有界矩阵 | 2 |
| I3 原子幂等 | 5 |
| I4 不可变冻结读取 | 5 |
| I5 事件与跨进程序号 | 6、9 |
| I6 产物身份与验真 | 4 |
| I7 导出范围与篡改 | 7、9 |
| I8 机器接口和错误 | 8、9 |

本计划未完成的复选框保留为实际执行记录。B 的 spec/plan 根据 A 的已验证接口继续细化；路线图已覆盖 B/C/D 的完整产品要求与明确完成条件，不能把 A 验收作为 P1 插件完成。

执行可选择当前会话按 executing-plans 分批推进，或用户选择后使用 subagent-driven-development。遇到与公共契约不一致的真实行为，先记录差异并修正规格/计划；不在实现中悄悄改变身份、预算、失败分母或重试规则。

## 计划编写阶段的检查记录

已检查规格覆盖、类型/方法命名、17 个 Python 代码块、JSON/TOML 格式及本地文档链接。2026-10-07 将计划中的完整核心模块和测试代码抽取到临时目录离线验证，20 项测试通过；使用 Pydantic 2.12.5、pytest 8.4.2，临时目录已删除。这项检查只验证参考代码的行为，不替代 Task 9 的实际包构建、安装验收与后续 P1-B/C/D 的真实集成检查。仓库本轮变更全部为 Markdown 文档。
