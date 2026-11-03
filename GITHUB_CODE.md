# 知行教室 · 完整源码（按文件复制到 GitHub）

本文档包含项目的全部可复制源码。按顺序在 GitHub 新建同名文件，粘贴对应代码块即可。

> `docs/screenshots/` 是二进制截图，不适合粘贴；可稍后通过 GitHub 的 **Upload files** 上传，README 中的图片就会正常显示。
> `data/` 数据库和缓存不会提交，首次运行会自动生成。

## 在 GitHub 网页中创建

1. 新建一个空仓库。
2. 点击 **Add file → Create new file**。
3. 文件名填写 `app.py`，粘贴下方 `app.py` 代码块，提交。
4. 继续 **Create new file**，文件名填写 `database.py`，粘贴对应代码块。
5. 文件名中带 `/` 会自动创建目录，例如 `static/styles.css`、`static/js/pages/overview.js`。
6. 所有文件创建完成后，在本地或 Codespaces 中运行：

```bash
python app.py
```

然后访问 `http://127.0.0.1:8000`。

## 文件清单

| 文件 | 作用 |
|---|---|
| `app.py` | 启动入口 |
| `database.py` | 建表、密码工具、演示数据 |
| `services.py` | 认证、空闲计算、课程与借用规则 |
| `web.py` | HTTP API、静态资源、会话与 CSRF |
| `static/index.html` | 页面入口 |
| `static/styles.css` | 全部样式 |
| `static/js/core.js` | 工具、API、弹窗 |
| `static/js/app.js` | 登录、路由、应用外壳 |
| `static/js/pages/overview.js` | 教室总览 |
| `static/js/pages/booking.js` | 借用、我的借用、借用管理 |
| `static/js/pages/courses.js` | 课程管理 |
| `static/js/pages/classrooms.js` | 教室管理 |
| `static/js/pages/schedule.js` | 周课表 |
| `tests/test_services.py` | 业务测试 |
| `README.md` | 项目说明 |
| `start.bat` | Windows 启动脚本 |
| `.gitignore` | Git 忽略规则 |

---


## `app.py`

`````python
"""知行教室 · 教室课程管理系统

启动方式：
    python app.py
    python app.py --port 8000 --host 127.0.0.1
"""

from __future__ import annotations

import argparse
import sys
from http.server import ThreadingHTTPServer

import database
from web import ApiHandler


class ClassroomServer(ThreadingHTTPServer):
    daemon_threads = True
    allow_reuse_address = True

    def handle_error(self, request, client_address) -> None:
        exc = sys.exc_info()[1]
        if isinstance(exc, (ConnectionResetError, BrokenPipeError)):
            return
        super().handle_error(request, client_address)


def main() -> int:
    parser = argparse.ArgumentParser(description="知行教室 · 教室课程管理系统")
    parser.add_argument("--host", default="127.0.0.1", help="监听地址，默认 127.0.0.1")
    parser.add_argument("--port", type=int, default=8000, help="监听端口，默认 8000")
    args = parser.parse_args()

    database.init_db()

    try:
        httpd = ClassroomServer((args.host, args.port), ApiHandler)
    except OSError as exc:
        print(f"[ERROR] Cannot start server on {args.host}:{args.port}: {exc}", file=sys.stderr)
        print("[HINT] The port may be in use. Try: python app.py --port 8001", file=sys.stderr)
        return 1

    print("=" * 62)
    print("  Zhixing Classroom - Course & Classroom Management")
    print(f"  Local:   http://{args.host}:{args.port}")
    print(f"  Data:    {database.get_db_path()}")
    print("  Demo:    student / teacher / admin    password: 123456")
    print("  Stop:    Ctrl+C")
    print("=" * 62)

    try:
        httpd.serve_forever()
    except KeyboardInterrupt:
        print("\nServer stopped.")
    finally:
        httpd.server_close()
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
`````


## `database.py`

`````python
"""SQLite 数据层：建表、演示数据与密码工具。"""

from __future__ import annotations

import hashlib
import os
import secrets
import sqlite3
from datetime import date, timedelta
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent

_db_path: Path | None = None
_secret: bytes | None = None


def set_db_path(path: str | Path) -> None:
    """供测试或多实例部署时覆盖数据库位置。"""
    global _db_path
    _db_path = Path(path).resolve()


def get_db_path() -> Path:
    if _db_path is not None:
        return _db_path
    env_path = os.environ.get("CLASSROOM_DB")
    if env_path:
        return Path(env_path).expanduser().resolve()
    return BASE_DIR / "data" / "classroom.db"


def connect() -> sqlite3.Connection:
    path = get_db_path()
    path.parent.mkdir(parents=True, exist_ok=True)
    conn = sqlite3.connect(path, timeout=10)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    conn.execute("PRAGMA journal_mode = WAL")
    return conn


def get_secret() -> bytes:
    """用于签名登录 Cookie 与 CSRF Token 的持久化密钥。"""
    global _secret
    if _secret is not None:
        return _secret

    env_secret = os.environ.get("CLASSROOM_SECRET")
    if env_secret:
        _secret = env_secret.encode("utf-8")
        return _secret

    path = get_db_path().parent / "session.key"
    path.parent.mkdir(parents=True, exist_ok=True)
    if path.exists():
        _secret = path.read_bytes()
    else:
        _secret = secrets.token_bytes(32)
        path.write_bytes(_secret)
    return _secret


def hash_password(password: str, salt_hex: str | None = None) -> tuple[str, str]:
    salt = bytes.fromhex(salt_hex) if salt_hex else secrets.token_bytes(16)
    digest = hashlib.pbkdf2_hmac("sha256", password.encode("utf-8"), salt, 180_000)
    return salt.hex(), digest.hex()


def verify_password(password: str, salt_hex: str, hash_hex: str) -> bool:
    salt = bytes.fromhex(salt_hex)
    digest = hashlib.pbkdf2_hmac("sha256", password.encode("utf-8"), salt, 180_000)
    return secrets.compare_digest(digest.hex(), hash_hex)


SCHEMA = """
CREATE TABLE IF NOT EXISTS users (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    username      TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    salt          TEXT NOT NULL,
    role          TEXT NOT NULL CHECK (role IN ('student', 'teacher', 'admin')),
    name          TEXT NOT NULL,
    identifier    TEXT,
    class_name    TEXT,
    created_at    TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS classrooms (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    name       TEXT NOT NULL UNIQUE,
    building   TEXT NOT NULL,
    room_no    TEXT NOT NULL,
    capacity   INTEGER NOT NULL CHECK (capacity > 0),
    room_type  TEXT NOT NULL,
    equipment  TEXT NOT NULL DEFAULT '',
    status     TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'maintenance')),
    created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (building, room_no)
);

CREATE TABLE IF NOT EXISTS courses (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    code         TEXT NOT NULL,
    name         TEXT NOT NULL,
    teacher_id   INTEGER NOT NULL REFERENCES users(id),
    classroom_id INTEGER NOT NULL REFERENCES classrooms(id),
    class_name   TEXT NOT NULL,
    weekday      INTEGER NOT NULL CHECK (weekday BETWEEN 1 AND 7),
    start_time   TEXT NOT NULL,
    end_time     TEXT NOT NULL,
    weeks_start  INTEGER NOT NULL CHECK (weeks_start BETWEEN 1 AND 30),
    weeks_end    INTEGER NOT NULL CHECK (weeks_end BETWEEN 1 AND 30),
    created_at   TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS bookings (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    classroom_id INTEGER NOT NULL REFERENCES classrooms(id),
    teacher_id   INTEGER NOT NULL REFERENCES users(id),
    date         TEXT NOT NULL,
    start_time   TEXT NOT NULL,
    end_time     TEXT NOT NULL,
    purpose      TEXT NOT NULL,
    status       TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'cancelled')),
    created_at   TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
    cancelled_at TEXT
);

CREATE TABLE IF NOT EXISTS settings (
    key   TEXT PRIMARY KEY,
    value TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_courses_weekday ON courses(weekday, teacher_id, classroom_id);
CREATE INDEX IF NOT EXISTS idx_bookings_date ON bookings(date, classroom_id, teacher_id);
CREATE INDEX IF NOT EXISTS idx_bookings_teacher ON bookings(teacher_id, date);
"""


def default_semester_start(today: date) -> date:
    if today.month >= 8:
        return date(today.year, 9, 1)
    if today.month <= 1:
        return date(today.year - 1, 9, 1)
    return date(today.year, 2, 20)


def semester_label(start: date) -> str:
    if start.month >= 8:
        return f"{start.year}-{start.year + 1} 学年 · 秋季学期"
    return f"{start.year - 1}-{start.year} 学年 · 春季学期"


def init_db() -> None:
    conn = connect()
    try:
        conn.executescript(SCHEMA)
        conn.commit()
        seed_demo_data(conn)
    finally:
        conn.close()


def _insert_user(
    conn: sqlite3.Connection,
    username: str,
    password: str,
    role: str,
    name: str,
    identifier: str,
    class_name: str | None = None,
) -> int:
    salt, digest = hash_password(password)
    cur = conn.execute(
        "INSERT INTO users (username, password_hash, salt, role, name, identifier, class_name) "
        "VALUES (?, ?, ?, ?, ?, ?, ?)",
        (username, digest, salt, role, name, identifier, class_name),
    )
    return int(cur.lastrowid)


def seed_demo_data(conn: sqlite3.Connection) -> None:
    if conn.execute("SELECT COUNT(*) FROM users").fetchone()[0] > 0:
        return

    today = date.today()
    semester_start = default_semester_start(today)
    conn.executemany(
        "INSERT INTO settings (key, value) VALUES (?, ?)",
        [
            ("semester_start", semester_start.isoformat()),
            ("semester_name", semester_label(semester_start)),
        ],
    )

    _insert_user(conn, "admin", "123456", "admin", "周明", "A0001")
    teachers: dict[str, int] = {}
    for username, name, identifier in [
        ("teacher", "张伟", "T1001"),
        ("lina", "李娜", "T1002"),
        ("wangqiang", "王强", "T1003"),
    ]:
        teachers[username] = _insert_user(
            conn, username, "123456", "teacher", name, identifier
        )

    _insert_user(conn, "student", "123456", "student", "陈晨", "2023010101", "计算机2301")
    _insert_user(conn, "liuyang", "123456", "student", "刘洋", "2023010102", "计算机2301")
    _insert_user(conn, "zhaomin", "123456", "student", "赵敏", "2023020201", "软件2302")

    classrooms = [
        ("综合楼 A101", "综合楼", "A101", 120, "多媒体教室", "投影,音响,空调"),
        ("综合楼 A102", "综合楼", "A102", 80, "多媒体教室", "投影,空调"),
        ("综合楼 A201", "综合楼", "A201", 160, "阶梯教室", "双投影,音响,录播"),
        ("综合楼 A202", "综合楼", "A202", 60, "普通教室", "白板,空调"),
        ("教学楼 B101", "教学楼", "B101", 100, "多媒体教室", "投影,音响"),
        ("教学楼 B102", "教学楼", "B102", 60, "普通教室", "白板,空调"),
        ("教学楼 B201", "教学楼", "B201", 48, "研讨室", "圆桌,投屏,白板"),
        ("教学楼 B202", "教学楼", "B202", 48, "研讨室", "圆桌,投屏"),
        ("信息楼 C301", "信息楼", "C301", 60, "机房", "60台电脑,投影"),
        ("信息楼 C302", "信息楼", "C302", 60, "机房", "60台电脑,投影"),
        ("信息楼 C401", "信息楼", "C401", 40, "智慧教室", "录播,互动大屏,空调"),
        ("图书馆 D201", "图书馆", "D201", 30, "研讨室", "圆桌,投屏,白板"),
    ]
    conn.executemany(
        "INSERT INTO classrooms (name, building, room_no, capacity, room_type, equipment) "
        "VALUES (?, ?, ?, ?, ?, ?)",
        classrooms,
    )
    room_ids = {
        row["name"]: row["id"]
        for row in conn.execute("SELECT id, name FROM classrooms").fetchall()
    }

    course_rows = [
        ("MATH101", "高等数学", "wangqiang", "教学楼 B102", "计算机2301", 1, "08:00", "09:35"),
        ("CS201", "数据结构", "teacher", "综合楼 A101", "计算机2301", 1, "10:05", "11:40"),
        ("PHY102", "大学物理", "lina", "教学楼 B101", "软件2302", 1, "14:00", "15:35"),
        ("CS101", "程序设计基础", "teacher", "信息楼 C301", "计算机2301", 1, "19:00", "20:35"),
        ("CS202", "操作系统", "lina", "综合楼 A102", "软件2302", 2, "08:00", "09:35"),
        ("CS201", "数据结构", "teacher", "综合楼 A101", "计算机2301", 2, "10:05", "11:40"),
        ("ENG101", "大学英语", "wangqiang", "教学楼 B101", "计算机2301", 2, "14:00", "15:35"),
        ("AI301", "人工智能导论", "teacher", "信息楼 C401", "计算机2301", 2, "16:05", "17:40"),
        ("CS203", "计算机网络", "lina", "教学楼 B101", "软件2302", 3, "08:00", "09:35"),
        ("MATH201", "线性代数", "wangqiang", "综合楼 A201", "计算机2301", 3, "10:05", "11:40"),
        ("CS204", "数据库原理", "teacher", "信息楼 C302", "计算机2301", 3, "14:00", "15:35"),
        ("CS205", "创新实践", "wangqiang", "教学楼 B201", "软件2302", 3, "19:00", "20:35"),
        ("SE301", "软件工程", "lina", "综合楼 A102", "软件2302", 4, "08:00", "09:35"),
        ("ENG102", "大学英语", "wangqiang", "教学楼 B102", "计算机2301", 4, "10:05", "11:40"),
        ("AI302", "机器学习", "teacher", "信息楼 C401", "计算机2301", 4, "14:00", "15:35"),
        ("CS206", "编译原理", "lina", "综合楼 A101", "软件2302", 5, "08:00", "09:35"),
        ("MATH202", "概率论与数理统计", "wangqiang", "综合楼 A201", "计算机2301", 5, "10:05", "11:40"),
        ("CS207", "课程设计", "teacher", "信息楼 C301", "计算机2301", 5, "14:00", "15:35"),
    ]
    for code, name, teacher, room, class_name, weekday, start, end in course_rows:
        conn.execute(
            "INSERT INTO courses "
            "(code, name, teacher_id, classroom_id, class_name, weekday, start_time, end_time, weeks_start, weeks_end) "
            "VALUES (?, ?, ?, ?, ?, ?, ?, ?, 1, 18)",
            (
                code,
                name,
                teachers[teacher],
                room_ids[room],
                class_name,
                weekday,
                start,
                end,
            ),
        )

    booking_rows = [
        ("综合楼 A201", "teacher", today, "12:10", "13:10", "课程组教学研讨"),
        ("教学楼 B201", "lina", today, "12:10", "13:10", "毕业设计答疑"),
        ("图书馆 D201", "wangqiang", today, "18:30", "20:00", "学科竞赛辅导"),
        (
            "综合楼 A101",
            "lina",
            today + timedelta(days=1),
            "19:00",
            "20:30",
            "算法兴趣小组集训",
        ),
    ]
    for room, teacher, booking_date, start, end, purpose in booking_rows:
        conn.execute(
            "INSERT INTO bookings "
            "(classroom_id, teacher_id, date, start_time, end_time, purpose, status) "
            "VALUES (?, ?, ?, ?, ?, ?, 'active')",
            (
                room_ids[room],
                teachers[teacher],
                booking_date.isoformat(),
                start,
                end,
                purpose,
            ),
        )

    conn.commit()
`````


## `services.py`

`````python
"""领域服务：认证、教室空闲计算、课程与借用业务规则。"""

from __future__ import annotations

import hashlib
import hmac
import re
import sqlite3
import time
from datetime import date, datetime, timedelta
from typing import Any

import database

WEEKDAY_LABELS = ["周一", "周二", "周三", "周四", "周五", "周六", "周日"]
WINDOW_START = 8 * 60
WINDOW_END = 22 * 60
BOOKING_DAYS_AHEAD = 90
SESSION_TTL = 7 * 24 * 3600
TIME_RE = re.compile(r"^([01]?\d|2[0-3]):([0-5]\d)$")
BUILDING_ORDER = ["综合楼", "教学楼", "信息楼", "图书馆"]
ROOM_ORDER_SQL = (
    "CASE building "
    "WHEN '综合楼' THEN 1 WHEN '教学楼' THEN 2 WHEN '信息楼' THEN 3 "
    "WHEN '图书馆' THEN 4 ELSE 5 END, room_no"
)


class ApiError(Exception):
    def __init__(self, message: str, status: int = 400, details: dict | None = None):
        super().__init__(message)
        self.message = message
        self.status = status
        self.details = details or {}


# ---------------------------------------------------------------------------
# 基础工具
# ---------------------------------------------------------------------------


def now_local() -> datetime:
    return datetime.now()


def parse_date(value: str, field: str = "日期") -> date:
    try:
        return date.fromisoformat(value)
    except (TypeError, ValueError):
        raise ApiError(f"{field}格式不正确，应为 YYYY-MM-DD")


def normalize_time(value: str, field: str = "时间") -> str:
    if not isinstance(value, str):
        raise ApiError(f"{field}格式不正确")
    value = value.strip()
    match = TIME_RE.match(value)
    if not match:
        raise ApiError(f"{field}格式不正确，应为 HH:MM")
    return f"{int(match.group(1)):02d}:{match.group(2)}"


def to_minutes(value: str) -> int:
    hour, minute = value.split(":")
    return int(hour) * 60 + int(minute)


def from_minutes(value: int) -> str:
    return f"{value // 60:02d}:{value % 60:02d}"


def overlap(start_a: int, end_a: int, start_b: int, end_b: int) -> bool:
    return start_a < end_b and end_a > start_b


def public_user(row: sqlite3.Row | dict) -> dict:
    return {
        "id": row["id"],
        "username": row["username"],
        "name": row["name"],
        "role": row["role"],
        "identifier": row["identifier"],
        "class_name": row["class_name"],
    }


def require_role(user: dict, *roles: str) -> None:
    if user["role"] not in roles:
        raise ApiError("你没有权限执行此操作", 403)


def _clean_text(value: Any, field: str, max_length: int = 80) -> str:
    if value is None:
        raise ApiError(f"请填写{field}")
    text = str(value).strip()
    if not text:
        raise ApiError(f"请填写{field}")
    if len(text) > max_length:
        raise ApiError(f"{field}不能超过 {max_length} 个字符")
    return text


def _int_value(value: Any, field: str, minimum: int, maximum: int) -> int:
    try:
        number = int(value)
    except (TypeError, ValueError):
        raise ApiError(f"{field}格式不正确")
    if number < minimum or number > maximum:
        raise ApiError(f"{field}应在 {minimum} 到 {maximum} 之间")
    return number


# ---------------------------------------------------------------------------
# 会话与认证
# ---------------------------------------------------------------------------


def authenticate(conn: sqlite3.Connection, username: str, password: str) -> dict:
    username = (username or "").strip()
    if not username or not password:
        raise ApiError("请输入用户名和密码")
    row = conn.execute("SELECT * FROM users WHERE username = ?", (username,)).fetchone()
    if not row or not database.verify_password(password, row["salt"], row["password_hash"]):
        raise ApiError("用户名或密码不正确", 401)
    return public_user(row)


def create_session(user_id: int) -> str:
    expires = int(time.time()) + SESSION_TTL
    payload = f"{user_id}.{expires}"
    signature = hmac.new(
        database.get_secret(), payload.encode("utf-8"), hashlib.sha256
    ).hexdigest()
    return f"{payload}.{signature}"


def read_session(conn: sqlite3.Connection, token: str | None) -> tuple[dict | None, str | None]:
    if not token:
        return None, None
    parts = token.split(".")
    if len(parts) != 3:
        return None, None
    user_id, expires, signature = parts
    payload = f"{user_id}.{expires}"
    expected = hmac.new(
        database.get_secret(), payload.encode("utf-8"), hashlib.sha256
    ).hexdigest()
    if not hmac.compare_digest(signature, expected):
        return None, None
    try:
        if int(expires) < int(time.time()):
            return None, None
        uid = int(user_id)
    except ValueError:
        return None, None
    row = conn.execute("SELECT * FROM users WHERE id = ?", (uid,)).fetchone()
    if not row:
        return None, None
    return public_user(row), token


def csrf_for_session(token: str) -> str:
    return hmac.new(
        database.get_secret(), ("csrf:" + token).encode("utf-8"), hashlib.sha256
    ).hexdigest()[:40]


# ---------------------------------------------------------------------------
# 学期、元数据
# ---------------------------------------------------------------------------


def semester_info(conn: sqlite3.Connection, on_date: date | None = None) -> dict:
    rows = {
        row["key"]: row["value"]
        for row in conn.execute(
            "SELECT key, value FROM settings WHERE key IN ('semester_start', 'semester_name')"
        ).fetchall()
    }
    try:
        start = date.fromisoformat(rows.get("semester_start", ""))
    except ValueError:
        start = database.default_semester_start(date.today())
    monday = start - timedelta(days=start.isoweekday() - 1)
    target = on_date or date.today()
    week = (target - monday).days // 7 + 1 if target >= monday else 0
    return {
        "start": monday.isoformat(),
        "name": rows.get("semester_name") or database.semester_label(monday),
        "week": week,
        "has_courses": 1 <= week <= 30,
    }


def get_meta(conn: sqlite3.Connection) -> dict:
    teachers = [
        {"id": row["id"], "name": row["name"], "identifier": row["identifier"]}
        for row in conn.execute(
            "SELECT id, name, identifier FROM users WHERE role = 'teacher' ORDER BY id"
        ).fetchall()
    ]
    classrooms = [
        _classroom_dict(row)
        for row in conn.execute(
            f"SELECT * FROM classrooms ORDER BY {ROOM_ORDER_SQL}"
        ).fetchall()
    ]
    buildings = list(dict.fromkeys(row["building"] for row in classrooms))
    room_types = sorted({row["room_type"] for row in classrooms})
    now = now_local()
    return {
        "teachers": teachers,
        "classrooms": classrooms,
        "buildings": buildings,
        "room_types": room_types,
        "semester": semester_info(conn),
        "today": now.date().isoformat(),
        "server_time": now.strftime("%Y-%m-%d %H:%M:%S"),
        "window": {"start": "08:00", "end": "22:00"},
    }


def _classroom_dict(row: sqlite3.Row) -> dict:
    return {
        "id": row["id"],
        "name": row["name"],
        "building": row["building"],
        "room_no": row["room_no"],
        "capacity": row["capacity"],
        "room_type": row["room_type"],
        "equipment": [item for item in (row["equipment"] or "").split(",") if item],
        "status": row["status"],
    }


# ---------------------------------------------------------------------------
# 空闲计算
# ---------------------------------------------------------------------------


def _course_intervals(
    conn: sqlite3.Connection, target: date, week: int, classroom_id: int | None = None
) -> list[dict]:
    if week < 1:
        return []
    sql = (
        "SELECT c.*, u.name AS teacher_name FROM courses c "
        "JOIN users u ON u.id = c.teacher_id "
        "WHERE c.weekday = ? AND c.weeks_start <= ? AND c.weeks_end >= ?"
    )
    params: list[Any] = [target.isoweekday(), week, week]
    if classroom_id is not None:
        sql += " AND c.classroom_id = ?"
        params.append(classroom_id)
    return [
        {
            "kind": "course",
            "title": row["name"],
            "subtitle": f"{row['teacher_name']} · {row['class_name']}",
            "teacher": row["teacher_name"],
            "start": row["start_time"],
            "end": row["end_time"],
            "start_min": to_minutes(row["start_time"]),
            "end_min": to_minutes(row["end_time"]),
            "course_id": row["id"],
        }
        for row in conn.execute(sql, params).fetchall()
    ]


def _booking_intervals(
    conn: sqlite3.Connection, target: date, classroom_id: int | None = None
) -> list[dict]:
    sql = (
        "SELECT b.*, u.name AS teacher_name FROM bookings b "
        "JOIN users u ON u.id = b.teacher_id "
        "WHERE b.date = ? AND b.status = 'active'"
    )
    params: list[Any] = [target.isoformat()]
    if classroom_id is not None:
        sql += " AND b.classroom_id = ?"
        params.append(classroom_id)
    return [
        {
            "kind": "booking",
            "title": row["purpose"],
            "subtitle": f"教师借用 · {row['teacher_name']}",
            "teacher": row["teacher_name"],
            "start": row["start_time"],
            "end": row["end_time"],
            "start_min": to_minutes(row["start_time"]),
            "end_min": to_minutes(row["end_time"]),
            "booking_id": row["id"],
        }
        for row in conn.execute(sql, params).fetchall()
    ]


def _intervals_for_date(conn: sqlite3.Connection, target: date) -> dict[int, list[dict]]:
    week = semester_info(conn, target)["week"]
    grouped: dict[int, list[dict]] = {}
    if week >= 1:
        for row in conn.execute(
            "SELECT c.*, u.name AS teacher_name FROM courses c "
            "JOIN users u ON u.id = c.teacher_id "
            "WHERE c.weekday = ? AND c.weeks_start <= ? AND c.weeks_end >= ?",
            (target.isoweekday(), week, week),
        ).fetchall():
            grouped.setdefault(row["classroom_id"], []).append(
                {
                    "kind": "course",
                    "title": row["name"],
                    "subtitle": f"{row['teacher_name']} · {row['class_name']}",
                    "teacher": row["teacher_name"],
                    "start": row["start_time"],
                    "end": row["end_time"],
                    "start_min": to_minutes(row["start_time"]),
                    "end_min": to_minutes(row["end_time"]),
                    "course_id": row["id"],
                }
            )
    for row in conn.execute(
        "SELECT b.*, u.name AS teacher_name FROM bookings b "
        "JOIN users u ON u.id = b.teacher_id "
        "WHERE b.date = ? AND b.status = 'active'",
        (target.isoformat(),),
    ).fetchall():
        grouped.setdefault(row["classroom_id"], []).append(
            {
                "kind": "booking",
                "title": row["purpose"],
                "subtitle": f"教师借用 · {row['teacher_name']}",
                "teacher": row["teacher_name"],
                "start": row["start_time"],
                "end": row["end_time"],
                "start_min": to_minutes(row["start_time"]),
                "end_min": to_minutes(row["end_time"]),
                "booking_id": row["id"],
            }
        )
    for items in grouped.values():
        items.sort(key=lambda item: item["start_min"])
    return grouped


def _teacher_intervals_for_date(
    conn: sqlite3.Connection, teacher_id: int, target: date
) -> list[dict]:
    week = semester_info(conn, target)["week"]
    intervals: list[dict] = []
    if week >= 1:
        for row in conn.execute(
            "SELECT c.*, r.name AS classroom_name FROM courses c "
            "JOIN classrooms r ON r.id = c.classroom_id "
            "WHERE c.teacher_id = ? AND c.weekday = ? AND c.weeks_start <= ? AND c.weeks_end >= ?",
            (teacher_id, target.isoweekday(), week, week),
        ).fetchall():
            intervals.append(
                {
                    "kind": "course",
                    "title": row["name"],
                    "subtitle": f"{row['classroom_name']} · {row['class_name']}",
                    "start": row["start_time"],
                    "end": row["end_time"],
                    "start_min": to_minutes(row["start_time"]),
                    "end_min": to_minutes(row["end_time"]),
                }
            )
    for row in conn.execute(
        "SELECT b.*, r.name AS classroom_name FROM bookings b "
        "JOIN classrooms r ON r.id = b.classroom_id "
        "WHERE b.teacher_id = ? AND b.date = ? AND b.status = 'active'",
        (teacher_id, target.isoformat()),
    ).fetchall():
        intervals.append(
            {
                "kind": "booking",
                "title": row["purpose"],
                "subtitle": f"已借用 {row['classroom_name']}",
                "start": row["start_time"],
                "end": row["end_time"],
                "start_min": to_minutes(row["start_time"]),
                "end_min": to_minutes(row["end_time"]),
            }
        )
    intervals.sort(key=lambda item: item["start_min"])
    return intervals


def get_overview(
    conn: sqlite3.Connection,
    date_value: str | None,
    at_value: str | None,
    user: dict | None = None,
) -> dict:
    target = parse_date(date_value, "查询日期") if date_value else date.today()
    if at_value:
        at = normalize_time(at_value, "查询时间")
    else:
        at = now_local().strftime("%H:%M")
    at_min = to_minutes(at)
    outside_window = at_min < WINDOW_START or at_min >= WINDOW_END
    week = semester_info(conn, target)["week"]
    grouped = _intervals_for_date(conn, target)

    classrooms = []
    for row in conn.execute(
        f"SELECT * FROM classrooms ORDER BY {ROOM_ORDER_SQL}"
    ).fetchall():
        room = _classroom_dict(row)
        intervals = grouped.get(row["id"], [])
        current = next(
            (item for item in intervals if item["start_min"] <= at_min < item["end_min"]), None
        )
        upcoming = next((item for item in intervals if item["start_min"] > at_min), None)
        if row["status"] == "maintenance":
            state = "maintenance"
        elif current:
            state = "busy"
        else:
            state = "free"
        room.update(
            {
                "intervals": intervals,
                "state": state,
                "current": current,
                "upcoming": upcoming,
                "free_until": (
                    from_minutes(upcoming["start_min"])
                    if state == "free" and upcoming
                    else ("22:00" if state == "free" else None)
                ),
            }
        )
        classrooms.append(room)

    active_rooms = [room for room in classrooms if room["status"] == "active"]
    stats = {
        "total": len(active_rooms),
        "free": sum(1 for room in active_rooms if room["state"] == "free"),
        "busy": sum(1 for room in active_rooms if room["state"] == "busy"),
        "maintenance": sum(1 for room in classrooms if room["status"] == "maintenance"),
        "bookings_today": conn.execute(
            "SELECT COUNT(*) FROM bookings WHERE date = ? AND status = 'active'",
            (date.today().isoformat(),),
        ).fetchone()[0],
    }

    now = now_local()
    semester = semester_info(conn, target)
    return {
        "date": target.isoformat(),
        "at": at,
        "week": week,
        "semester": semester,
        "outside_window": outside_window,
        "window": {"start": "08:00", "end": "22:00"},
        "today": date.today().isoformat(),
        "server_time": now.strftime("%Y-%m-%d %H:%M:%S"),
        "classrooms": classrooms,
        "stats": stats,
        "teacher_intervals": (
            _teacher_intervals_for_date(conn, user["id"], target)
            if user and user["role"] == "teacher"
            else []
        ),
    }


# ---------------------------------------------------------------------------
# 课程
# ---------------------------------------------------------------------------


def _course_dict(row: sqlite3.Row) -> dict:
    return {
        "id": row["id"],
        "code": row["code"],
        "name": row["name"],
        "teacher_id": row["teacher_id"],
        "teacher_name": row["teacher_name"],
        "classroom_id": row["classroom_id"],
        "classroom_name": row["classroom_name"],
        "class_name": row["class_name"],
        "weekday": row["weekday"],
        "weekday_label": WEEKDAY_LABELS[row["weekday"] - 1],
        "start_time": row["start_time"],
        "end_time": row["end_time"],
        "weeks_start": row["weeks_start"],
        "weeks_end": row["weeks_end"],
        "weeks_label": f"{row['weeks_start']}-{row['weeks_end']} 周",
    }


COURSE_SELECT = (
    "SELECT c.*, u.name AS teacher_name, r.name AS classroom_name "
    "FROM courses c JOIN users u ON u.id = c.teacher_id "
    "JOIN classrooms r ON r.id = c.classroom_id "
)


def list_courses(conn: sqlite3.Connection, user: dict) -> list[dict]:
    sql = COURSE_SELECT
    params: list[Any] = []
    if user["role"] == "teacher":
        sql += "WHERE c.teacher_id = ? "
        params.append(user["id"])
    elif user["role"] == "student":
        sql += "WHERE c.class_name = ? "
        params.append(user["class_name"] or "")
    sql += "ORDER BY c.weekday, c.start_time, c.name"
    return [_course_dict(row) for row in conn.execute(sql, params).fetchall()]


def _course_dates(conn: sqlite3.Connection, weekday: int, weeks_start: int, weeks_end: int) -> list[str]:
    semester = semester_info(conn)
    monday = date.fromisoformat(semester["start"])
    dates = []
    for week in range(weeks_start, weeks_end + 1):
        d = monday + timedelta(days=(week - 1) * 7 + weekday - 1)
        dates.append(d.isoformat())
    return dates


def _course_booking_conflicts(
    conn: sqlite3.Connection,
    weekday: int,
    weeks_start: int,
    weeks_end: int,
    start_min: int,
    end_min: int,
    classroom_id: int,
    teacher_id: int,
    exclude_id: int | None = None,
) -> str | None:
    dates = _course_dates(conn, weekday, weeks_start, weeks_end)
    if not dates:
        return None
    placeholders = ",".join("?" for _ in dates)
    rows = conn.execute(
        f"SELECT b.*, u.name AS teacher_name, r.name AS classroom_name FROM bookings b "
        f"JOIN users u ON u.id = b.teacher_id JOIN classrooms r ON r.id = b.classroom_id "
        f"WHERE b.status = 'active' AND b.date IN ({placeholders}) "
        f"AND (b.classroom_id = ? OR b.teacher_id = ?)",
        (*dates, classroom_id, teacher_id),
    ).fetchall()
    for row in rows:
        if not overlap(start_min, end_min, to_minutes(row["start_time"]), to_minutes(row["end_time"])):
            continue
        if row["classroom_id"] == classroom_id:
            return (
                f"{row['date']} {row['start_time']}-{row['end_time']}，"
                f"{row['classroom_name']} 已被 {row['teacher_name']} 借用"
            )
        return (
            f"{row['date']} {row['start_time']}-{row['end_time']}，"
            f"{row['teacher_name']} 老师已有教室借用"
        )
    return None


def _validate_course_payload(
    conn: sqlite3.Connection, payload: dict, exclude_id: int | None = None
) -> dict:
    code = _clean_text(payload.get("code"), "课程代码", 24)
    name = _clean_text(payload.get("name"), "课程名称", 60)
    class_name = _clean_text(payload.get("class_name"), "授课班级", 40)
    teacher_id = _int_value(payload.get("teacher_id"), "授课教师", 1, 10**9)
    classroom_id = _int_value(payload.get("classroom_id"), "教室", 1, 10**9)
    weekday = _int_value(payload.get("weekday"), "星期", 1, 7)
    start = normalize_time(payload.get("start_time", ""), "开始时间")
    end = normalize_time(payload.get("end_time", ""), "结束时间")
    weeks_start = _int_value(payload.get("weeks_start"), "起始周", 1, 30)
    weeks_end = _int_value(payload.get("weeks_end"), "结束周", 1, 30)

    start_min, end_min = to_minutes(start), to_minutes(end)
    if end_min <= start_min:
        raise ApiError("结束时间必须晚于开始时间")
    if start_min < WINDOW_START or end_min > WINDOW_END:
        raise ApiError("课程时间需在 08:00-22:00 之间")
    if weeks_end < weeks_start:
        raise ApiError("结束周不能早于起始周")

    teacher = conn.execute(
        "SELECT id, name, role FROM users WHERE id = ?", (teacher_id,)
    ).fetchone()
    if not teacher or teacher["role"] != "teacher":
        raise ApiError("授课教师不存在")
    room = conn.execute("SELECT * FROM classrooms WHERE id = ?", (classroom_id,)).fetchone()
    if not room:
        raise ApiError("教室不存在")

    rows = conn.execute(
        "SELECT c.*, u.name AS teacher_name, r.name AS classroom_name FROM courses c "
        "JOIN users u ON u.id = c.teacher_id JOIN classrooms r ON r.id = c.classroom_id "
        "WHERE c.weekday = ? AND c.id != ?",
        (weekday, exclude_id or 0),
    ).fetchall()
    for row in rows:
        if row["weeks_end"] < weeks_start or row["weeks_start"] > weeks_end:
            continue
        if not overlap(start_min, end_min, to_minutes(row["start_time"]), to_minutes(row["end_time"])):
            continue
        if row["classroom_id"] == classroom_id:
            raise ApiError(
                f"教室冲突：{row['classroom_name']} 在{WEEKDAY_LABELS[weekday - 1]} "
                f"{row['start_time']}-{row['end_time']} 已安排《{row['name']}》"
            )
        if row["teacher_id"] == teacher_id:
            raise ApiError(
                f"教师冲突：{row['teacher_name']} 在{WEEKDAY_LABELS[weekday - 1]} "
                f"{row['start_time']}-{row['end_time']} 已安排《{row['name']}》"
            )
        if (row["class_name"] or "").strip().lower() == class_name.lower():
            raise ApiError(
                f"班级冲突：{class_name} 在{WEEKDAY_LABELS[weekday - 1]} "
                f"{row['start_time']}-{row['end_time']} 已安排《{row['name']}》"
            )

    booking_conflict = _course_booking_conflicts(
        conn,
        weekday,
        weeks_start,
        weeks_end,
        start_min,
        end_min,
        classroom_id,
        teacher_id,
        exclude_id,
    )
    if booking_conflict:
        raise ApiError(f"借用冲突：{booking_conflict}")

    return {
        "code": code,
        "name": name,
        "teacher_id": teacher_id,
        "classroom_id": classroom_id,
        "class_name": class_name,
        "weekday": weekday,
        "start_time": start,
        "end_time": end,
        "weeks_start": weeks_start,
        "weeks_end": weeks_end,
    }


def create_course(conn: sqlite3.Connection, user: dict, payload: dict) -> dict:
    require_role(user, "admin")
    values = _validate_course_payload(conn, payload)
    cur = conn.execute(
        "INSERT INTO courses "
        "(code, name, teacher_id, classroom_id, class_name, weekday, start_time, end_time, weeks_start, weeks_end) "
        "VALUES (:code, :name, :teacher_id, :classroom_id, :class_name, :weekday, :start_time, :end_time, :weeks_start, :weeks_end)",
        values,
    )
    conn.commit()
    row = conn.execute(COURSE_SELECT + "WHERE c.id = ?", (cur.lastrowid,)).fetchone()
    return _course_dict(row)


def update_course(conn: sqlite3.Connection, user: dict, course_id: int, payload: dict) -> dict:
    require_role(user, "admin")
    existing = conn.execute("SELECT id FROM courses WHERE id = ?", (course_id,)).fetchone()
    if not existing:
        raise ApiError("课程不存在", 404)
    values = _validate_course_payload(conn, payload, exclude_id=course_id)
    conn.execute(
        "UPDATE courses SET code = :code, name = :name, teacher_id = :teacher_id, "
        "classroom_id = :classroom_id, class_name = :class_name, weekday = :weekday, "
        "start_time = :start_time, end_time = :end_time, weeks_start = :weeks_start, "
        "weeks_end = :weeks_end WHERE id = :id",
        {**values, "id": course_id},
    )
    conn.commit()
    row = conn.execute(COURSE_SELECT + "WHERE c.id = ?", (course_id,)).fetchone()
    return _course_dict(row)


def delete_course(conn: sqlite3.Connection, user: dict, course_id: int) -> None:
    require_role(user, "admin")
    existing = conn.execute("SELECT id FROM courses WHERE id = ?", (course_id,)).fetchone()
    if not existing:
        raise ApiError("课程不存在", 404)
    conn.execute("DELETE FROM courses WHERE id = ?", (course_id,))
    conn.commit()


def get_schedule(conn: sqlite3.Connection, user: dict) -> dict:
    courses = list_courses(conn, user)
    semester = semester_info(conn)
    week = semester["week"]
    if 1 <= week <= 30:
        courses = [
            course for course in courses if course["weeks_start"] <= week <= course["weeks_end"]
        ]
    else:
        courses = []
    monday = date.fromisoformat(semester["start"]) + timedelta(days=(max(week, 1) - 1) * 7)
    today = date.today()
    days = []
    for index in range(1, 8):
        d = monday + timedelta(days=index - 1)
        days.append(
            {
                "weekday": index,
                "label": WEEKDAY_LABELS[index - 1],
                "date": d.isoformat(),
                "day": d.day,
                "month": d.month,
                "is_today": d == today,
            }
        )
    today_courses = [c for c in courses if c["weekday"] == today.isoweekday()]
    total_minutes = sum(to_minutes(c["end_time"]) - to_minutes(c["start_time"]) for c in courses)
    return {
        "courses": courses,
        "semester": semester,
        "days": days,
        "today": today.isoformat(),
        "stats": {
            "course_count": len(courses),
            "today_count": len(today_courses),
            "weekly_hours": round(total_minutes / 60, 1),
        },
    }


# ---------------------------------------------------------------------------
# 借用
# ---------------------------------------------------------------------------


def _booking_dict(row: sqlite3.Row) -> dict:
    start_min = to_minutes(row["start_time"])
    end_min = to_minutes(row["end_time"])
    today = date.today()
    booking_date = date.fromisoformat(row["date"])
    if row["status"] == "cancelled":
        state = "cancelled"
    elif booking_date < today or (booking_date == today and end_min <= now_local().hour * 60 + now_local().minute):
        state = "ended"
    else:
        state = "active"
    return {
        "id": row["id"],
        "classroom_id": row["classroom_id"],
        "classroom_name": row["classroom_name"],
        "building": row["building"],
        "teacher_id": row["teacher_id"],
        "teacher_name": row["teacher_name"],
        "date": row["date"],
        "start_time": row["start_time"],
        "end_time": row["end_time"],
        "purpose": row["purpose"],
        "status": row["status"],
        "state": state,
        "created_at": row["created_at"],
    }


BOOKING_SELECT = (
    "SELECT b.*, r.name AS classroom_name, r.building, u.name AS teacher_name "
    "FROM bookings b JOIN classrooms r ON r.id = b.classroom_id "
    "JOIN users u ON u.id = b.teacher_id "
)


def list_bookings(conn: sqlite3.Connection, user: dict) -> list[dict]:
    require_role(user, "teacher", "admin")
    sql = BOOKING_SELECT
    params: list[Any] = []
    if user["role"] == "teacher":
        sql += "WHERE b.teacher_id = ? "
        params.append(user["id"])
    sql += "ORDER BY b.date DESC, b.start_time DESC"
    return [_booking_dict(row) for row in conn.execute(sql, params).fetchall()]


def _teacher_booking_conflict(
    conn: sqlite3.Connection,
    teacher_id: int,
    target: date,
    start_min: int,
    end_min: int,
) -> str | None:
    week = semester_info(conn, target)["week"]
    if week >= 1:
        for row in conn.execute(
            "SELECT c.*, r.name AS classroom_name FROM courses c "
            "JOIN classrooms r ON r.id = c.classroom_id "
            "WHERE c.teacher_id = ? AND c.weekday = ? AND c.weeks_start <= ? AND c.weeks_end >= ?",
            (teacher_id, target.isoweekday(), week, week),
        ).fetchall():
            if overlap(start_min, end_min, to_minutes(row["start_time"]), to_minutes(row["end_time"])):
                return (
                    f"你在该时段已有课程《{row['name']}》"
                    f"（{row['start_time']}-{row['end_time']}，{row['classroom_name']}）"
                )
    for row in conn.execute(
        "SELECT b.*, r.name AS classroom_name FROM bookings b "
        "JOIN classrooms r ON r.id = b.classroom_id "
        "WHERE b.teacher_id = ? AND b.date = ? AND b.status = 'active'",
        (teacher_id, target.isoformat()),
    ).fetchall():
        if overlap(start_min, end_min, to_minutes(row["start_time"]), to_minutes(row["end_time"])):
            return (
                f"你在该时段已借用 {row['classroom_name']} "
                f"（{row['start_time']}-{row['end_time']}）"
            )
    return None


def create_booking(conn: sqlite3.Connection, user: dict, payload: dict) -> dict:
    require_role(user, "teacher")
    room_id = _int_value(payload.get("classroom_id"), "教室", 1, 10**9)
    target = parse_date(payload.get("date", ""), "借用日期")
    start = normalize_time(payload.get("start_time", ""), "开始时间")
    end = normalize_time(payload.get("end_time", ""), "结束时间")
    purpose = _clean_text(payload.get("purpose"), "借用用途", 100)
    start_min, end_min = to_minutes(start), to_minutes(end)

    today = date.today()
    if target < today:
        raise ApiError("不能借用过去的日期")
    if target > today + timedelta(days=BOOKING_DAYS_AHEAD):
        raise ApiError(f"最多只能提前 {BOOKING_DAYS_AHEAD} 天借用")
    if target == today:
        now = now_local()
        if end_min <= now.hour * 60 + now.minute:
            raise ApiError("该时段已经结束，请选择稍后的时间")
    if end_min <= start_min:
        raise ApiError("结束时间必须晚于开始时间")
    if start_min < WINDOW_START or end_min > WINDOW_END:
        raise ApiError("借用时间需在 08:00-22:00 之间")

    room = conn.execute("SELECT * FROM classrooms WHERE id = ?", (room_id,)).fetchone()
    if not room:
        raise ApiError("教室不存在")
    if room["status"] != "active":
        raise ApiError("该教室正在维护中，暂不可借用")

    week = semester_info(conn, target)["week"]
    for course in _course_intervals(conn, target, week, classroom_id=room_id):
        if overlap(start_min, end_min, course["start_min"], course["end_min"]):
            raise ApiError(
                f"时间冲突：{course['title']} {course['start']}-{course['end']} "
                f"正在该教室上课"
            )
    for booking in _booking_intervals(conn, target, classroom_id=room_id):
        if overlap(start_min, end_min, booking["start_min"], booking["end_min"]):
            raise ApiError(
                f"时间冲突：该教室 {booking['start']}-{booking['end']} "
                f"已被 {booking['teacher']} 借用"
            )
    teacher_conflict = _teacher_booking_conflict(conn, user["id"], target, start_min, end_min)
    if teacher_conflict:
        raise ApiError(f"时间冲突：{teacher_conflict}")

    cur = conn.execute(
        "INSERT INTO bookings (classroom_id, teacher_id, date, start_time, end_time, purpose, status) "
        "VALUES (?, ?, ?, ?, ?, ?, 'active')",
        (room_id, user["id"], target.isoformat(), start, end, purpose),
    )
    conn.commit()
    row = conn.execute(BOOKING_SELECT + "WHERE b.id = ?", (cur.lastrowid,)).fetchone()
    return _booking_dict(row)


def cancel_booking(conn: sqlite3.Connection, user: dict, booking_id: int) -> dict:
    require_role(user, "teacher", "admin")
    row = conn.execute(BOOKING_SELECT + "WHERE b.id = ?", (booking_id,)).fetchone()
    if not row:
        raise ApiError("借用记录不存在", 404)
    if user["role"] == "teacher" and row["teacher_id"] != user["id"]:
        raise ApiError("只能取消自己的借用", 403)
    if row["status"] == "cancelled":
        raise ApiError("该借用已取消")
    booking_date = date.fromisoformat(row["date"])
    now = now_local()
    if booking_date < now.date() or (
        booking_date == now.date()
        and to_minutes(row["end_time"]) <= now.hour * 60 + now.minute
    ):
        raise ApiError("该借用已结束，无法取消")
    conn.execute(
        "UPDATE bookings SET status = 'cancelled', cancelled_at = CURRENT_TIMESTAMP WHERE id = ?",
        (booking_id,),
    )
    conn.commit()
    updated = conn.execute(BOOKING_SELECT + "WHERE b.id = ?", (booking_id,)).fetchone()
    return _booking_dict(updated)


# ---------------------------------------------------------------------------
# 教室与设置
# ---------------------------------------------------------------------------


def _validate_classroom(conn: sqlite3.Connection, payload: dict, exclude_id: int | None = None) -> dict:
    name = _clean_text(payload.get("name"), "教室名称", 40)
    building = _clean_text(payload.get("building"), "所属楼栋", 30)
    room_no = _clean_text(payload.get("room_no"), "房间号", 20)
    capacity = _int_value(payload.get("capacity"), "容纳人数", 1, 2000)
    room_type = _clean_text(payload.get("room_type"), "教室类型", 30)
    equipment = str(payload.get("equipment") or "").strip()
    if len(equipment) > 200:
        raise ApiError("设备说明不能超过 200 个字符")
    status = payload.get("status") or "active"
    if status not in ("active", "maintenance"):
        raise ApiError("教室状态不正确")
    duplicate = conn.execute(
        "SELECT id FROM classrooms WHERE (name = ? OR (building = ? AND room_no = ?)) AND id != ?",
        (name, building, room_no, exclude_id or 0),
    ).fetchone()
    if duplicate:
        raise ApiError("教室名称或楼栋房间号已存在")
    return {
        "name": name,
        "building": building,
        "room_no": room_no,
        "capacity": capacity,
        "room_type": room_type,
        "equipment": equipment,
        "status": status,
    }


def create_classroom(conn: sqlite3.Connection, user: dict, payload: dict) -> dict:
    require_role(user, "admin")
    values = _validate_classroom(conn, payload)
    cur = conn.execute(
        "INSERT INTO classrooms (name, building, room_no, capacity, room_type, equipment, status) "
        "VALUES (:name, :building, :room_no, :capacity, :room_type, :equipment, :status)",
        values,
    )
    conn.commit()
    row = conn.execute("SELECT * FROM classrooms WHERE id = ?", (cur.lastrowid,)).fetchone()
    return _classroom_dict(row)


def update_classroom(conn: sqlite3.Connection, user: dict, room_id: int, payload: dict) -> dict:
    require_role(user, "admin")
    existing = conn.execute("SELECT id FROM classrooms WHERE id = ?", (room_id,)).fetchone()
    if not existing:
        raise ApiError("教室不存在", 404)
    values = _validate_classroom(conn, payload, exclude_id=room_id)
    conn.execute(
        "UPDATE classrooms SET name = :name, building = :building, room_no = :room_no, "
        "capacity = :capacity, room_type = :room_type, equipment = :equipment, status = :status "
        "WHERE id = :id",
        {**values, "id": room_id},
    )
    conn.commit()
    row = conn.execute("SELECT * FROM classrooms WHERE id = ?", (room_id,)).fetchone()
    return _classroom_dict(row)


def list_classrooms(conn: sqlite3.Connection, user: dict) -> list[dict]:
    return [
        _classroom_dict(row)
        for row in conn.execute(
            f"SELECT * FROM classrooms ORDER BY {ROOM_ORDER_SQL}"
        ).fetchall()
    ]


def update_semester(conn: sqlite3.Connection, user: dict, payload: dict) -> dict:
    require_role(user, "admin")
    start = parse_date(payload.get("start", ""), "学期开始日期")
    name = _clean_text(payload.get("name"), "学期名称", 60)
    monday = start - timedelta(days=start.isoweekday() - 1)
    conn.execute(
        "INSERT INTO settings (key, value) VALUES ('semester_start', ?) "
        "ON CONFLICT(key) DO UPDATE SET value = excluded.value",
        (monday.isoformat(),),
    )
    conn.execute(
        "INSERT INTO settings (key, value) VALUES ('semester_name', ?) "
        "ON CONFLICT(key) DO UPDATE SET value = excluded.value",
        (name,),
    )
    conn.commit()
    return semester_info(conn)
`````


## `web.py`

`````python
"""HTTP 层：JSON API、静态资源与会话 Cookie。"""

from __future__ import annotations

import hmac
import json
import mimetypes
import os
import re
import traceback
import urllib.parse
from http.server import BaseHTTPRequestHandler
from pathlib import Path

import database
import services
from services import ApiError

BASE_DIR = Path(__file__).resolve().parent
STATIC_DIR = BASE_DIR / "static"
COOKIE_NAME = "classroom_session"
MAX_BODY = 1_000_000

ID_ROUTE = re.compile(r"^/api/(courses|classrooms|bookings)/(\d+)$")


class ApiHandler(BaseHTTPRequestHandler):
    server_version = "ZhixingClassroom/1.0"
    protocol_version = "HTTP/1.1"

    # ------------------------------------------------------------------
    # 请求入口
    # ------------------------------------------------------------------

    def do_GET(self) -> None:  # noqa: N802
        self._dispatch("GET")

    def do_POST(self) -> None:  # noqa: N802
        self._dispatch("POST")

    def do_PUT(self) -> None:  # noqa: N802
        self._dispatch("PUT")

    def do_DELETE(self) -> None:  # noqa: N802
        self._dispatch("DELETE")

    def _dispatch(self, method: str) -> None:
        parsed = urllib.parse.urlsplit(self.path)
        path = urllib.parse.unquote(parsed.path)
        query = urllib.parse.parse_qs(parsed.query)
        try:
            if path.startswith("/api/"):
                self._handle_api(method, path, query)
            elif method == "GET":
                self._serve_static(path)
            else:
                self._send_json(405, {"error": "不支持的请求方法"})
        except ApiError as exc:
            self._send_json(
                exc.status, {"error": exc.message, **({"details": exc.details} if exc.details else {})}
            )
        except (BrokenPipeError, ConnectionResetError):
            return
        except Exception:
            traceback.print_exc()
            self._send_json(500, {"error": "服务器开小差了，请稍后重试"})

    # ------------------------------------------------------------------
    # API
    # ------------------------------------------------------------------

    def _handle_api(self, method: str, path: str, query: dict) -> None:
        if path == "/api/health" and method == "GET":
            self._send_json(200, {"ok": True, "service": "zhixing-classroom"})
            return

        body = self._read_body() if method in ("POST", "PUT", "DELETE") else {}
        conn = database.connect()
        try:
            if path == "/api/login" and method == "POST":
                username = body.get("username", "")
                password = body.get("password", "")
                user = services.authenticate(conn, username, password)
                token = services.create_session(user["id"])
                csrf = services.csrf_for_session(token)
                self._send_json(
                    200,
                    {
                        "user": user,
                        "csrf": csrf,
                        "meta": services.get_meta(conn),
                    },
                    extra_headers=[self._session_cookie(token)],
                )
                return

            user, session = services.read_session(conn, self._session_token())
            if not user or not session:
                raise ApiError("登录状态已失效，请重新登录", 401)

            if method in ("POST", "PUT", "DELETE"):
                self._verify_csrf(session)

            if path == "/api/me" and method == "GET":
                self._send_json(200, {"user": user, "csrf": services.csrf_for_session(session)})
                return

            if path == "/api/logout" and method == "POST":
                self._send_json(
                    200,
                    {"ok": True},
                    extra_headers=[f"{COOKIE_NAME}=; Path=/; HttpOnly; SameSite=Lax; Max-Age=0"],
                )
                return

            if path == "/api/meta" and method == "GET":
                self._send_json(200, services.get_meta(conn))
                return

            if path == "/api/overview" and method == "GET":
                date_value = _first(query, "date")
                at_value = _first(query, "at")
                self._send_json(200, services.get_overview(conn, date_value, at_value, user))
                return

            if path == "/api/schedule" and method == "GET":
                self._send_json(200, services.get_schedule(conn, user))
                return

            if path == "/api/courses" and method == "GET":
                self._send_json(200, {"courses": services.list_courses(conn, user)})
                return

            if path == "/api/courses" and method == "POST":
                self._send_json(201, {"course": services.create_course(conn, user, body)})
                return

            if path == "/api/classrooms" and method == "GET":
                self._send_json(200, {"classrooms": services.list_classrooms(conn, user)})
                return

            if path == "/api/classrooms" and method == "POST":
                self._send_json(201, {"classroom": services.create_classroom(conn, user, body)})
                return

            if path == "/api/bookings" and method == "GET":
                self._send_json(200, {"bookings": services.list_bookings(conn, user)})
                return

            if path == "/api/bookings" and method == "POST":
                self._send_json(201, {"booking": services.create_booking(conn, user, body)})
                return

            if path == "/api/settings" and method == "GET":
                self._send_json(200, services.semester_info(conn))
                return

            if path == "/api/settings" and method == "PUT":
                semester = services.update_semester(conn, user, body)
                self._send_json(200, {"semester": semester})
                return

            match = ID_ROUTE.match(path)
            if match:
                resource, raw_id = match.groups()
                item_id = int(raw_id)
                if resource == "courses":
                    if method == "PUT":
                        self._send_json(
                            200, {"course": services.update_course(conn, user, item_id, body)}
                        )
                        return
                    if method == "DELETE":
                        services.delete_course(conn, user, item_id)
                        self._send_json(200, {"ok": True})
                        return
                if resource == "classrooms":
                    if method == "PUT":
                        self._send_json(
                            200,
                            {"classroom": services.update_classroom(conn, user, item_id, body)},
                        )
                        return
                if resource == "bookings":
                    if method == "DELETE":
                        self._send_json(
                            200, {"booking": services.cancel_booking(conn, user, item_id)}
                        )
                        return

            raise ApiError("接口不存在", 404)
        finally:
            conn.close()

    def _read_body(self) -> dict:
        try:
            length = int(self.headers.get("Content-Length") or 0)
        except ValueError:
            raise ApiError("请求体格式不正确")
        if length > MAX_BODY:
            raise ApiError("请求体过大", 413)
        if length == 0:
            return {}
        raw = self.rfile.read(length)
        try:
            data = json.loads(raw.decode("utf-8"))
        except (UnicodeDecodeError, json.JSONDecodeError):
            raise ApiError("请求体必须是合法的 JSON")
        if not isinstance(data, dict):
            raise ApiError("请求体格式不正确")
        return data

    # ------------------------------------------------------------------
    # 响应
    # ------------------------------------------------------------------

    def _send_json(self, status: int, payload: dict, extra_headers: list[str] | None = None) -> None:
        data = json.dumps(payload, ensure_ascii=False, separators=(",", ":")).encode("utf-8")
        self.send_response(status)
        self.send_header("Content-Type", "application/json; charset=utf-8")
        self.send_header("Content-Length", str(len(data)))
        self.send_header("Cache-Control", "no-store")
        self._security_headers()
        for header in extra_headers or []:
            self.send_header("Set-Cookie", header)
        self.end_headers()
        self.wfile.write(data)

    def _serve_static(self, path: str) -> None:
        if path == "/favicon.ico":
            self.send_response(204)
            self.end_headers()
            return
        if path in ("/", "/index.html"):
            file_path = STATIC_DIR / "index.html"
        elif path.startswith("/static/"):
            relative = path[len("/static/") :]
            file_path = (STATIC_DIR / relative).resolve()
            if STATIC_DIR.resolve() not in file_path.parents:
                self._send_json(403, {"error": "禁止访问"})
                return
        else:
            self._send_json(404, {"error": "页面不存在"})
            return

        if not file_path.is_file():
            self._send_json(404, {"error": "资源不存在"})
            return

        mime, _ = mimetypes.guess_type(file_path.name)
        if file_path.suffix == ".js":
            mime = "text/javascript; charset=utf-8"
        elif file_path.suffix == ".css":
            mime = "text/css; charset=utf-8"
        elif file_path.suffix == ".html":
            mime = "text/html; charset=utf-8"
        elif mime and mime.startswith("text/"):
            mime += "; charset=utf-8"
        else:
            mime = mime or "application/octet-stream"

        data = file_path.read_bytes()
        self.send_response(200)
        self.send_header("Content-Type", mime)
        self.send_header("Content-Length", str(len(data)))
        self.send_header(
            "Cache-Control",
            "no-cache" if file_path.suffix == ".html" else "public, max-age=3600",
        )
        self._security_headers()
        self.end_headers()
        self.wfile.write(data)

    def _security_headers(self) -> None:
        self.send_header("X-Content-Type-Options", "nosniff")
        self.send_header("X-Frame-Options", "DENY")
        self.send_header("Referrer-Policy", "same-origin")
        self.send_header(
            "Content-Security-Policy",
            "default-src 'self'; img-src 'self' data:; style-src 'self' 'unsafe-inline'; "
            "script-src 'self'; font-src 'self'; connect-src 'self'; frame-ancestors 'none'",
        )

    # ------------------------------------------------------------------
    # Cookie / CSRF
    # ------------------------------------------------------------------

    def _session_token(self) -> str | None:
        raw = self.headers.get("Cookie")
        if not raw:
            return None
        for part in raw.split(";"):
            name, _, value = part.strip().partition("=")
            if name == COOKIE_NAME:
                return value
        return None

    def _session_cookie(self, token: str) -> str:
        return (
            f"{COOKIE_NAME}={token}; Path=/; HttpOnly; SameSite=Lax; "
            f"Max-Age={services.SESSION_TTL}"
        )

    def _verify_csrf(self, session: str) -> None:
        expected = services.csrf_for_session(session)
        supplied = self.headers.get("X-CSRF-Token", "")
        if not supplied or not hmac.compare_digest(supplied, expected):
            raise ApiError("安全校验失败，请刷新页面后重试", 403)

    # ------------------------------------------------------------------
    # 日志
    # ------------------------------------------------------------------

    def log_message(self, format: str, *args) -> None:  # noqa: A002
        if os.environ.get("CLASSROOM_DEBUG"):
            super().log_message(format, *args)


def _first(query: dict, key: str) -> str | None:
    values = query.get(key)
    return values[0] if values else None
`````


## `static/index.html`

`````html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta name="color-scheme" content="light">
  <meta name="theme-color" content="#0f2442">
  <title>知行教室 · 教室课程管理系统</title>
  <link rel="stylesheet" href="/static/styles.css">
</head>
<body>
  <div id="app">
    <div class="boot-screen">
      <div class="boot-mark">知</div>
      <div class="boot-text">正在进入知行教室</div>
      <div class="boot-bar"><span></span></div>
    </div>
  </div>
  <div id="modal-root"></div>
  <div id="toast-root" class="toast-root" aria-live="polite"></div>
  <script type="module" src="/static/js/app.js" defer></script>
</body>
</html>
`````


## `static/styles.css`

`````css
/* 知行教室 · 视觉样式 */

:root {
  --ink: #142033;
  --ink-2: #24344d;
  --muted: #68758a;
  --muted-2: #8c99ac;
  --line: #e4eaf3;
  --line-strong: #d6dfec;
  --bg: #f4f7fb;
  --card: #ffffff;
  --navy: #0d1f39;
  --navy-2: #122b4c;
  --blue: #2563eb;
  --blue-2: #3b82f6;
  --teal: #0ea5a3;
  --green: #10b981;
  --green-soft: #e8f8f2;
  --amber: #f59e0b;
  --amber-soft: #fff6e6;
  --red: #e5484d;
  --red-soft: #fdeeee;
  --radius: 18px;
  --radius-sm: 12px;
  --shadow: 0 1px 2px rgba(15, 36, 66, 0.04), 0 12px 32px rgba(15, 36, 66, 0.055);
  --shadow-lg: 0 28px 80px rgba(9, 25, 48, 0.24);
  --font: "Segoe UI", "Microsoft YaHei", "PingFang SC", "Noto Sans SC", system-ui, -apple-system, sans-serif;
}

* {
  box-sizing: border-box;
}

[hidden] {
  display: none !important;
}

html {
  -webkit-text-size-adjust: 100%;
}

body {
  margin: 0;
  min-height: 100vh;
  background: var(--bg);
  color: var(--ink);
  font-family: var(--font);
  font-size: 14px;
  line-height: 1.5;
  -webkit-font-smoothing: antialiased;
}

button,
input,
select,
textarea {
  font: inherit;
}

button {
  color: inherit;
}

a {
  color: inherit;
  text-decoration: none;
}

h1,
h2,
h3,
h4,
p {
  margin: 0;
}

svg.icon {
  width: 18px;
  height: 18px;
  flex: 0 0 auto;
}

::selection {
  background: #cfe0ff;
}

/* ------------------------------------------------------------------ */
/* 启动与基础组件                                                      */
/* ------------------------------------------------------------------ */

.boot-screen {
  min-height: 100vh;
  display: grid;
  place-content: center;
  justify-items: center;
  gap: 14px;
  background: var(--navy);
  color: #fff;
}

.boot-mark {
  width: 54px;
  height: 54px;
  border-radius: 16px;
  display: grid;
  place-items: center;
  background: linear-gradient(135deg, #37d4c4, #2f7ef8);
  font-size: 25px;
  font-weight: 800;
  box-shadow: 0 14px 30px rgba(47, 126, 248, 0.35);
}

.boot-text {
  font-size: 14px;
  letter-spacing: 0.16em;
  color: #a9bdd8;
}

.boot-bar {
  width: 160px;
  height: 3px;
  background: rgba(255, 255, 255, 0.12);
  border-radius: 2px;
  overflow: hidden;
}

.boot-bar span {
  display: block;
  height: 100%;
  width: 45%;
  background: linear-gradient(90deg, #37d4c4, #4f8dfd);
  border-radius: 2px;
  animation: boot-slide 1.2s ease-in-out infinite;
}

@keyframes boot-slide {
  0% { transform: translateX(-110%); }
  100% { transform: translateX(260%); }
}

.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 7px;
  height: 42px;
  padding: 0 17px;
  border-radius: 11px;
  border: 1px solid transparent;
  background: transparent;
  font-size: 13.5px;
  font-weight: 650;
  cursor: pointer;
  white-space: nowrap;
  transition: transform 0.16s ease, box-shadow 0.16s ease, background 0.16s ease, border-color 0.16s ease;
}

.btn:hover:not(:disabled) {
  transform: translateY(-1px);
}

.btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
  transform: none !important;
}

.btn-primary {
  color: #fff;
  background: linear-gradient(135deg, #2f6ff0, #1f54d6);
  box-shadow: 0 8px 18px rgba(37, 99, 235, 0.24);
}

.btn-primary:hover:not(:disabled) {
  box-shadow: 0 12px 24px rgba(37, 99, 235, 0.32);
}

.btn-ghost {
  color: var(--ink-2);
  background: #fff;
  border-color: var(--line-strong);
}

.btn-ghost:hover:not(:disabled) {
  background: #f7faff;
  border-color: #b9c9e1;
}

.btn-danger {
  color: #fff;
  background: linear-gradient(135deg, #f05b60, #d7363d);
  box-shadow: 0 8px 18px rgba(229, 72, 77, 0.22);
}

.btn-danger-text {
  color: var(--red);
  border-color: #f4d3d4;
  background: #fffafa;
}

.btn-sm {
  height: 34px;
  padding: 0 12px;
  font-size: 12.5px;
  border-radius: 9px;
}

.btn-spinner {
  width: 15px;
  height: 15px;
  border-radius: 50%;
  border: 2px solid rgba(255, 255, 255, 0.4);
  border-top-color: #fff;
  animation: spin 0.7s linear infinite;
}

.spinner {
  display: inline-block;
  width: 17px;
  height: 17px;
  border-radius: 50%;
  border: 2px solid #cbd9ee;
  border-top-color: var(--blue);
  animation: spin 0.7s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.icon-btn {
  display: inline-grid;
  place-items: center;
  width: 38px;
  height: 38px;
  border: 1px solid var(--line);
  border-radius: 10px;
  background: #fff;
  color: var(--muted);
  cursor: pointer;
  transition: 0.16s ease;
}

.icon-btn:hover {
  color: var(--ink);
  border-color: var(--line-strong);
  background: #f8fbff;
}

.eyebrow {
  display: inline-block;
  font-size: 10.5px;
  line-height: 1;
  font-weight: 750;
  letter-spacing: 0.18em;
  text-transform: uppercase;
  color: var(--blue);
}

.live-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: var(--green);
  box-shadow: 0 0 0 4px rgba(16, 185, 129, 0.12);
  animation: pulse-dot 2.2s ease-out infinite;
}

@keyframes pulse-dot {
  0% { box-shadow: 0 0 0 0 rgba(16, 185, 129, 0.35); }
  70% { box-shadow: 0 0 0 7px rgba(16, 185, 129, 0); }
  100% { box-shadow: 0 0 0 0 rgba(16, 185, 129, 0); }
}

/* ------------------------------------------------------------------ */
/* 登录                                                                */
/* ------------------------------------------------------------------ */

.login-page {
  min-height: 100vh;
  display: grid;
  grid-template-columns: minmax(0, 1.08fr) minmax(420px, 0.92fr);
  background: #fff;
}

.login-visual {
  position: relative;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  padding: 42px 54px 28px;
  background:
    radial-gradient(1100px 700px at 10% 12%, rgba(47, 126, 248, 0.25), transparent 60%),
    radial-gradient(700px 520px at 92% 88%, rgba(28, 196, 177, 0.2), transparent 62%),
    linear-gradient(145deg, #0a1930 0%, #0e2748 53%, #0b1c34 100%);
  color: #fff;
}

.login-visual::before {
  content: "";
  position: absolute;
  inset: 0;
  background-image:
    linear-gradient(rgba(255, 255, 255, 0.038) 1px, transparent 1px),
    linear-gradient(90deg, rgba(255, 255, 255, 0.038) 1px, transparent 1px);
  background-size: 58px 58px;
  mask-image: linear-gradient(to bottom, rgba(0, 0, 0, 0.7), transparent 78%);
  pointer-events: none;
}

.login-visual::after {
  content: "";
  position: absolute;
  width: 540px;
  height: 540px;
  right: -240px;
  top: -230px;
  border-radius: 50%;
  border: 1px solid rgba(255, 255, 255, 0.09);
  box-shadow: 0 0 0 80px rgba(255, 255, 255, 0.018), 0 0 0 160px rgba(255, 255, 255, 0.012);
  pointer-events: none;
}

.login-visual-bg {
  position: absolute;
  inset: 0;
}

.login-brand,
.login-copy,
.login-features,
.login-showcase,
.login-visual-foot {
  position: relative;
  z-index: 1;
}

.login-brand {
  display: flex;
  align-items: center;
  gap: 13px;
}

.brand {
  display: flex;
  align-items: center;
  gap: 12px;
}

.brand-mark {
  width: 41px;
  height: 41px;
  flex: 0 0 auto;
  border-radius: 13px;
  display: grid;
  place-items: center;
  color: #fff;
  font-size: 20px;
  font-weight: 800;
  background: linear-gradient(140deg, #36d1c1 0%, #2f7ef8 100%);
  box-shadow: 0 8px 20px rgba(37, 99, 235, 0.24);
}

.brand-mark-light {
  box-shadow: 0 0 0 1px rgba(255, 255, 255, 0.1), 0 14px 28px rgba(0, 0, 0, 0.2);
}

.brand-copy {
  display: flex;
  flex-direction: column;
  line-height: 1.25;
}

.brand-copy strong {
  font-size: 17px;
  font-weight: 750;
  letter-spacing: 0.06em;
}

.brand-copy span {
  font-size: 10px;
  letter-spacing: 0.16em;
  color: var(--muted-2);
  margin-top: 3px;
}

.login-brand .brand-copy span {
  color: #91a9c8;
}

.login-copy {
  margin-top: clamp(48px, 9vh, 92px);
  max-width: 590px;
}

.eyebrow-light {
  color: #72ddcf;
}

.login-copy h1 {
  margin-top: 17px;
  font-size: clamp(40px, 4.4vw, 60px);
  line-height: 1.14;
  font-weight: 760;
  letter-spacing: -0.035em;
}

.login-copy h1 em {
  font-style: normal;
  background: linear-gradient(100deg, #79ead9, #7fb4ff 72%);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
}

.login-copy p {
  margin-top: 20px;
  max-width: 480px;
  font-size: 14.5px;
  line-height: 1.85;
  color: #a7bcd8;
}

.login-features {
  display: flex;
  gap: 34px;
  margin-top: 32px;
  flex-wrap: wrap;
}

.login-feature {
  display: flex;
  align-items: center;
  gap: 10px;
}

.login-feature > span {
  width: 38px;
  height: 38px;
  display: grid;
  place-items: center;
  border-radius: 11px;
  background: rgba(255, 255, 255, 0.07);
  border: 1px solid rgba(255, 255, 255, 0.08);
  color: #79ded2;
}

.login-feature div {
  display: flex;
  flex-direction: column;
}

.login-feature strong {
  font-size: 13px;
  font-weight: 650;
}

.login-feature small {
  font-size: 10.5px;
  color: #8299b8;
  margin-top: 2px;
}

.login-showcase {
  margin-top: auto;
  padding-top: 38px;
  width: min(460px, 100%);
}

.showcase-top {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 10.5px;
  letter-spacing: 0.08em;
  color: #b6c8df;
  margin-bottom: 11px;
}

.showcase-top em {
  margin-left: auto;
  font-style: normal;
  color: #7f96b4;
  font-size: 10px;
}

.login-showcase {
  background: linear-gradient(145deg, rgba(255, 255, 255, 0.085), rgba(255, 255, 255, 0.035));
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 16px;
  padding: 18px 19px 15px;
  backdrop-filter: blur(8px);
}

.login-showcase .showcase-top {
  margin-bottom: 12px;
}

.showcase-room {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.showcase-room div {
  display: flex;
  flex-direction: column;
}

.showcase-room strong {
  font-size: 14px;
  font-weight: 650;
}

.showcase-room small {
  color: #8fa6c3;
  font-size: 10.5px;
  margin-top: 3px;
}

.showcase-status {
  font-size: 10px;
  font-weight: 650;
  color: #7df0c9;
  background: rgba(16, 185, 129, 0.14);
  border: 1px solid rgba(81, 226, 174, 0.2);
  border-radius: 999px;
  padding: 4px 9px;
}

.showcase-timeline {
  position: relative;
  height: 8px;
  margin-top: 16px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.09);
  overflow: visible;
}

.showcase-seg {
  position: absolute;
  top: 0;
  height: 8px;
  border-radius: 999px;
}

.seg-course { background: #5b8def; }
.seg-free { background: rgba(55, 212, 196, 0.5); }
.seg-book { background: #f1b25c; }

.showcase-now {
  position: absolute;
  top: -5px;
  width: 2px;
  height: 18px;
  border-radius: 2px;
  background: #fff;
  box-shadow: 0 0 0 3px rgba(255, 255, 255, 0.16);
}

.showcase-scale {
  display: flex;
  justify-content: space-between;
  margin-top: 9px;
  color: #7890ad;
  font-size: 9.5px;
  letter-spacing: 0.04em;
}

.login-visual-foot {
  margin-top: 22px;
  font-size: 10.5px;
  color: #6e86a5;
  letter-spacing: 0.06em;
}

.login-panel {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 48px 42px 28px;
  background:
    radial-gradient(700px 450px at 85% 8%, rgba(37, 99, 235, 0.055), transparent 60%),
    #fff;
}

.login-card {
  width: min(430px, 100%);
}

.login-card-head h2 {
  margin-top: 12px;
  font-size: 28px;
  letter-spacing: -0.03em;
  font-weight: 730;
}

.login-card-head p {
  margin-top: 7px;
  color: var(--muted);
  font-size: 13px;
}

.login-alert {
  display: flex;
  align-items: center;
  gap: 9px;
  margin-top: 20px;
  padding: 11px 13px;
  border-radius: 10px;
  background: #eef5ff;
  border: 1px solid #d7e6ff;
  color: #275aa8;
  font-size: 12.5px;
}

.login-alert-error {
  background: var(--red-soft);
  border-color: #f5d3d5;
  color: #b52c32;
}

.login-form {
  display: grid;
  gap: 17px;
  margin-top: 28px;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 7px;
  min-width: 0;
}

.field > span {
  font-size: 12.5px;
  font-weight: 650;
  color: var(--ink-2);
}

.field input,
.field select,
.field textarea,
.compact-field input,
.compact-field select,
.search-input,
.time-range {
  width: 100%;
  border: 1px solid var(--line-strong);
  background: #fbfcfe;
  border-radius: 10px;
  color: var(--ink);
  transition: border-color 0.16s ease, box-shadow 0.16s ease, background 0.16s ease;
}

.field input,
.field select {
  height: 45px;
  padding: 0 13px;
}

.field textarea {
  padding: 12px 13px;
  resize: vertical;
  min-height: 90px;
  line-height: 1.6;
}

.field input:focus,
.field select:focus,
.field textarea:focus,
.compact-field input:focus,
.compact-field select:focus,
.search-input:focus-within {
  outline: none;
  border-color: #7ca7f8;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.12);
  background: #fff;
}

.field input::placeholder,
.field textarea::placeholder,
.compact-field input::placeholder,
.search-input input::placeholder {
  color: #a5afbd;
}

.input-with-icon {
  position: relative;
}

.input-with-icon > .icon {
  position: absolute;
  left: 14px;
  top: 50%;
  transform: translateY(-50%);
  color: #93a0b4;
  pointer-events: none;
}

.input-with-icon input {
  padding-left: 43px;
  padding-right: 58px;
}

.password-toggle {
  position: absolute;
  right: 10px;
  top: 50%;
  transform: translateY(-50%);
  border: 0;
  background: none;
  color: var(--blue);
  font-size: 11.5px;
  font-weight: 600;
  cursor: pointer;
  padding: 6px;
}

.btn-login {
  width: 100%;
  height: 48px;
  margin-top: 3px;
  border-radius: 11px;
  font-size: 14px;
}

.login-divider {
  display: flex;
  align-items: center;
  gap: 13px;
  margin: 25px 0 16px;
  color: #97a3b5;
  font-size: 10.5px;
  letter-spacing: 0.04em;
}

.login-divider::before,
.login-divider::after {
  content: "";
  height: 1px;
  flex: 1;
  background: var(--line);
}

.demo-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 9px;
}

.demo-account {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 2px;
  padding: 12px 11px;
  text-align: left;
  background: #fbfcff;
  border: 1px solid var(--line);
  border-radius: 11px;
  cursor: pointer;
  transition: 0.16s ease;
}

.demo-account:hover {
  border-color: #a9c5f8;
  background: #f4f8ff;
  transform: translateY(-1px);
  box-shadow: 0 8px 18px rgba(37, 99, 235, 0.08);
}

.demo-role {
  font-size: 9.5px;
  font-weight: 700;
  letter-spacing: 0.08em;
  color: var(--blue);
}

.demo-account strong {
  font-size: 12.5px;
  font-weight: 700;
  color: var(--ink);
}

.demo-account small {
  font-size: 9.5px;
  color: var(--muted-2);
  white-space: nowrap;
}

.demo-hint {
  margin-top: 14px;
  text-align: center;
  color: #94a0b2;
  font-size: 10.5px;
}

.demo-hint code {
  background: #f1f5fb;
  border: 1px solid var(--line);
  border-radius: 4px;
  padding: 1px 5px;
  font-family: inherit;
  color: #4a5b73;
}

.login-foot {
  margin-top: 28px;
  padding-top: 0;
  color: #a0abb9;
  font-size: 10.5px;
  letter-spacing: 0.03em;
  text-align: center;
}

/* ------------------------------------------------------------------ */
/* 应用外壳                                                            */
/* ------------------------------------------------------------------ */

.app-shell {
  min-height: 100vh;
}

.sidebar {
  position: fixed;
  inset: 0 auto 0 0;
  z-index: 40;
  width: 258px;
  display: flex;
  flex-direction: column;
  padding: 25px 17px 18px;
  background:
    radial-gradient(480px 380px at 0% 0%, rgba(47, 126, 248, 0.22), transparent 62%),
    linear-gradient(180deg, #0c1d35 0%, #0e2340 62%, #0b1a2f 100%);
  color: #fff;
  overflow-y: auto;
  scrollbar-width: thin;
}

.sidebar .brand {
  padding: 0 7px;
}

.sidebar .brand-copy strong {
  color: #f2f6fd;
}

.sidebar .brand-copy span {
  color: #7c93b2;
}

.sidebar-term {
  margin-top: 26px;
  padding: 13px 13px 12px;
  border-radius: 13px;
  background: rgba(255, 255, 255, 0.045);
  border: 1px solid rgba(255, 255, 255, 0.07);
}

.term-top {
  display: flex;
  align-items: center;
  justify-content: space-between;
  color: #7f95b4;
  font-size: 9.5px;
  letter-spacing: 0.08em;
}

.term-top em {
  font-style: normal;
  color: #6ee7d6;
  background: rgba(45, 212, 191, 0.1);
  border-radius: 999px;
  padding: 2px 7px;
  font-size: 9px;
  letter-spacing: 0.02em;
}

.sidebar-term > strong {
  display: block;
  margin-top: 7px;
  font-size: 12px;
  line-height: 1.45;
  color: #dce8f8;
  font-weight: 600;
}

.term-bar {
  height: 4px;
  margin-top: 10px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.08);
  overflow: hidden;
}

.term-bar span {
  display: block;
  height: 100%;
  border-radius: inherit;
  background: linear-gradient(90deg, #34d3c0, #5f9bff);
}

.sidebar-term small {
  display: block;
  margin-top: 7px;
  color: #6f86a5;
  font-size: 9.5px;
}

.nav-label {
  margin: 24px 12px 9px;
  font-size: 9.5px;
  letter-spacing: 0.15em;
  color: #617a9a;
  font-weight: 700;
}

.sidebar-nav {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.nav-item {
  position: relative;
  display: flex;
  align-items: center;
  gap: 11px;
  padding: 10px 11px;
  border-radius: 11px;
  color: #a4b8d3;
  transition: 0.16s ease;
}

.nav-item:hover {
  background: rgba(255, 255, 255, 0.045);
  color: #eaf2ff;
}

.nav-item.is-active {
  background: linear-gradient(100deg, rgba(59, 130, 246, 0.2), rgba(45, 212, 191, 0.08));
  color: #fff;
  box-shadow: inset 0 0 0 1px rgba(124, 169, 255, 0.16);
}

.nav-item.is-active::before {
  content: "";
  position: absolute;
  left: 0;
  top: 13px;
  bottom: 13px;
  width: 3px;
  border-radius: 0 3px 3px 0;
  background: #4bd7c6;
}

.nav-icon {
  width: 30px;
  height: 30px;
  display: grid;
  place-items: center;
  border-radius: 8px;
  background: rgba(255, 255, 255, 0.04);
  color: #89a3c4;
}

.nav-item.is-active .nav-icon {
  color: #6ee7d6;
  background: rgba(45, 212, 191, 0.09);
}

.nav-copy {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.nav-copy strong {
  font-size: 12.5px;
  font-weight: 600;
  letter-spacing: 0.01em;
}

.nav-copy small {
  margin-top: 1px;
  font-size: 9px;
  color: #69809f;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.nav-item.is-active .nav-copy small {
  color: #8eabc9;
}

.nav-arrow {
  margin-left: auto;
  opacity: 0;
  transform: translateX(-3px);
  transition: 0.16s ease;
}

.nav-arrow .icon {
  width: 13px;
  height: 13px;
}

.nav-item:hover .nav-arrow,
.nav-item.is-active .nav-arrow {
  opacity: 0.7;
  transform: translateX(0);
}

.sidebar-foot {
  margin-top: auto;
  padding-top: 18px;
}

.sidebar-tip {
  display: flex;
  gap: 10px;
  padding: 11px 12px;
  border-radius: 11px;
  background: rgba(255, 255, 255, 0.035);
  border: 1px solid rgba(255, 255, 255, 0.05);
}

.tip-icon {
  color: #6ee7d6;
  margin-top: 1px;
}

.tip-icon .icon {
  width: 15px;
  height: 15px;
}

.sidebar-tip div {
  display: flex;
  flex-direction: column;
}

.sidebar-tip strong {
  font-size: 10.5px;
  color: #c6d7eb;
  font-weight: 600;
}

.sidebar-tip small {
  margin-top: 2px;
  font-size: 9px;
  color: #6e87a6;
}

.sidebar-user {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-top: 13px;
  padding: 11px 7px 0;
  border-top: 1px solid rgba(255, 255, 255, 0.07);
}

.avatar {
  width: 36px;
  height: 36px;
  flex: 0 0 auto;
  border-radius: 10px;
  display: grid;
  place-items: center;
  background: linear-gradient(140deg, #2e6ee8, #24b8ad);
  color: #fff;
  font-size: 13px;
  font-weight: 700;
}

.avatar-sm {
  width: 27px;
  height: 27px;
  border-radius: 8px;
  font-size: 10.5px;
}

.user-copy {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.user-copy strong {
  font-size: 12px;
  color: #e6eefb;
  font-weight: 600;
}

.user-copy span {
  margin-top: 1px;
  font-size: 9px;
  color: #7189a8;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.logout-btn {
  margin-left: auto;
  width: 32px;
  height: 32px;
  padding: 0;
  border-color: transparent;
  background: transparent;
  color: #728ba9;
}

.logout-btn:hover {
  background: rgba(255, 255, 255, 0.06);
  color: #ff9da1;
  border-color: transparent;
}

.logout-btn .icon {
  width: 16px;
  height: 16px;
}

.main-shell {
  margin-left: 258px;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.topbar {
  position: sticky;
  top: 0;
  z-index: 30;
  height: 76px;
  flex: 0 0 auto;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 20px;
  padding: 0 32px;
  background: rgba(244, 247, 251, 0.86);
  backdrop-filter: blur(14px);
  border-bottom: 1px solid rgba(217, 227, 240, 0.72);
}

.topbar-left {
  display: flex;
  align-items: center;
  gap: 12px;
  min-width: 0;
}

.mobile-menu {
  display: none;
}

.topbar-heading span {
  display: block;
  font-size: 9px;
  font-weight: 700;
  letter-spacing: 0.16em;
  color: #8b99ac;
}

.topbar-heading h1 {
  margin-top: 2px;
  font-size: 17px;
  font-weight: 700;
  letter-spacing: -0.01em;
}

.topbar-right {
  display: flex;
  align-items: center;
  gap: 16px;
}

.live-clock {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  line-height: 1.2;
}

.live-clock strong {
  font-size: 13px;
  font-weight: 650;
  letter-spacing: 0.04em;
  font-variant-numeric: tabular-nums;
}

.live-clock span {
  margin-top: 2px;
  font-size: 9.5px;
  color: var(--muted-2);
}

.topbar-role {
  padding: 5px 10px;
  border-radius: 999px;
  background: #eaf1ff;
  border: 1px solid #d6e4ff;
  color: #2b5db8;
  font-size: 10.5px;
  font-weight: 650;
}

.page-content {
  width: 100%;
  max-width: 1500px;
  margin: 0 auto;
  padding: 28px 32px 54px;
  flex: 1;
}

.sidebar-scrim {
  display: none;
}

/* ------------------------------------------------------------------ */
/* 页面标题与筛选                                                      */
/* ------------------------------------------------------------------ */

.page-heading {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 24px;
  margin-bottom: 22px;
}

.page-heading h1 {
  margin-top: 8px;
  font-size: 30px;
  line-height: 1.15;
  font-weight: 750;
  letter-spacing: -0.035em;
}

.page-heading p {
  margin-top: 8px;
  color: var(--muted);
  font-size: 13px;
}

.heading-actions {
  display: flex;
  align-items: center;
  gap: 9px;
  flex: 0 0 auto;
}

.live-pill {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  height: 36px;
  padding: 0 13px;
  border-radius: 999px;
  background: #fff;
  border: 1px solid var(--line);
  box-shadow: 0 3px 10px rgba(15, 36, 66, 0.04);
  color: var(--muted);
  font-size: 11px;
}

.live-pill b {
  font-weight: 650;
  color: var(--ink-2);
  font-variant-numeric: tabular-nums;
}

.live-pill-static {
  gap: 7px;
  color: var(--ink-2);
  font-weight: 600;
}

.notice {
  display: flex;
  align-items: flex-start;
  gap: 9px;
  padding: 11px 14px;
  margin-bottom: 16px;
  border-radius: 11px;
  font-size: 12px;
  line-height: 1.6;
}

.notice-info {
  background: #eef5ff;
  border: 1px solid #dbe8ff;
  color: #2b5a9f;
}

.notice-warning {
  background: #fff8e9;
  border: 1px solid #f6e3b9;
  color: #956418;
}

.notice-error {
  background: var(--red-soft);
  border: 1px solid #f3d1d3;
  color: #af2d32;
}

.notice-icon {
  margin-top: 1px;
}

.notice-icon .icon {
  width: 15px;
  height: 15px;
}

.notice > span:nth-child(2) {
  flex: 1;
  min-width: 0;
}

.notice-btn {
  margin-left: auto;
  flex: 0 0 auto;
  align-self: center;
  height: 27px;
  padding: 0 10px;
  border-radius: 7px;
  border: 1px solid #e7c987;
  background: #fffaf0;
  color: #875b10;
  font-size: 10px;
  font-weight: 650;
  cursor: pointer;
  white-space: nowrap;
}

.notice-btn:hover {
  background: #fff3d7;
}

.filter-card {
  background: #fff;
  border: 1px solid var(--line);
  border-radius: var(--radius);
  padding: 17px 19px 14px;
  box-shadow: var(--shadow);
  margin-bottom: 22px;
}

.filter-grid {
  display: grid;
  grid-template-columns: 155px 125px 145px 130px minmax(210px, 1fr) auto;
  gap: 12px;
  align-items: end;
}

.compact-field {
  display: flex;
  flex-direction: column;
  gap: 6px;
  min-width: 0;
}

.compact-field > span {
  font-size: 10.5px;
  font-weight: 650;
  color: #7b889b;
  letter-spacing: 0.02em;
}

.compact-field input,
.compact-field select {
  height: 40px;
  padding: 0 11px;
  font-size: 12.5px;
}

.search-input {
  display: flex;
  align-items: center;
  gap: 8px;
  height: 40px;
  padding: 0 11px;
  background: #fbfcfe;
  border: 1px solid var(--line-strong);
  border-radius: 10px;
  transition: 0.16s ease;
}

.search-input .icon {
  color: #98a4b5;
  width: 15px;
  height: 15px;
}

.search-input input {
  width: 100%;
  min-width: 0;
  height: 100%;
  padding: 0;
  border: 0;
  outline: none;
  background: transparent;
  font-size: 12.5px;
}

.btn-now {
  height: 40px;
}

.filter-bottom {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 14px;
  margin-top: 14px;
  padding-top: 12px;
  border-top: 1px solid #edf1f7;
}

.segmented {
  display: inline-flex;
  gap: 2px;
  padding: 3px;
  border-radius: 9px;
  background: #f2f5fa;
}

.segment {
  border: 0;
  background: transparent;
  border-radius: 7px;
  padding: 6px 12px;
  color: #708096;
  font-size: 11.5px;
  font-weight: 550;
  cursor: pointer;
  transition: 0.15s ease;
}

.segment:hover {
  color: var(--ink-2);
}

.segment.is-active {
  color: #1d4ed8;
  background: #fff;
  box-shadow: 0 2px 6px rgba(20, 45, 82, 0.09);
  font-weight: 650;
}

.filter-count {
  margin-left: auto;
  color: #8b97a9;
  font-size: 10.5px;
  white-space: nowrap;
}

.course-filter-grid {
  grid-template-columns: minmax(260px, 1fr) 180px;
}

.admin-booking-filter {
  grid-template-columns: minmax(280px, 1fr) 190px;
}

.classroom-filter-grid {
  display: grid;
  grid-template-columns: minmax(260px, 1fr) 180px 160px;
  gap: 12px;
  align-items: end;
}

.borrow-filter-grid {
  display: grid;
  grid-template-columns: 155px 125px 125px 150px 130px auto;
  gap: 12px;
  align-items: end;
}

.btn-search-rooms {
  height: 40px;
}

/* ------------------------------------------------------------------ */
/* 统计卡片                                                            */
/* ------------------------------------------------------------------ */

.stats-grid {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 15px;
  margin-bottom: 24px;
}

.stat-card {
  position: relative;
  display: flex;
  align-items: flex-start;
  gap: 13px;
  min-width: 0;
  padding: 18px 18px 16px;
  background: #fff;
  border: 1px solid var(--line);
  border-radius: 16px;
  box-shadow: var(--shadow);
  overflow: hidden;
}

.stat-card::after {
  content: "";
  position: absolute;
  right: -22px;
  top: -26px;
  width: 86px;
  height: 86px;
  border-radius: 50%;
  background: currentColor;
  opacity: 0.035;
}

.stat-icon {
  width: 42px;
  height: 42px;
  flex: 0 0 auto;
  display: grid;
  place-items: center;
  border-radius: 12px;
}

.stat-icon .icon {
  width: 19px;
  height: 19px;
}

.stat-free { color: var(--green); }
.stat-free .stat-icon { background: var(--green-soft); color: #07996c; }
.stat-busy { color: var(--amber); }
.stat-busy .stat-icon { background: var(--amber-soft); color: #cf8a13; }
.stat-book { color: var(--blue); }
.stat-book .stat-icon { background: #eaf1ff; color: #2f6be0; }
.stat-muted { color: #8492a6; }
.stat-muted .stat-icon { background: #f1f4f8; color: #7b8a9f; }

.stat-body {
  min-width: 0;
  flex: 1;
}

.stat-label {
  display: block;
  font-size: 11px;
  font-weight: 550;
  color: #7e8b9e;
}

.stat-value {
  margin-top: 4px;
  font-size: 28px;
  line-height: 1.15;
  font-weight: 720;
  letter-spacing: -0.045em;
  color: var(--ink);
}

.stat-value small {
  margin-left: 4px;
  font-size: 10.5px;
  font-weight: 500;
  letter-spacing: 0;
  color: #93a0b2;
}

.stat-card p {
  margin-top: 5px;
  font-size: 10px;
  color: #96a2b3;
  line-height: 1.45;
}

.stat-meter {
  height: 4px;
  margin-top: 8px;
  border-radius: 999px;
  background: #eef3f9;
  overflow: hidden;
}

.stat-meter span {
  display: block;
  height: 100%;
  border-radius: inherit;
  background: linear-gradient(90deg, #34d3b1, #4a9cf5);
}

.stats-grid-mini {
  grid-template-columns: repeat(auto-fit, minmax(190px, 1fr));
  margin-bottom: 20px;
}

.stats-grid-mini .stat-card {
  padding: 14px 16px;
}

.stats-grid-mini .stat-value {
  font-size: 23px;
}

.stat-card-horizontal {
  align-items: center;
}

.stat-card-horizontal .stat-icon {
  width: 38px;
  height: 38px;
}

.stats-grid-mini .stat-card-horizontal:nth-child(3) {
  grid-column: auto;
}

/* ------------------------------------------------------------------ */
/* 分区标题与图例                                                      */
/* ------------------------------------------------------------------ */

.section-head {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 20px;
  margin: 25px 0 14px;
}

.section-head.compact-head {
  margin-top: 24px;
}

.section-head h2 {
  display: flex;
  align-items: center;
  gap: 9px;
  font-size: 18px;
  font-weight: 700;
  letter-spacing: -0.02em;
}

.section-head p {
  margin-top: 5px;
  font-size: 11.5px;
  color: var(--muted-2);
}

.section-badge {
  display: inline-flex;
  align-items: center;
  height: 20px;
  padding: 0 7px;
  border-radius: 999px;
  background: #edf3fc;
  border: 1px solid #dfe9f8;
  color: #5e7594;
  font-size: 9.5px;
  font-weight: 550;
  letter-spacing: 0;
}

.legend {
  display: flex;
  align-items: center;
  gap: 14px;
  color: #8794a7;
  font-size: 10px;
  white-space: nowrap;
}

.legend span {
  display: inline-flex;
  align-items: center;
  gap: 5px;
}

.legend-dot {
  width: 7px;
  height: 7px;
  border-radius: 3px;
  display: inline-block;
}

.legend-dot.course { background: #5b8def; }
.legend-dot.booking { background: #efb45d; }
.legend-dot.now { background: #ef4444; }
.legend-dot.free { background: var(--green); }
.legend-dot.busy { background: var(--amber); }

/* ------------------------------------------------------------------ */
/* 教室卡片                                                            */
/* ------------------------------------------------------------------ */

.room-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(302px, 1fr));
  gap: 15px;
}

.room-card {
  position: relative;
  display: flex;
  flex-direction: column;
  padding: 17px 18px 14px;
  background: #fff;
  border: 1px solid var(--line);
  border-radius: var(--radius);
  box-shadow: var(--shadow);
  transition: transform 0.18s ease, box-shadow 0.18s ease, border-color 0.18s ease;
  overflow: hidden;
}

.room-card:hover {
  transform: translateY(-2px);
  border-color: #d3dfef;
  box-shadow: 0 16px 38px rgba(15, 36, 66, 0.08);
}

.room-card::before {
  content: "";
  position: absolute;
  inset: 0 auto 0 0;
  width: 3px;
  background: var(--green);
  opacity: 0.65;
}

.room-card.room-busy::before { background: var(--amber); }
.room-card.room-maintenance::before { background: #a8b3c2; }

.room-card-head {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
}

.room-title-wrap {
  display: flex;
  align-items: center;
  gap: 11px;
  min-width: 0;
}

.room-icon {
  width: 40px;
  height: 40px;
  flex: 0 0 auto;
  display: grid;
  place-items: center;
  border-radius: 11px;
  background: #eef4ff;
  color: #356bd4;
}

.room-icon .icon {
  width: 18px;
  height: 18px;
}

.room-title-wrap h3 {
  font-size: 14.5px;
  font-weight: 680;
  letter-spacing: -0.01em;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.room-title-wrap p {
  margin-top: 2px;
  font-size: 10px;
  color: #8b98aa;
}

.status-pill {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  flex: 0 0 auto;
  height: 22px;
  padding: 0 8px;
  border-radius: 999px;
  font-size: 9.5px;
  font-weight: 650;
  white-space: nowrap;
}

.status-pill i {
  width: 4px;
  height: 4px;
  border-radius: 50%;
  background: currentColor;
}

.status-free {
  color: #07845f;
  background: #e8f8f2;
  border: 1px solid #c9efe3;
}

.status-busy {
  color: #a46707;
  background: #fff5e2;
  border: 1px solid #f7e3ba;
}

.status-maintenance {
  color: #748196;
  background: #f1f4f8;
  border: 1px solid #e2e8f1;
}

.status-muted {
  color: #748196;
  background: #f1f4f8;
  border: 1px solid #e2e8f1;
}

.room-facts {
  display: flex;
  flex-wrap: wrap;
  gap: 6px 13px;
  margin-top: 12px;
  padding-bottom: 12px;
  border-bottom: 1px solid #f0f3f8;
}

.room-facts span {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  color: #7f8da0;
  font-size: 9.5px;
}

.room-facts .icon {
  width: 12px;
  height: 12px;
  color: #9aa8ba;
}

.occupancy-note {
  margin-top: 12px;
  min-height: 59px;
  padding: 10px 11px;
  border-radius: 11px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 4px;
}

.occupancy-note.is-free {
  background: #f2fbf7;
  border: 1px solid #e2f4ec;
}

.occupancy-note.is-busy {
  background: #fffaf0;
  border: 1px solid #f9edd3;
}

.occupancy-note.is-maintenance {
  background: #f6f8fb;
  border: 1px solid #ebeff5;
}

.occupancy-note > span {
  font-size: 9.5px;
  color: #8390a2;
}

.occupancy-title {
  display: flex;
  align-items: center;
  gap: 7px;
}

.occupancy-title strong {
  font-size: 12px;
  font-weight: 650;
  color: var(--ink-2);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.now-badge {
  display: inline-flex;
  align-items: center;
  height: 18px;
  padding: 0 6px;
  border-radius: 5px;
  background: #dff5eb;
  color: #087e5d;
  font-size: 8.5px;
  font-weight: 700;
  flex: 0 0 auto;
}

.is-busy .now-badge {
  background: #ffedca;
  color: #9b6305;
}

.is-maintenance .now-badge {
  background: #e9eef5;
  color: #6f7e91;
}

.mini-timeline-wrap {
  margin-top: 13px;
}

.timeline {
  position: relative;
}

.tl-track {
  position: relative;
  height: 7px;
  border-radius: 999px;
  background: #eef2f8;
  overflow: hidden;
}

.tl-seg {
  position: absolute;
  top: 0;
  height: 100%;
  border-radius: 2px;
}

.tl-course { background: #6a98ea; }
.tl-booking { background: #efb45d; }

.tl-now {
  position: absolute;
  top: -3px;
  width: 2px;
  height: 13px;
  border-radius: 2px;
  background: #ef4444;
  box-shadow: 0 0 0 2px rgba(239, 68, 68, 0.12);
}

.timeline-scale {
  display: flex;
  justify-content: space-between;
  margin-top: 5px;
  color: #a0aab8;
  font-size: 8.5px;
}

.room-card-foot {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  margin-top: 11px;
  padding-top: 10px;
  border-top: 1px solid #f0f3f8;
}

.link-btn {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 0;
  border: 0;
  background: none;
  color: #57708f;
  font-size: 10.5px;
  font-weight: 550;
  cursor: pointer;
}

.link-btn:hover {
  color: var(--blue);
}

.link-btn .icon {
  width: 12px;
  height: 12px;
  transition: transform 0.16s ease;
}

.link-btn:hover .icon {
  transform: translateX(2px);
}

.room-id {
  color: #b3bdcb;
  font-size: 8.5px;
  font-weight: 650;
  letter-spacing: 0.1em;
}

/* ------------------------------------------------------------------ */
/* 空状态、加载                                                        */
/* ------------------------------------------------------------------ */

.empty-state {
  grid-column: 1 / -1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  text-align: center;
  padding: 58px 24px;
  background: #fff;
  border: 1px dashed #d8e1ee;
  border-radius: var(--radius);
}

.empty-icon {
  width: 52px;
  height: 52px;
  display: grid;
  place-items: center;
  border-radius: 15px;
  background: #eff5fd;
  color: #6786aa;
  margin-bottom: 14px;
}

.empty-icon .icon {
  width: 23px;
  height: 23px;
}

.empty-state h3 {
  font-size: 15px;
  font-weight: 680;
}

.empty-state p {
  margin-top: 6px;
  max-width: 380px;
  color: var(--muted-2);
  font-size: 11.5px;
  line-height: 1.65;
}

.empty-action {
  margin-top: 16px;
}

.page-loading-inline {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 9px;
  min-height: 160px;
  color: #8290a4;
  font-size: 12px;
}

.error-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  padding: 80px 20px;
}

.error-icon {
  width: 56px;
  height: 56px;
  display: grid;
  place-items: center;
  border-radius: 16px;
  background: var(--red-soft);
  color: var(--red);
  margin-bottom: 15px;
}

.error-state h2 {
  font-size: 18px;
}

.error-state p {
  margin: 7px 0 20px;
  color: var(--muted);
  font-size: 12px;
}

.skeleton {
  position: relative;
  overflow: hidden;
  background: #e9eef6;
  border-radius: 12px;
}

.skeleton::after {
  content: "";
  position: absolute;
  inset: 0;
  transform: translateX(-100%);
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.65), transparent);
  animation: shimmer 1.35s infinite;
}

@keyframes shimmer {
  100% { transform: translateX(100%); }
}

.skeleton-stats {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 15px;
  margin-bottom: 24px;
}

.sk-card { height: 112px; }
.sk-room { height: 254px; }
.sk-borrow { height: 218px; }

.skeleton-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(302px, 1fr));
  gap: 15px;
}

/* ------------------------------------------------------------------ */
/* 弹窗                                                                */
/* ------------------------------------------------------------------ */

body.modal-open {
  overflow: hidden;
}

.modal-backdrop {
  position: fixed;
  inset: 0;
  z-index: 100;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
  background: rgba(8, 20, 37, 0.5);
  backdrop-filter: blur(4px);
  opacity: 0;
  transition: opacity 0.18s ease;
}

.modal-backdrop.is-visible {
  opacity: 1;
}

.modal {
  width: min(620px, 100%);
  max-height: calc(100vh - 48px);
  display: flex;
  flex-direction: column;
  background: #fff;
  border-radius: 19px;
  box-shadow: var(--shadow-lg);
  transform: translateY(14px) scale(0.985);
  transition: transform 0.2s ease;
}

.modal-backdrop.is-visible .modal {
  transform: translateY(0) scale(1);
}

.modal-sm { width: min(430px, 100%); }
.modal-md { width: min(620px, 100%); }
.modal-lg { width: min(790px, 100%); }

.modal-head {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 18px;
  padding: 22px 24px 16px;
  border-bottom: 1px solid #f0f3f8;
}

.modal-head h2 {
  font-size: 18px;
  font-weight: 700;
  letter-spacing: -0.02em;
}

.modal-head p {
  margin-top: 4px;
  font-size: 11px;
  color: var(--muted-2);
}

.modal-close {
  width: 32px;
  height: 32px;
  flex: 0 0 auto;
  display: grid;
  place-items: center;
  border: 1px solid var(--line);
  border-radius: 9px;
  background: #fff;
  color: #7e8b9e;
  cursor: pointer;
}

.modal-close:hover {
  background: #f5f8fc;
  color: var(--ink);
}

.modal-body {
  padding: 21px 24px;
  overflow-y: auto;
}

.modal-footer {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 9px;
  padding: 15px 24px 18px;
  border-top: 1px solid #f0f3f8;
}

.confirm-body {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  padding: 4px 4px 2px;
}

.confirm-icon {
  width: 46px;
  height: 46px;
  display: grid;
  place-items: center;
  border-radius: 13px;
  background: #eef5ff;
  color: var(--blue);
  margin-bottom: 13px;
}

.confirm-icon.is-danger {
  background: var(--red-soft);
  color: var(--red);
}

.confirm-body p {
  color: #53637a;
  font-size: 12.5px;
  line-height: 1.7;
}

/* 表单 */

.stack-form {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.form-grid {
  display: grid;
  gap: 14px;
}

.form-grid.two {
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

.form-section-label {
  margin: 3px 0 -4px;
  padding-bottom: 8px;
  border-bottom: 1px solid #eef2f7;
  color: #8a97a9;
  font-size: 9.5px;
  font-weight: 700;
  letter-spacing: 0.13em;
  text-transform: uppercase;
}

.form-error {
  display: none;
  padding: 10px 12px;
  border-radius: 9px;
  background: var(--red-soft);
  border: 1px solid #f4d1d3;
  color: #b02e33;
  font-size: 11.5px;
  line-height: 1.5;
}

.form-error.is-visible {
  display: block;
}

.form-note {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  padding: 11px 12px;
  border-radius: 9px;
  background: #f6f9fe;
  border: 1px solid #e7eef9;
  color: #6d7e95;
  font-size: 10.5px;
  line-height: 1.6;
}

.form-note .icon {
  width: 14px;
  height: 14px;
  margin-top: 1px;
  color: #7d97b8;
}

.field-hint {
  font-size: 10px;
  color: #97a3b4;
  margin-top: -1px;
}

.time-range {
  display: flex;
  align-items: center;
  gap: 7px;
  height: 45px;
  padding: 0 10px;
  background: #fbfcfe;
}

.time-range input {
  height: 35px;
  flex: 1;
  min-width: 0;
  border: 0;
  background: transparent;
  padding: 0;
  outline: none;
}

.time-range span {
  color: #9aa6b7;
  font-size: 10px;
}

.borrow-room-summary {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 13px 14px;
  border-radius: 12px;
  background: #f7faff;
  border: 1px solid #e5edfa;
}

.borrow-room-summary > div:nth-child(2) {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.borrow-room-summary strong {
  font-size: 13px;
  font-weight: 650;
}

.borrow-room-summary span:not(.status-pill) {
  margin-top: 2px;
  font-size: 10px;
  color: #8794a6;
}

.borrow-room-summary .status-pill {
  margin-left: auto;
}

/* 教室详情 */

.detail-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 14px;
  margin-bottom: 15px;
}

.detail-badges {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-wrap: wrap;
}

.soft-badge {
  display: inline-flex;
  align-items: center;
  height: 21px;
  padding: 0 8px;
  border-radius: 6px;
  background: #f2f6fb;
  border: 1px solid #e6ecf4;
  color: #65758c;
  font-size: 9.5px;
  font-weight: 550;
  white-space: nowrap;
}

.detail-room-code {
  color: #aab4c1;
  font-size: 9px;
  font-weight: 650;
  letter-spacing: 0.12em;
}

.detail-timeline-block {
  padding: 18px 16px 12px;
  border-radius: 12px;
  background: #f8fafd;
  border: 1px solid #edf1f8;
  margin-bottom: 19px;
}

.timeline-full .tl-track {
  height: 20px;
  background: #e9eef6;
}

.timeline-full .tl-seg {
  border-radius: 5px;
}

.timeline-full .tl-now {
  top: -5px;
  height: 30px;
}

.detail-section-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 9px;
}

.detail-section-head h3 {
  font-size: 13px;
  font-weight: 650;
}

.detail-section-head span {
  font-size: 9.5px;
  color: #9ba6b5;
}

.schedule-list {
  display: flex;
  flex-direction: column;
  gap: 7px;
}

.schedule-row {
  display: grid;
  grid-template-columns: 54px 16px minmax(0, 1fr) auto;
  align-items: center;
  gap: 9px;
  padding: 8px 10px;
  border: 1px solid #eef2f7;
  border-radius: 9px;
}

.schedule-time {
  display: flex;
  flex-direction: column;
  line-height: 1.2;
}

.schedule-time strong {
  font-size: 11px;
  font-weight: 650;
}

.schedule-time span {
  margin-top: 2px;
  font-size: 8.5px;
  color: #9aa6b6;
}

.schedule-rail {
  position: relative;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.schedule-rail::before {
  content: "";
  position: absolute;
  top: 0;
  bottom: 0;
  left: 50%;
  width: 1px;
  background: #e8edf5;
}

.schedule-rail i {
  position: relative;
  z-index: 1;
  width: 7px;
  height: 7px;
  border-radius: 50%;
  border: 2px solid #fff;
}

.rail-course { background: #5b8def; box-shadow: 0 0 0 1.5px #5b8def; }
.rail-booking { background: #efb45d; box-shadow: 0 0 0 1.5px #efb45d; }

.schedule-info {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.schedule-info strong {
  font-size: 11.5px;
  font-weight: 600;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.schedule-info span {
  margin-top: 2px;
  font-size: 9.5px;
  color: #8c98a9;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.tag {
  display: inline-flex;
  height: 19px;
  align-items: center;
  padding: 0 6px;
  border-radius: 5px;
  font-size: 8.5px;
  font-weight: 600;
}

.tag-course { color: #2c63bf; background: #eaf2ff; }
.tag-booking { color: #9a680f; background: #fff4df; }

.detail-empty {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 7px;
  padding: 24px;
  border: 1px dashed #dde5f0;
  border-radius: 10px;
  color: #98a5b6;
  font-size: 11px;
}

.detail-empty .icon {
  width: 15px;
  height: 15px;
}

.detail-equipment {
  display: flex;
  align-items: center;
  gap: 13px;
  margin-top: 18px;
  padding-top: 14px;
  border-top: 1px solid #f0f3f8;
}

.detail-equipment > span {
  flex: 0 0 auto;
  font-size: 10px;
  color: #8c99aa;
}

.detail-equipment div {
  display: flex;
  flex-wrap: wrap;
  gap: 5px;
}

.detail-equipment em {
  font-style: normal;
  padding: 3px 7px;
  border-radius: 5px;
  background: #f4f7fb;
  color: #66778e;
  font-size: 9px;
}

/* ------------------------------------------------------------------ */
/* 借用页                                                              */
/* ------------------------------------------------------------------ */

.borrow-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(315px, 1fr));
  gap: 15px;
}

.borrow-card {
  display: flex;
  flex-direction: column;
  padding: 17px 18px 14px;
  background: #fff;
  border: 1px solid var(--line);
  border-radius: var(--radius);
  box-shadow: var(--shadow);
}

.borrow-card.is-available {
  border-color: #cdeee2;
}

.borrow-card.is-unavailable {
  background: #fdfefe;
}

.borrow-card-head {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 10px;
}

.borrow-facts {
  display: flex;
  flex-wrap: wrap;
  gap: 5px 12px;
  margin-top: 12px;
  padding-bottom: 12px;
  border-bottom: 1px solid #f0f3f8;
}

.borrow-facts span {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  color: #7f8da0;
  font-size: 9.5px;
}

.borrow-facts .icon {
  width: 12px;
  height: 12px;
  color: #9aa8ba;
}

.borrow-state {
  display: flex;
  align-items: flex-start;
  gap: 9px;
  padding: 10px 11px;
  margin-top: 12px;
  border-radius: 10px;
}

.borrow-state > .icon {
  width: 16px;
  height: 16px;
  margin-top: 1px;
}

.borrow-state div {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.borrow-state strong {
  font-size: 11px;
  font-weight: 650;
}

.borrow-state span {
  margin-top: 2px;
  font-size: 9.5px;
  line-height: 1.5;
}

.borrow-state.is-free {
  background: #f0fbf6;
  border: 1px solid #dcf4ea;
  color: #087e5d;
}

.borrow-state.is-free span { color: #6a9285; }

.borrow-state.is-conflict {
  background: #fff8ec;
  border: 1px solid #f7e8c9;
  color: #9b680d;
}

.borrow-state.is-conflict span { color: #a18251; }

.borrow-state.is-muted {
  background: #f5f7fa;
  border: 1px solid #eaeef4;
  color: #7a8798;
}

.borrow-state.is-muted span { color: #9aa5b4; }

.borrow-card-foot {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  margin-top: auto;
  padding-top: 13px;
}

.borrow-time {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  color: #7f8da0;
  font-size: 9.5px;
  font-variant-numeric: tabular-nums;
}

.borrow-time .icon {
  width: 12px;
  height: 12px;
}

.borrow-skeleton {
  grid-template-columns: repeat(auto-fill, minmax(315px, 1fr));
}

/* ------------------------------------------------------------------ */
/* 借用记录                                                            */
/* ------------------------------------------------------------------ */

.booking-list-wrap {
  min-height: 200px;
}

.booking-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.booking-item {
  display: grid;
  grid-template-columns: 68px minmax(0, 1fr) auto;
  align-items: center;
  gap: 17px;
  padding: 15px 18px;
  background: #fff;
  border: 1px solid var(--line);
  border-radius: 15px;
  box-shadow: 0 4px 14px rgba(15, 36, 66, 0.035);
}

.booking-item.is-history {
  background: #fcfdff;
}

.booking-date {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 1px;
  min-height: 66px;
  border-radius: 11px;
  background: #f1f6ff;
  border: 1px solid #e3ecfa;
}

.booking-date strong {
  font-size: 24px;
  line-height: 1;
  font-weight: 720;
  letter-spacing: -0.04em;
  color: #285ab8;
}

.booking-date span {
  font-size: 8.5px;
  color: #6f86a8;
}

.booking-date em {
  margin-top: 3px;
  font-style: normal;
  font-size: 8.5px;
  color: #8d9db3;
}

.booking-main {
  min-width: 0;
}

.booking-title {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}

.booking-title h3 {
  font-size: 14px;
  font-weight: 680;
}

.relative-chip {
  display: inline-flex;
  height: 19px;
  align-items: center;
  padding: 0 7px;
  border-radius: 5px;
  background: #eef4ff;
  color: #3e6dbd;
  font-size: 8.5px;
  font-weight: 600;
}

.booking-meta {
  display: flex;
  flex-wrap: wrap;
  gap: 5px 15px;
  margin-top: 6px;
  color: #7e8c9f;
  font-size: 9.5px;
}

.booking-meta span {
  display: inline-flex;
  align-items: center;
  gap: 4px;
}

.booking-meta .icon {
  width: 11px;
  height: 11px;
  color: #9ba8b9;
}

.booking-purpose {
  margin-top: 7px;
  color: #596a80;
  font-size: 11px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.booking-purpose::before {
  content: "用途 · ";
  color: #9aa6b7;
  font-size: 9.5px;
}

.booking-action {
  display: flex;
  align-items: center;
}

/* ------------------------------------------------------------------ */
/* 表格                                                                */
/* ------------------------------------------------------------------ */

.table-card {
  background: #fff;
  border: 1px solid var(--line);
  border-radius: var(--radius);
  box-shadow: var(--shadow);
  overflow: hidden;
}

.table-scroll {
  width: 100%;
  overflow-x: auto;
}

.data-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 11.5px;
}

.data-table th {
  padding: 12px 17px;
  text-align: left;
  background: #f8fafd;
  border-bottom: 1px solid #edf1f7;
  color: #8693a6;
  font-size: 9px;
  font-weight: 650;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  white-space: nowrap;
}

.data-table td {
  padding: 13px 17px;
  border-bottom: 1px solid #f1f4f9;
  vertical-align: middle;
}

.data-table tbody tr:last-child td {
  border-bottom: 0;
}

.data-table tbody tr:hover {
  background: #fbfdff;
}

.course-cell,
.room-cell,
.time-cell {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.course-cell strong,
.room-cell strong {
  font-size: 12px;
  font-weight: 650;
}

.course-code {
  color: #8d9bb0;
  font-size: 8.5px;
  font-weight: 650;
  letter-spacing: 0.08em;
}

.room-cell span {
  color: #98a4b5;
  font-size: 9px;
}

.time-cell strong {
  font-size: 11px;
  font-weight: 550;
}

.time-cell span {
  color: #9ba6b5;
  font-size: 8.5px;
}

.person-cell {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 11.5px;
  white-space: nowrap;
}

.row-actions {
  display: flex;
  align-items: center;
  gap: 7px;
  white-space: nowrap;
}

.table-action {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  border: 0;
  background: none;
  padding: 4px 2px;
  color: #4b6b98;
  font-size: 10px;
  font-weight: 550;
  cursor: pointer;
}

.table-action:hover {
  color: var(--blue);
}

.table-action.danger {
  color: #c04a50;
}

.table-action.danger:hover {
  color: var(--red);
}

.table-action .icon {
  width: 12px;
  height: 12px;
}

.nowrap {
  white-space: nowrap;
}

.purpose-cell {
  max-width: 220px;
  color: #5d6d83;
  font-size: 10.5px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.actions-cell {
  text-align: right;
}

/* ------------------------------------------------------------------ */
/* 课程管理                                                            */
/* ------------------------------------------------------------------ */

.semester-banner {
  position: relative;
  display: grid;
  grid-template-columns: minmax(0, 1fr) 230px 170px;
  align-items: center;
  gap: 30px;
  padding: 23px 26px;
  margin-bottom: 22px;
  border-radius: var(--radius);
  background:
    radial-gradient(600px 200px at 100% 0%, rgba(55, 212, 196, 0.17), transparent 65%),
    linear-gradient(120deg, #0e2442, #17385f);
  color: #fff;
  box-shadow: 0 16px 40px rgba(13, 31, 57, 0.16);
  overflow: hidden;
}

.semester-banner::after {
  content: "";
  position: absolute;
  right: -50px;
  bottom: -90px;
  width: 270px;
  height: 270px;
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 50%;
  box-shadow: 0 0 0 42px rgba(255, 255, 255, 0.026), 0 0 0 84px rgba(255, 255, 255, 0.018);
}

.semester-main,
.semester-week,
.semester-facts {
  position: relative;
  z-index: 1;
}

.semester-kicker {
  font-size: 9px;
  letter-spacing: 0.15em;
  text-transform: uppercase;
  color: #72ddcf;
  font-weight: 650;
}

.semester-main h2 {
  margin-top: 8px;
  font-size: 19px;
  font-weight: 650;
  letter-spacing: -0.01em;
}

.semester-main p {
  margin-top: 5px;
  color: #95aecb;
  font-size: 10.5px;
}

.semester-week > span {
  font-size: 9px;
  color: #88a2c1;
  letter-spacing: 0.04em;
}

.semester-week strong {
  display: block;
  margin-top: 3px;
  font-size: 24px;
  line-height: 1.2;
  font-weight: 700;
}

.semester-week small {
  font-size: 9.5px;
  font-weight: 500;
  color: #85a0be;
}

.semester-week .term-bar {
  margin-top: 8px;
  background: rgba(255, 255, 255, 0.1);
}

.semester-facts {
  display: flex;
  gap: 26px;
  padding-left: 24px;
  border-left: 1px solid rgba(255, 255, 255, 0.1);
}

.semester-facts div {
  display: flex;
  flex-direction: column;
}

.semester-facts strong {
  font-size: 25px;
  font-weight: 700;
  line-height: 1.1;
}

.semester-facts span {
  margin-top: 3px;
  color: #8ca6c4;
  font-size: 9px;
}

/* ------------------------------------------------------------------ */
/* 教室管理                                                            */
/* ------------------------------------------------------------------ */

.classroom-admin-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(310px, 1fr));
  gap: 15px;
}

.classroom-admin-card {
  display: flex;
  flex-direction: column;
  padding: 17px 18px 13px;
  background: #fff;
  border: 1px solid var(--line);
  border-radius: var(--radius);
  box-shadow: var(--shadow);
}

.classroom-admin-card.is-maintenance {
  background: #fbfcfe;
}

.classroom-admin-head {
  display: flex;
  align-items: flex-start;
  gap: 11px;
}

.classroom-admin-title {
  min-width: 0;
  flex: 1;
}

.classroom-admin-title h3 {
  font-size: 14px;
  font-weight: 680;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.classroom-admin-title p {
  margin-top: 2px;
  color: #8b98aa;
  font-size: 10px;
}

.classroom-admin-head .status-pill {
  margin-left: auto;
}

.classroom-admin-facts {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 9px;
  margin-top: 15px;
  padding: 12px;
  border-radius: 10px;
  background: #f8fafd;
  border: 1px solid #eef2f8;
}

.classroom-admin-facts div {
  display: flex;
  flex-direction: column;
}

.classroom-admin-facts span {
  color: #97a3b4;
  font-size: 8.5px;
}

.classroom-admin-facts strong {
  margin-top: 2px;
  color: #43536a;
  font-size: 11.5px;
  font-weight: 600;
}

.classroom-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 5px;
  margin-top: 12px;
}

.classroom-tags span {
  padding: 3px 7px;
  border-radius: 5px;
  background: #f3f7fc;
  border: 1px solid #e9eff7;
  color: #6b7c92;
  font-size: 8.5px;
}

.classroom-admin-foot {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  margin-top: auto;
  padding-top: 13px;
  border-top: 1px solid #f1f4f9;
  color: #b0bac7;
  font-size: 8.5px;
  font-weight: 650;
  letter-spacing: 0.1em;
}

/* ------------------------------------------------------------------ */
/* 课表                                                                */
/* ------------------------------------------------------------------ */

.schedule-layout {
  display: grid;
  grid-template-columns: minmax(0, 1fr) 292px;
  gap: 18px;
  align-items: start;
}

.calendar-card {
  background: #fff;
  border: 1px solid var(--line);
  border-radius: var(--radius);
  box-shadow: var(--shadow);
  padding: 20px 20px 18px;
  min-width: 0;
}

.calendar-head {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 16px;
  margin-bottom: 16px;
}

.calendar-head h2 {
  font-size: 17px;
  font-weight: 700;
}

.calendar-head p {
  margin-top: 4px;
  color: #8b98aa;
  font-size: 10.5px;
}

.calendar-legend {
  display: flex;
  gap: 13px;
  color: #8b98aa;
  font-size: 9.5px;
}

.calendar-legend span {
  display: inline-flex;
  align-items: center;
  gap: 5px;
}

.week-board-scroll {
  width: 100%;
  overflow-x: auto;
  scrollbar-width: thin;
}

.week-board {
  min-width: 900px;
}

.week-head {
  display: grid;
  grid-template-columns: 60px repeat(7, minmax(0, 1fr));
  border-bottom: 1px solid #e9eef6;
}

.week-corner {
  display: flex;
  align-items: center;
  justify-content: center;
  color: #a1acba;
  font-size: 8.5px;
  letter-spacing: 0.08em;
}

.week-day-head {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2px;
  padding: 9px 4px 10px;
  border-left: 1px solid #f1f4f9;
}

.week-day-head span {
  color: #8d9aac;
  font-size: 9px;
}

.week-day-head strong {
  font-size: 12px;
  font-weight: 650;
  color: #46566d;
}

.week-day-head.is-today span {
  color: var(--blue);
}

.week-day-head.is-today strong {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 34px;
  height: 21px;
  border-radius: 6px;
  background: var(--blue);
  color: #fff;
  font-size: 10.5px;
}

.week-grid-body {
  display: grid;
  grid-template-columns: 60px repeat(7, minmax(0, 1fr));
  height: 840px;
}

.week-time-axis {
  position: relative;
  border-right: 1px solid #edf1f7;
}

.week-time-axis span {
  position: absolute;
  right: 8px;
  transform: translateY(-50%);
  color: #a0abba;
  font-size: 8.5px;
  font-variant-numeric: tabular-nums;
  white-space: nowrap;
}

.week-day-lane {
  position: relative;
  border-left: 1px solid #f1f4f9;
  background-image: repeating-linear-gradient(
    to bottom,
    transparent 0,
    transparent 59px,
    #f0f4f9 59px,
    #f0f4f9 60px
  );
}

.week-day-lane.is-today {
  background-color: #fbfdff;
}

.timetable-block {
  position: absolute;
  left: 5px;
  right: 5px;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 2px;
  padding: 7px 9px;
  overflow: hidden;
  text-align: left;
  border: 1px solid #dbe8ff;
  border-left: 3px solid #4f86e9;
  border-radius: 8px;
  background: linear-gradient(135deg, #edf4ff, #e7f0ff);
  color: #254b85;
  cursor: pointer;
  transition: 0.16s ease;
}

.timetable-block:hover {
  z-index: 5;
  transform: translateY(-1px);
  box-shadow: 0 8px 18px rgba(37, 99, 235, 0.15);
}

.block-time {
  color: #6a89b6;
  font-size: 8.5px;
  font-weight: 600;
  letter-spacing: 0.02em;
  white-space: nowrap;
}

.timetable-block strong {
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
  font-size: 11px;
  line-height: 1.35;
  font-weight: 650;
}

.block-room {
  display: inline-flex;
  align-items: center;
  gap: 2px;
  font-size: 8.5px;
  color: #5d7ca7;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 100%;
}

.block-room .icon {
  width: 9px;
  height: 9px;
}

.timetable-block small {
  font-size: 8px;
  color: #7b93b4;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 100%;
}

.timetable-block.is-compact {
  padding: 5px 8px;
}

.timetable-block.is-compact .block-time {
  font-size: 7px;
}

.timetable-block.is-compact strong {
  font-size: 9.5px;
  -webkit-line-clamp: 1;
}

.week-now-line {
  position: absolute;
  left: 0;
  right: 0;
  z-index: 4;
  height: 1.5px;
  background: #ef5b5f;
  pointer-events: none;
}

.week-now-line::before {
  content: "";
  position: absolute;
  left: -3px;
  top: -2.5px;
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: #ef5b5f;
}

.week-now-line span {
  position: absolute;
  right: 3px;
  top: -13px;
  padding: 1px 4px;
  border-radius: 3px;
  background: #ef5b5f;
  color: #fff;
  font-size: 6.5px;
  white-space: nowrap;
}

.schedule-side {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.side-card {
  padding: 18px;
  background: #fff;
  border: 1px solid var(--line);
  border-radius: 16px;
  box-shadow: var(--shadow);
}

.side-card-head {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 10px;
  padding-bottom: 12px;
  border-bottom: 1px solid #f0f3f8;
}

.side-card-head h3 {
  font-size: 13px;
  font-weight: 680;
}

.side-card-head span {
  color: #96a2b3;
  font-size: 8.5px;
}

.today-list {
  display: flex;
  flex-direction: column;
  gap: 2px;
  margin-top: 5px;
}

.today-item {
  display: grid;
  grid-template-columns: 44px 10px minmax(0, 1fr);
  align-items: center;
  gap: 7px;
  padding: 10px 4px;
  border: 0;
  border-bottom: 1px solid #f4f6fa;
  background: none;
  text-align: left;
  cursor: pointer;
}

.today-item:last-child {
  border-bottom: 0;
}

.today-item:hover .today-copy strong {
  color: var(--blue);
}

.today-time {
  display: flex;
  flex-direction: column;
  font-size: 10px;
  font-weight: 650;
  color: #3f5473;
  font-variant-numeric: tabular-nums;
}

.today-time em {
  margin-top: 1px;
  font-style: normal;
  font-size: 7.5px;
  font-weight: 500;
  color: #a0abba;
}

.today-line {
  position: relative;
  height: 30px;
}

.today-line::before {
  content: "";
  position: absolute;
  top: 50%;
  left: 50%;
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: #5b8def;
  transform: translate(-50%, -50%);
  box-shadow: 0 0 0 3px #eaf1ff;
}

.today-copy {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.today-copy strong {
  font-size: 10.5px;
  font-weight: 600;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  transition: color 0.15s ease;
}

.today-copy small {
  margin-top: 2px;
  font-size: 8px;
  color: #939faf;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.side-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  padding: 30px 10px 20px;
  color: #a1adbc;
}

.side-empty .icon {
  width: 24px;
  height: 24px;
  color: #8ba6c4;
}

.side-empty p {
  margin-top: 8px;
  font-size: 11px;
  font-weight: 600;
  color: #68798f;
}

.side-empty span {
  margin-top: 3px;
  font-size: 8.5px;
}

.term-side-card {
  background:
    radial-gradient(300px 180px at 100% 0%, rgba(47, 212, 191, 0.14), transparent 65%),
    linear-gradient(130deg, #112a4a, #0d1f39);
  border-color: transparent;
  color: #fff;
}

.term-side-card h3 {
  margin-top: 7px;
  font-size: 13.5px;
  font-weight: 620;
  line-height: 1.5;
}

.term-side-card p {
  margin-top: 4px;
  color: #8ba4c2;
  font-size: 9.5px;
}

.term-progress {
  height: 4px;
  margin-top: 13px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.1);
  overflow: hidden;
}

.term-progress span {
  display: block;
  height: 100%;
  border-radius: inherit;
  background: linear-gradient(90deg, #39d6c3, #6a9fff);
}

.term-side-foot {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-top: 13px;
  padding-top: 11px;
  border-top: 1px solid rgba(255, 255, 255, 0.09);
}

.term-side-foot span {
  color: #7e97b7;
  font-size: 8px;
}

.term-side-foot strong {
  color: #b9cce2;
  font-size: 8.5px;
  font-weight: 550;
}

.schedule-empty-layout {
  display: grid;
  grid-template-columns: minmax(0, 1fr) 292px;
  gap: 18px;
  align-items: start;
}

.schedule-empty-layout .empty-state {
  min-height: 380px;
}

.course-detail-hero {
  padding: 17px;
  border-radius: 13px;
  background: #f4f8ff;
  border: 1px solid #e5eefb;
  margin-bottom: 16px;
}

.course-detail-code {
  display: inline-flex;
  height: 19px;
  align-items: center;
  padding: 0 7px;
  border-radius: 5px;
  background: #e5efff;
  color: #376cc4;
  font-size: 8px;
  font-weight: 650;
  letter-spacing: 0.07em;
}

.course-detail-hero h3 {
  margin-top: 7px;
  font-size: 18px;
  font-weight: 700;
}

.course-detail-hero p {
  margin-top: 4px;
  color: #7c8ea7;
  font-size: 10px;
}

.detail-info-list {
  display: flex;
  flex-direction: column;
  gap: 1px;
}

.detail-info-list > div {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 15px;
  padding: 10px 2px;
  border-bottom: 1px solid #f2f5f9;
}

.detail-info-list > div:last-child {
  border-bottom: 0;
}

.detail-info-list span {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  color: #8b98aa;
  font-size: 10px;
}

.detail-info-list span .icon {
  width: 13px;
  height: 13px;
}

.detail-info-list strong {
  color: #3f4f66;
  font-size: 10.5px;
  font-weight: 600;
  text-align: right;
}

/* ------------------------------------------------------------------ */
/* Toast                                                               */
/* ------------------------------------------------------------------ */

.toast-root {
  position: fixed;
  right: 22px;
  bottom: 22px;
  z-index: 200;
  display: flex;
  flex-direction: column;
  gap: 8px;
  pointer-events: none;
}

.toast {
  display: flex;
  align-items: center;
  gap: 9px;
  max-width: 380px;
  padding: 11px 15px;
  border-radius: 11px;
  background: #10233f;
  color: #fff;
  font-size: 11.5px;
  box-shadow: 0 14px 32px rgba(9, 25, 48, 0.25);
  opacity: 0;
  transform: translateY(8px);
  transition: 0.22s ease;
}

.toast.is-visible {
  opacity: 1;
  transform: translateY(0);
}

.toast-error {
  background: #9f2a30;
}

.toast-info {
  background: #24508f;
}

.toast-icon {
  display: flex;
}

.toast-icon .icon {
  width: 15px;
  height: 15px;
}

/* ------------------------------------------------------------------ */
/* 响应式                                                              */
/* ------------------------------------------------------------------ */

@media (max-width: 1280px) {
  .filter-grid {
    grid-template-columns: repeat(3, minmax(0, 1fr));
  }

  .filter-grid .btn-now {
    width: 100%;
  }

  .borrow-filter-grid {
    grid-template-columns: repeat(3, minmax(0, 1fr));
  }

  .borrow-filter-grid .btn-search-rooms {
    width: 100%;
  }

  .schedule-layout,
  .schedule-empty-layout {
    grid-template-columns: minmax(0, 1fr) 270px;
  }

  .semester-banner {
    grid-template-columns: minmax(0, 1fr) 190px;
  }

  .semester-facts {
    display: none;
  }
}

@media (max-width: 1080px) {
  .stats-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .schedule-layout,
  .schedule-empty-layout {
    grid-template-columns: minmax(0, 1fr);
  }

  .schedule-side {
    display: grid;
    grid-template-columns: 1fr 1fr;
  }

  .page-content {
    padding: 26px 24px 48px;
  }

  .topbar {
    padding: 0 24px;
  }

  .login-page {
    grid-template-columns: minmax(0, 1fr) 430px;
  }

  .login-visual {
    padding: 38px 38px 24px;
  }

  .login-features {
    gap: 20px;
  }
}

@media (max-width: 900px) {
  .sidebar {
    transform: translateX(-104%);
    transition: transform 0.24s ease;
    box-shadow: 20px 0 60px rgba(4, 15, 30, 0.3);
  }

  .app-shell.sidebar-open .sidebar {
    transform: translateX(0);
  }

  .sidebar-scrim {
    display: block;
    position: fixed;
    inset: 0;
    z-index: 35;
    background: rgba(6, 17, 32, 0.48);
    backdrop-filter: blur(2px);
    opacity: 0;
    pointer-events: none;
    transition: opacity 0.22s ease;
  }

  .app-shell.sidebar-open .sidebar-scrim {
    opacity: 1;
    pointer-events: auto;
  }

  .main-shell {
    margin-left: 0;
  }

  .mobile-menu {
    display: inline-grid;
  }

  .login-page {
    grid-template-columns: 1fr;
  }

  .login-visual {
    min-height: 420px;
    padding: 32px 34px 26px;
  }

  .login-copy {
    margin-top: 40px;
  }

  .login-copy h1 {
    font-size: 42px;
  }

  .login-showcase {
    display: none;
  }

  .login-panel {
    padding: 42px 30px;
  }

  .login-foot {
    margin-top: 30px;
  }

  .semester-banner {
    grid-template-columns: minmax(0, 1fr);
  }

  .semester-week {
    display: none;
  }
}

@media (max-width: 760px) {
  .page-content {
    padding: 20px 16px 42px;
  }

  .topbar {
    height: 66px;
    padding: 0 16px;
  }

  .topbar-heading h1 {
    font-size: 15px;
  }

  .topbar-heading span,
  .topbar-role {
    display: none;
  }

  .live-clock span {
    display: none;
  }

  .page-heading {
    align-items: flex-start;
    flex-direction: column;
    gap: 14px;
  }

  .page-heading h1 {
    font-size: 25px;
  }

  .heading-actions {
    width: 100%;
    flex-wrap: wrap;
  }

  .heading-actions .btn {
    flex: 1;
  }

  .filter-grid,
  .course-filter-grid,
  .admin-booking-filter,
  .classroom-filter-grid,
  .borrow-filter-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .filter-grid .search-field,
  .course-filter-grid .search-field,
  .admin-booking-filter .search-field,
  .classroom-filter-grid .search-field {
    grid-column: 1 / -1;
  }

  .filter-grid .btn-now {
    grid-column: 1 / -1;
  }

  .filter-bottom {
    align-items: flex-start;
    flex-direction: column;
  }

  .filter-count {
    margin-left: 0;
  }

  .segmented {
    width: 100%;
    overflow-x: auto;
  }

  .segment {
    flex: 1;
    white-space: nowrap;
  }

  .stats-grid,
  .stats-grid-mini,
  .skeleton-stats {
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 10px;
  }

  .stat-card {
    padding: 14px;
  }

  .stat-value {
    font-size: 23px;
  }

  .stat-card p {
    display: none;
  }

  .section-head {
    align-items: flex-start;
    flex-direction: column;
    gap: 8px;
  }

  .room-grid,
  .borrow-grid,
  .classroom-admin-grid,
  .skeleton-grid {
    grid-template-columns: 1fr;
  }

  .booking-item {
    grid-template-columns: 58px minmax(0, 1fr);
    gap: 12px;
  }

  .booking-action {
    grid-column: 2;
  }

  .booking-action .btn {
    width: 100%;
  }

  .booking-date {
    min-height: 60px;
  }

  .booking-date strong {
    font-size: 21px;
  }

  .schedule-side {
    grid-template-columns: 1fr;
  }

  .calendar-card {
    padding: 15px 13px;
  }

  .calendar-head {
    align-items: flex-start;
    flex-direction: column;
  }

  .semester-banner {
    padding: 19px;
  }

  .semester-main h2 {
    font-size: 16px;
  }

  .form-grid.two {
    grid-template-columns: 1fr;
  }

  .modal-backdrop {
    padding: 12px;
    align-items: flex-end;
  }

  .modal {
    max-height: calc(100vh - 24px);
    border-radius: 18px;
  }

  .modal-head,
  .modal-body,
  .modal-footer {
    padding-left: 18px;
    padding-right: 18px;
  }

  .login-visual {
    min-height: 350px;
    padding: 28px 24px 22px;
  }

  .login-copy {
    margin-top: 30px;
  }

  .login-copy h1 {
    font-size: 35px;
  }

  .login-copy p {
    font-size: 12.5px;
  }

  .login-features {
    margin-top: 24px;
    gap: 14px;
  }

  .login-feature:nth-child(3) {
    display: none;
  }

  .login-panel {
    padding: 34px 22px;
  }

  .demo-grid {
    grid-template-columns: 1fr;
  }

  .demo-account {
    flex-direction: row;
    align-items: center;
    gap: 8px;
  }

  .demo-role {
    flex: 0 0 48px;
  }

  .demo-account small {
    margin-left: auto;
  }

  .detail-timeline-block {
    padding: 15px 12px 10px;
  }

  .schedule-row {
    grid-template-columns: 46px 12px minmax(0, 1fr);
  }

  .schedule-row .tag {
    display: none;
  }

  .toast-root {
    left: 14px;
    right: 14px;
    bottom: 14px;
  }

  .toast {
    max-width: none;
  }
}

@media (max-width: 480px) {
  .stats-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .stat-card p {
    display: none;
  }

  .filter-grid,
  .course-filter-grid,
  .admin-booking-filter,
  .classroom-filter-grid,
  .borrow-filter-grid {
    grid-template-columns: 1fr;
  }

  .filter-grid .search-field,
  .course-filter-grid .search-field,
  .admin-booking-filter .search-field,
  .classroom-filter-grid .search-field {
    grid-column: auto;
  }

  .login-copy h1 {
    font-size: 31px;
  }

  .login-features {
    display: none;
  }

  .schedule-layout {
    margin: 0 -5px;
  }
}
`````


## `static/js/core.js`

`````javascript
/* 核心工具：图标、状态、API、弹窗与通用格式化。 */

export const $ = (selector, root = document) => root.querySelector(selector);
export const $$ = (selector, root = document) => [...root.querySelectorAll(selector)];

export const state = {
  user: null,
  csrf: null,
  meta: null,
  page: "overview",
  overview: null,
  filters: null,
  borrowFilters: null,
  followNow: false,
  pollTimer: null,
  clockTimer: null,
  lastPage: null,
};

const PATHS = {
  grid: '<rect x="3" y="3" width="7" height="7" rx="1.8"/><rect x="14" y="3" width="7" height="7" rx="1.8"/><rect x="3" y="14" width="7" height="7" rx="1.8"/><rect x="14" y="14" width="7" height="7" rx="1.8"/>',
  calendar: '<rect x="3" y="5" width="18" height="16" rx="3"/><path d="M8 3v4M16 3v4M3 10h18"/><path d="M8 14h2M14 14h2M8 18h2M14 18h2"/>',
  door: '<path d="M4 21h16"/><path d="M6 21V4a1 1 0 0 1 1-1h9v18"/><path d="M9 12h.01"/><path d="M19 8v13"/>',
  clipboard: '<rect x="5" y="4" width="14" height="17" rx="2.5"/><path d="M9 4.5V3h6v1.5"/><path d="M9 9h6M9 13h6M9 17h4"/>',
  book: '<path d="M4 5.5A2.5 2.5 0 0 1 6.5 3H20v15H6.5A2.5 2.5 0 0 0 4 20.5z"/><path d="M4 20.5A2.5 2.5 0 0 1 6.5 18H20v3H6.5A2.5 2.5 0 0 1 4 20.5z"/>',
  building: '<rect x="4" y="3" width="16" height="18" rx="2.5"/><path d="M9 8h1M14 8h1M9 12h1M14 12h1M9 16h1M14 16h1"/><path d="M9 21v-2h6v2"/>',
  logout: '<path d="M10 17l5-5-5-5M15 12H3"/><path d="M14 4h4a2 2 0 0 1 2 2v12a2 2 0 0 1-2 2h-4"/>',
  search: '<circle cx="11" cy="11" r="7"/><path d="m20 20-4-4"/>',
  plus: '<path d="M12 5v14M5 12h14"/>',
  clock: '<circle cx="12" cy="12" r="9"/><path d="M12 7v5l3.5 2"/>',
  users: '<path d="M16 21v-2a4 4 0 0 0-4-4H6a4 4 0 0 0-4 4v2"/><circle cx="9" cy="7" r="4"/><path d="M22 21v-2a4 4 0 0 0-3-3.87M16 3.13a4 4 0 0 1 0 7.75"/>',
  arrow: '<path d="M5 12h14M13 6l6 6-6 6"/>',
  x: '<path d="M18 6 6 18M6 6l12 12"/>',
  check: '<path d="m5 12 4.5 4.5L19 7"/>',
  alert: '<path d="M10.3 3.9 2.4 17.4A2 2 0 0 0 4.1 20h15.8a2 2 0 0 0 1.7-2.6L13.7 3.9a2 2 0 0 0-3.4 0z"/><path d="M12 9v4M12 17h.01"/>',
  info: '<circle cx="12" cy="12" r="9"/><path d="M12 11v5M12 8h.01"/>',
  refresh: '<path d="M20 11a8 8 0 1 0-2.3 6.7"/><path d="M20 4v7h-7"/>',
  settings: '<circle cx="12" cy="12" r="3"/><path d="M19.4 15a1.7 1.7 0 0 0 .3 1.9l.1.1a2 2 0 1 1-2.8 2.8l-.1-.1a1.7 1.7 0 0 0-1.9-.3 1.7 1.7 0 0 0-1 1.6V21a2 2 0 1 1-4 0v-.1a1.7 1.7 0 0 0-1.1-1.6 1.7 1.7 0 0 0-1.9.3l-.1.1a2 2 0 1 1-2.8-2.8l.1-.1a1.7 1.7 0 0 0 .3-1.9 1.7 1.7 0 0 0-1.6-1H3a2 2 0 1 1 0-4h.1a1.7 1.7 0 0 0 1.6-1.1 1.7 1.7 0 0 0-.3-1.9l-.1-.1a2 2 0 1 1 2.8-2.8l.1.1a1.7 1.7 0 0 0 1.9.3H9a1.7 1.7 0 0 0 1-1.6V3a2 2 0 1 1 4 0v.1a1.7 1.7 0 0 0 1 1.6 1.7 1.7 0 0 0 1.9-.3l.1-.1a2 2 0 1 1 2.8 2.8l-.1.1a1.7 1.7 0 0 0-.3 1.9V9a1.7 1.7 0 0 0 1.6 1H21a2 2 0 1 1 0 4h-.1a1.7 1.7 0 0 0-1.6 1z"/>',
  trash: '<path d="M3 6h18M8 6V4h8v2M19 6l-1 14H6L5 6M10 11v5M14 11v5"/>',
  edit: '<path d="m14.5 4.5 5 5L8 21H3v-5z"/><path d="m12.5 6.5 5 5"/>',
  chevron: '<path d="m6 9 6 6 6-6"/>',
  lock: '<rect x="4" y="10" width="16" height="11" rx="2.5"/><path d="M8 10V7a4 4 0 0 1 8 0v3"/>',
  user: '<circle cx="12" cy="8" r="4"/><path d="M4 21a8 8 0 0 1 16 0"/>',
  spark: '<path d="m12 3 1.4 4.6L18 9l-4.6 1.4L12 15l-1.4-4.6L6 9l4.6-1.4z"/><path d="M19 15l.7 2.3L22 18l-2.3.7L19 21l-.7-2.3L16 18l2.3-.7z"/>',
  pin: '<path d="M20 10c0 6-8 11-8 11S4 16 4 10a8 8 0 1 1 16 0z"/><circle cx="12" cy="10" r="2.5"/>',
  monitor: '<rect x="3" y="4" width="18" height="12" rx="2"/><path d="M8 20h8M12 16v4"/>',
  shield: '<path d="M12 22s8-3.5 8-10V5l-8-3-8 3v7c0 6.5 8 10 8 10z"/><path d="m9 12 2 2 4-4"/>',
  arrowLeft: '<path d="M19 12H5M11 18l-6-6 6-6"/>',
  dot: '<circle cx="12" cy="12" r="4" fill="currentColor" stroke="none"/>',
};

export function icon(name, className = "") {
  return `<svg class="icon ${className}" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">${PATHS[name] || ""}</svg>`;
}

export function esc(value) {
  return String(value ?? "").replace(/[&<>"']/g, (char) => ({
    "&": "&amp;",
    "<": "&lt;",
    ">": "&gt;",
    '"': "&quot;",
    "'": "&#39;",
  })[char]);
}

export function initial(name) {
  return esc(String(name || "?").trim().slice(0, 1));
}

export const ROLE_LABELS = { student: "学生", teacher: "教师", admin: "管理员" };

export class ApiError extends Error {
  constructor(message, status = 0, details = {}) {
    super(message);
    this.name = "ApiError";
    this.status = status;
    this.details = details;
  }
}

export async function api(path, { method = "GET", body } = {}) {
  const options = {
    method,
    credentials: "same-origin",
    headers: { Accept: "application/json" },
  };
  if (body !== undefined) {
    options.headers["Content-Type"] = "application/json";
    options.body = JSON.stringify(body);
  }
  if (method !== "GET" && state.csrf) {
    options.headers["X-CSRF-Token"] = state.csrf;
  }

  const response = await fetch(`/api${path}`, options);
  let data = {};
  try {
    data = await response.json();
  } catch {
    data = {};
  }
  if (!response.ok) {
    if (response.status === 401 && state.user) {
      state.user = null;
      state.csrf = null;
      window.dispatchEvent(new CustomEvent("auth-expired"));
    }
    throw new ApiError(data.error || `请求失败（${response.status}）`, response.status, data.details || {});
  }
  return data;
}

export async function loadMeta() {
  state.meta = await api("/meta");
  return state.meta;
}

/* ------------------------------------------------------------------ */
/* 日期与时间                                                          */
/* ------------------------------------------------------------------ */

export function pad(value) {
  return String(value).padStart(2, "0");
}

export function dateString(date = new Date()) {
  return `${date.getFullYear()}-${pad(date.getMonth() + 1)}-${pad(date.getDate())}`;
}

export function parseDate(value) {
  const [year, month, day] = String(value).split("-").map(Number);
  return new Date(year, month - 1, day);
}

export function addDays(value, amount) {
  const date = typeof value === "string" ? parseDate(value) : new Date(value);
  date.setDate(date.getDate() + amount);
  return date;
}

export const WEEKDAYS = ["周一", "周二", "周三", "周四", "周五", "周六", "周日"];

export function weekdayLabel(value) {
  const date = typeof value === "string" ? parseDate(value) : value;
  return WEEKDAYS[(date.getDay() + 6) % 7];
}

export function formatDateCN(value, withWeekday = true) {
  const date = typeof value === "string" ? parseDate(value) : value;
  return `${date.getFullYear()}年${date.getMonth() + 1}月${date.getDate()}日${withWeekday ? ` · ${weekdayLabel(date)}` : ""}`;
}

export function formatShortDate(value) {
  const date = typeof value === "string" ? parseDate(value) : value;
  return `${date.getMonth() + 1}月${date.getDate()}日`;
}

export function relativeDate(value) {
  const today = dateString();
  const tomorrow = dateString(addDays(new Date(), 1));
  if (value === today) return "今天";
  if (value === tomorrow) return "明天";
  return formatShortDate(value);
}

export function timeToMinutes(value) {
  if (!value) return 0;
  const [hour, minute] = String(value).split(":").map(Number);
  return hour * 60 + (minute || 0);
}

export function minutesToTime(value) {
  return `${pad(Math.floor(value / 60))}:${pad(value % 60)}`;
}

export function nowMinutes() {
  const now = new Date();
  return now.getHours() * 60 + now.getMinutes();
}

export function nowTime() {
  return minutesToTime(nowMinutes());
}

export function formatDuration(start, end) {
  const minutes = timeToMinutes(end) - timeToMinutes(start);
  if (minutes <= 0) return "—";
  const hours = Math.floor(minutes / 60);
  const rest = minutes % 60;
  if (!hours) return `${rest} 分钟`;
  if (!rest) return `${hours} 小时`;
  return `${hours} 小时 ${rest} 分`;
}

export function rangeOverlaps(start, end, otherStart, otherEnd) {
  return timeToMinutes(start) < timeToMinutes(otherEnd) && timeToMinutes(end) > timeToMinutes(otherStart);
}

export function roundUpTo(value, step = 5) {
  return Math.ceil(value / step) * step;
}

export function suggestedBorrowRange(room, fallbackTime) {
  let start = timeToMinutes(fallbackTime);
  if (room?.current) start = room.current.end_min;
  if (start < 480 || start >= 1320) start = 600;
  start = Math.min(1320, roundUpTo(start, 5));
  if (start >= 1260) start = 600;
  let end = Math.min(1320, start + 120);
  if (end - start < 30) {
    start = 600;
    end = 720;
  }
  return { start: minutesToTime(start), end: minutesToTime(end) };
}

export function defaultBorrowFilters() {
  const now = new Date();
  const current = now.getHours() * 60 + now.getMinutes();
  if (current < 720) {
    return { date: dateString(now), start: "12:10", end: "14:00", building: "", capacity: "", q: "" };
  }
  if (current >= 1230) {
    const tomorrow = addDays(now, 1);
    return {
      date: dateString(tomorrow),
      start: "12:10",
      end: "14:00",
      building: "",
      capacity: "",
      q: "",
    };
  }
  const start = Math.min(1260, roundUpTo(current + 30, 30));
  const end = Math.min(1320, start + 120);
  if (end - start < 60) {
    const tomorrow = addDays(now, 1);
    return {
      date: dateString(tomorrow),
      start: "12:10",
      end: "14:00",
      building: "",
      capacity: "",
      q: "",
    };
  }
  return {
    date: dateString(now),
    start: minutesToTime(start),
    end: minutesToTime(end),
    building: "",
    capacity: "",
    q: "",
  };
}

export function timestampLabel(value) {
  if (!value) return "--:--:--";
  const parts = String(value).split(" ");
  return parts[1] || value;
}

/* ------------------------------------------------------------------ */
/* 通用 UI                                                            */
/* ------------------------------------------------------------------ */

export function pageSkeleton(cards = 4) {
  return `
    <div class="skeleton-stats">
      ${Array.from({ length: 4 }, () => '<div class="skeleton sk-card"></div>').join("")}
    </div>
    <div class="skeleton-grid">
      ${Array.from({ length: cards }, () => '<div class="skeleton sk-room"></div>').join("")}
    </div>`;
}

export function emptyState(iconName, title, text, action = "") {
  return `
    <div class="empty-state">
      <div class="empty-icon">${icon(iconName)}</div>
      <h3>${esc(title)}</h3>
      <p>${esc(text)}</p>
      ${action ? `<div class="empty-action">${action}</div>` : ""}
    </div>`;
}

export function toast(message, type = "success") {
  const root = $("#toast-root");
  if (!root) return;
  const item = document.createElement("div");
  item.className = `toast toast-${type}`;
  const iconName = type === "error" ? "alert" : type === "info" ? "info" : "check";
  item.innerHTML = `<span class="toast-icon">${icon(iconName)}</span><span>${esc(message)}</span>`;
  root.appendChild(item);
  requestAnimationFrame(() => item.classList.add("is-visible"));
  setTimeout(() => {
    item.classList.remove("is-visible");
    setTimeout(() => item.remove(), 260);
  }, 3600);
}

export function formError(form, message) {
  const box = form?.querySelector(".form-error");
  if (!box) {
    toast(message, "error");
    return;
  }
  box.textContent = message;
  box.classList.add("is-visible");
  box.scrollIntoView({ block: "nearest", behavior: "smooth" });
}

export function clearFormError(form) {
  const box = form?.querySelector(".form-error");
  if (box) {
    box.textContent = "";
    box.classList.remove("is-visible");
  }
}

export function setButtonLoading(button, loading, label = "处理中...") {
  if (!button) return;
  if (loading) {
    button.dataset.original = button.innerHTML;
    button.disabled = true;
    button.innerHTML = `<span class="btn-spinner"></span>${esc(label)}`;
  } else {
    button.disabled = false;
    if (button.dataset.original) {
      button.innerHTML = button.dataset.original;
      delete button.dataset.original;
    }
  }
}

/* ------------------------------------------------------------------ */
/* 弹窗                                                               */
/* ------------------------------------------------------------------ */

let activeModal = null;

export function openModal({
  title,
  subtitle = "",
  body = "",
  footer = "",
  size = "",
  onMount,
  onClose,
}) {
  closeActiveModal();

  const root = $("#modal-root");
  const backdrop = document.createElement("div");
  backdrop.className = "modal-backdrop";
  backdrop.innerHTML = `
    <div class="modal ${esc(size)}" role="dialog" aria-modal="true" aria-label="${esc(title)}">
      <div class="modal-head">
        <div>
          <h2>${esc(title)}</h2>
          ${subtitle ? `<p>${esc(subtitle)}</p>` : ""}
        </div>
        <button class="modal-close" type="button" aria-label="关闭">${icon("x")}</button>
      </div>
      <div class="modal-body">${body}</div>
      ${footer ? `<div class="modal-footer">${footer}</div>` : ""}
    </div>`;
  root.appendChild(backdrop);
  document.body.classList.add("modal-open");

  let closed = false;
  const close = (result) => {
    if (closed) return;
    closed = true;
    document.removeEventListener("keydown", onKeydown);
    backdrop.classList.remove("is-visible");
    document.body.classList.remove("modal-open");
    setTimeout(() => backdrop.remove(), 180);
    activeModal = null;
    if (onClose) onClose(result);
  };
  const onKeydown = (event) => {
    if (event.key === "Escape") close(false);
  };
  document.addEventListener("keydown", onKeydown);
  backdrop.querySelector(".modal-close")?.addEventListener("click", () => close(false));
  backdrop.addEventListener("mousedown", (event) => {
    if (event.target === backdrop) close(false);
  });

  activeModal = { backdrop, close };
  requestAnimationFrame(() => backdrop.classList.add("is-visible"));
  setTimeout(() => {
    backdrop.querySelector("[autofocus]")?.focus();
  }, 80);
  if (onMount) onMount(backdrop.querySelector(".modal"), close);
  return { element: backdrop.querySelector(".modal"), close };
}

export function closeActiveModal() {
  if (activeModal) {
    activeModal.close(false);
    activeModal = null;
  }
}

export function confirmDialog({
  title,
  message,
  confirmText = "确认",
  cancelText = "取消",
  danger = false,
}) {
  return new Promise((resolve) => {
    let settled = false;
    const finish = (value) => {
      if (settled) return;
      settled = true;
      resolve(value);
    };
    openModal({
      title,
      size: "modal-sm",
      body: `<div class="confirm-body"><div class="confirm-icon ${danger ? "is-danger" : ""}">${icon(danger ? "alert" : "info")}</div><p>${esc(message)}</p></div>`,
      footer: `
        <button type="button" class="btn btn-ghost" data-confirm-cancel>${esc(cancelText)}</button>
        <button type="button" class="btn ${danger ? "btn-danger" : "btn-primary"}" data-confirm-ok>${esc(confirmText)}</button>`,
      onMount(element, close) {
        element.querySelector("[data-confirm-cancel]").addEventListener("click", () => {
          finish(false);
          close(false);
        });
        element.querySelector("[data-confirm-ok]").addEventListener("click", () => {
          finish(true);
          close(true);
        });
      },
      onClose(result) {
        finish(result === true);
      },
    });
  });
}

/* ------------------------------------------------------------------ */
/* 业务状态文案                                                        */
/* ------------------------------------------------------------------ */

export function roomStateMeta(roomState) {
  return {
    free: { label: "空闲中", tone: "free" },
    busy: { label: "使用中", tone: "busy" },
    maintenance: { label: "维护中", tone: "maintenance" },
  }[roomState] || { label: "未知", tone: "muted" };
}

export function bookingStateMeta(bookingState) {
  return {
    active: { label: "已生效", tone: "free" },
    ended: { label: "已结束", tone: "muted" },
    cancelled: { label: "已取消", tone: "muted" },
  }[bookingState] || { label: "未知", tone: "muted" };
}

export function roomAvailabilityForRange(room, start, end) {
  if (!room || room.status !== "active") {
    return { available: false, conflicts: room?.intervals || [] };
  }
  const conflicts = (room.intervals || []).filter((item) =>
    rangeOverlaps(start, end, item.start, item.end)
  );
  return { available: conflicts.length === 0, conflicts };
}

export function roomLabel(room) {
  return room?.name || "";
}

export function getQuery(name) {
  return new URLSearchParams(location.search).get(name);
}
`````


## `static/js/app.js`

`````javascript
/* 应用入口：登录、导航外壳与路由。 */

import {
  $,
  $$,
  state,
  icon,
  esc,
  initial,
  ROLE_LABELS,
  api,
  loadMeta,
  confirmDialog,
  toast,
  setButtonLoading,
  closeActiveModal,
} from "./core.js";
import { renderOverviewPage } from "./pages/overview.js";
import { renderBorrowPage, renderMyBookingsPage, renderAdminBookingsPage } from "./pages/booking.js";
import { renderCoursesPage } from "./pages/courses.js";
import { renderClassroomsPage } from "./pages/classrooms.js";
import { renderSchedulePage } from "./pages/schedule.js";

const NAV = {
  student: [
    { id: "overview", label: "教室总览", hint: "实时空闲情况", icon: "grid" },
    { id: "schedule", label: "我的课表", hint: "本周课程安排", icon: "calendar" },
  ],
  teacher: [
    { id: "overview", label: "教室总览", hint: "实时空闲情况", icon: "grid" },
    { id: "borrow", label: "借用教室", hint: "查找并借用空闲教室", icon: "door" },
    { id: "bookings", label: "我的借用", hint: "查看借用记录", icon: "clipboard" },
    { id: "schedule", label: "我的课表", hint: "本周授课安排", icon: "calendar" },
  ],
  admin: [
    { id: "overview", label: "教室总览", hint: "实时空闲情况", icon: "grid" },
    { id: "courses", label: "课程管理", hint: "课程与排课维护", icon: "book" },
    { id: "classrooms", label: "教室管理", hint: "教室档案与状态", icon: "building" },
    { id: "admin-bookings", label: "借用管理", hint: "全校借用记录", icon: "clipboard" },
  ],
};

const PAGE_META = {
  overview: { title: "教室总览", desc: "实时掌握每一间教室的使用状态", kicker: "REAL-TIME CAMPUS" },
  schedule: { title: "我的课表", desc: "按周查看课程与上课地点", kicker: "WEEKLY SCHEDULE" },
  borrow: { title: "借用教室", desc: "按日期与时段查找可用的空闲教室", kicker: "ROOM BOOKING" },
  bookings: { title: "我的借用", desc: "查看、管理已提交的教室借用", kicker: "MY BOOKINGS" },
  courses: { title: "课程管理", desc: "维护课程、教师、教室与上课时间", kicker: "COURSE ADMIN" },
  classrooms: { title: "教室管理", desc: "维护教室档案与开放状态", kicker: "CLASSROOM ADMIN" },
  "admin-bookings": { title: "借用管理", desc: "查看全校教师的教室借用记录", kicker: "BOOKING ADMIN" },
};

const RENDERERS = {
  overview: renderOverviewPage,
  schedule: renderSchedulePage,
  borrow: renderBorrowPage,
  bookings: renderMyBookingsPage,
  courses: renderCoursesPage,
  classrooms: renderClassroomsPage,
  "admin-bookings": renderAdminBookingsPage,
};

function allowedPages() {
  return (NAV[state.user?.role] || []).map((item) => item.id);
}

function pageFromHash() {
  const value = location.hash.replace(/^#\/?/, "").trim();
  return value || "overview";
}

function go(page, replace = false) {
  const target = `#/${page}`;
  if (replace) {
    history.replaceState(null, "", target);
    route();
  } else if (location.hash === target) {
    route();
  } else {
    location.hash = target;
  }
}

function startClock() {
  clearInterval(state.clockTimer);
  const tick = () => {
    const el = $("#live-clock");
    if (!el) return;
    const now = new Date();
    el.innerHTML = `
      <strong>${String(now.getHours()).padStart(2, "0")}:${String(now.getMinutes()).padStart(2, "0")}:${String(now.getSeconds()).padStart(2, "0")}</strong>
      <span>${now.getMonth() + 1}月${now.getDate()}日 · ${["周日", "周一", "周二", "周三", "周四", "周五", "周六"][now.getDay()]}</span>`;
  };
  tick();
  state.clockTimer = setInterval(tick, 1000);
}

function renderShell(page) {
  clearInterval(state.pollTimer);
  state.pollTimer = null;
  const user = state.user;
  const meta = state.meta;
  const nav = NAV[user.role] || [];
  const pageMeta = PAGE_META[page] || PAGE_META.overview;
  const semester = meta?.semester || {};
  const weekText = semester.week > 0 && semester.week <= 30 ? `第 ${semester.week} 教学周` : "假期";
  const navHtml = nav
    .map(
      (item) => `
      <a class="nav-item ${item.id === page ? "is-active" : ""}" href="#/${item.id}" data-nav>
        <span class="nav-icon">${icon(item.icon)}</span>
        <span class="nav-copy"><strong>${item.label}</strong><small>${item.hint}</small></span>
        <span class="nav-arrow">${icon("arrow")}</span>
      </a>`
    )
    .join("");

  $("#app").innerHTML = `
    <div class="app-shell">
      <div class="sidebar-scrim" data-action="close-sidebar"></div>
      <aside class="sidebar" id="sidebar">
        <div class="brand">
          <div class="brand-mark">知</div>
          <div class="brand-copy">
            <strong>知行教室</strong>
            <span>教室课程管理系统</span>
          </div>
        </div>

        <div class="sidebar-term">
          <div class="term-top"><span>当前学期</span><em>${semester.week > 0 && semester.week <= 30 ? `第 ${semester.week} 周` : "假期"}</em></div>
          <strong>${esc(semester.name || "—")}</strong>
          <div class="term-bar"><span style="width:${Math.min(100, Math.max(4, ((semester.week || 0) / 18) * 100))}%"></span></div>
          <small>${weekText} · 共 18 周教学安排</small>
        </div>

        <div class="nav-label">工作台</div>
        <nav class="sidebar-nav">${navHtml}</nav>

        <div class="sidebar-foot">
          <div class="sidebar-tip">
            <span class="tip-icon">${icon("spark")}</span>
            <div><strong>数据实时同步</strong><small>教室状态每 10 秒自动刷新</small></div>
          </div>
          <div class="sidebar-user">
            <div class="avatar">${initial(user.name)}</div>
            <div class="user-copy">
              <strong>${esc(user.name)}</strong>
              <span>${ROLE_LABELS[user.role]} · ${esc(user.identifier || "—")}</span>
            </div>
            <button type="button" class="icon-btn logout-btn" data-action="logout" title="退出登录" aria-label="退出登录">${icon("logout")}</button>
          </div>
        </div>
      </aside>

      <div class="main-shell">
        <header class="topbar">
          <div class="topbar-left">
            <button type="button" class="icon-btn mobile-menu" data-action="toggle-sidebar" aria-label="打开菜单">${icon("grid")}</button>
            <div class="topbar-heading">
              <span>${pageMeta.kicker}</span>
              <h1>${pageMeta.title}</h1>
            </div>
          </div>
          <div class="topbar-right">
            <div class="live-clock" id="live-clock"></div>
            <div class="topbar-role">${ROLE_LABELS[user.role]}端</div>
          </div>
        </header>
        <main class="page-content" id="page-content">
          <div class="page-loading-inline"><span class="spinner"></span>正在载入页面...</div>
        </main>
      </div>
    </div>`;

  $("[data-action='logout']")?.addEventListener("click", logout);
  $("[data-action='toggle-sidebar']")?.addEventListener("click", () => {
    $(".app-shell")?.classList.add("sidebar-open");
  });
  $("[data-action='close-sidebar']")?.addEventListener("click", () => {
    $(".app-shell")?.classList.remove("sidebar-open");
  });
  $$("[data-nav]").forEach((link) => {
    link.addEventListener("click", () => $(".app-shell")?.classList.remove("sidebar-open"));
  });

  document.title = `${pageMeta.title} · 知行教室`;
  startClock();
}

async function route() {
  if (!state.user) {
    renderLoginPage();
    return;
  }
  const pages = allowedPages();
  let page = pageFromHash();
  if (!pages.includes(page)) {
    page = "overview";
    history.replaceState(null, "", "#/overview");
  }
  state.page = page;
  const meta = PAGE_META[page];
  document.title = `${meta.title} · 知行教室`;
  renderShell(page);
  window.scrollTo({ top: 0, behavior: "auto" });

  const content = $("#page-content");
  try {
    await RENDERERS[page](content);
  } catch (error) {
    console.error(error);
    content.innerHTML = `
      <div class="error-state">
        <div class="error-icon">${icon("alert")}</div>
        <h2>页面加载失败</h2>
        <p>${esc(error.message || "请稍后重试")}</p>
        <button type="button" class="btn btn-primary" data-retry>${icon("refresh")}重新加载</button>
      </div>`;
    content.querySelector("[data-retry]")?.addEventListener("click", route);
  }
}

async function logout() {
  const confirmed = await confirmDialog({
    title: "退出登录",
    message: "确定要退出当前账号吗？",
    confirmText: "退出登录",
  });
  if (!confirmed) return;
  try {
    await api("/logout", { method: "POST" });
  } catch {
    /* 即使请求失败也回到登录页。 */
  }
  clearInterval(state.pollTimer);
  clearInterval(state.clockTimer);
  state.user = null;
  state.csrf = null;
  state.meta = null;
  state.overview = null;
  state.filters = null;
  state.borrowFilters = null;
  history.replaceState(null, "", location.pathname);
  renderLoginPage();
}

function renderLoginPage(message = "") {
  clearInterval(state.pollTimer);
  clearInterval(state.clockTimer);
  closeActiveModal();
  $("#modal-root").innerHTML = "";
  document.body.classList.remove("modal-open");
  document.title = "登录 · 知行教室";
  $("#app").innerHTML = `
    <div class="login-page">
      <section class="login-visual">
        <div class="login-visual-bg"></div>
        <div class="login-brand">
          <div class="brand-mark brand-mark-light">知</div>
          <div class="brand-copy">
            <strong>知行教室</strong>
            <span>ZHIXING CLASSROOM</span>
          </div>
        </div>

        <div class="login-copy">
          <span class="eyebrow eyebrow-light">SMART CAMPUS · 2026</span>
          <h1>让每一间教室，<br><em>都被看见。</em></h1>
          <p>面向学生、教师与教务管理者的教室课程协同平台。实时掌握空闲状态，让教学空间调度更简单。</p>
        </div>

        <div class="login-features">
          <div class="login-feature">
            <span>${icon("grid")}</span>
            <div><strong>实时空闲</strong><small>教室状态一目了然</small></div>
          </div>
          <div class="login-feature">
            <span>${icon("door")}</span>
            <div><strong>一键借用</strong><small>教师快速预约空间</small></div>
          </div>
          <div class="login-feature">
            <span>${icon("calendar")}</span>
            <div><strong>课程协同</strong><small>课表与排课统一管理</small></div>
          </div>
        </div>

        <div class="login-showcase" aria-hidden="true">
          <div class="showcase-top">
            <span class="live-dot"></span>
            <span>实时教室状态</span>
            <em>刚刚更新</em>
          </div>
          <div class="showcase-room">
            <div>
              <strong>综合楼 A101</strong>
              <small>多媒体教室 · 120 人</small>
            </div>
            <span class="showcase-status">空闲中</span>
          </div>
          <div class="showcase-timeline">
            <span class="showcase-seg seg-course" style="left:14%;width:12%"></span>
            <span class="showcase-seg seg-free" style="left:26%;width:34%"></span>
            <span class="showcase-seg seg-book" style="left:60%;width:9%"></span>
            <span class="showcase-seg seg-course" style="left:72%;width:16%"></span>
            <i class="showcase-now" style="left:43%"></i>
          </div>
          <div class="showcase-scale"><span>08:00</span><span>15:00</span><span>22:00</span></div>
        </div>

        <div class="login-visual-foot">知行教室 · 教学空间数字化平台</div>
      </section>

      <section class="login-panel">
        <div class="login-card">
          <div class="login-card-head">
            <span class="eyebrow">WELCOME BACK</span>
            <h2>登录你的工作台</h2>
            <p>学生、教师与管理员使用统一入口</p>
          </div>

          ${message ? `<div class="login-alert">${icon("info")}<span>${esc(message)}</span></div>` : ""}
          <div class="login-alert login-alert-error" id="login-error" hidden></div>

          <form id="login-form" class="login-form" novalidate>
            <label class="field">
              <span>账号</span>
              <div class="input-with-icon">
                ${icon("user")}
                <input type="text" name="username" id="login-username" autocomplete="username" placeholder="请输入账号" autofocus>
              </div>
            </label>
            <label class="field">
              <span>密码</span>
              <div class="input-with-icon">
                ${icon("lock")}
                <input type="password" name="password" id="login-password" autocomplete="current-password" placeholder="请输入密码">
                <button type="button" class="password-toggle" id="password-toggle">显示</button>
              </div>
            </label>
            <button type="submit" class="btn btn-primary btn-login" id="login-submit">
              登录系统
              ${icon("arrow")}
            </button>
          </form>

          <div class="login-divider"><span>演示账号 · 一键进入</span></div>
          <div class="demo-grid">
            <button type="button" class="demo-account" data-demo="student">
              <span class="demo-role">学生端</span>
              <strong>student</strong>
              <small>陈晨 · 计算机2301</small>
            </button>
            <button type="button" class="demo-account" data-demo="teacher">
              <span class="demo-role">教师端</span>
              <strong>teacher</strong>
              <small>张伟 · 计算机学院</small>
            </button>
            <button type="button" class="demo-account" data-demo="admin">
              <span class="demo-role">管理员</span>
              <strong>admin</strong>
              <small>周明 · 教务管理</small>
            </button>
          </div>
          <p class="demo-hint">演示账号密码均为 <code>123456</code>，点击卡片可直接登录</p>
        </div>
        <footer class="login-foot">© 2026 知行教室 · 让教学空间更有秩序</footer>
      </section>
    </div>`;

  $("#password-toggle")?.addEventListener("click", () => {
    const input = $("#login-password");
    const button = $("#password-toggle");
    const show = input.type === "password";
    input.type = show ? "text" : "password";
    button.textContent = show ? "隐藏" : "显示";
  });

  $("#login-form")?.addEventListener("submit", (event) => {
    event.preventDefault();
    performLogin($("#login-username").value.trim(), $("#login-password").value, $("#login-submit"));
  });

  $$("[data-demo]").forEach((button) => {
    button.addEventListener("click", () => {
      $("#login-username").value = button.dataset.demo;
      $("#login-password").value = "123456";
      performLogin(button.dataset.demo, "123456", button);
    });
  });
}

async function performLogin(username, password, button) {
  const errorBox = $("#login-error");
  if (errorBox) {
    errorBox.hidden = true;
    errorBox.textContent = "";
  }
  if (!username || !password) {
    if (errorBox) {
      errorBox.hidden = false;
      errorBox.innerHTML = `${icon("alert")}<span>请输入账号和密码</span>`;
    }
    return;
  }
  setButtonLoading(button, true, "登录中...");
  try {
    const response = await fetch("/api/login", {
      method: "POST",
      credentials: "same-origin",
      headers: { "Content-Type": "application/json", Accept: "application/json" },
      body: JSON.stringify({ username, password }),
    });
    const data = await response.json().catch(() => ({}));
    if (!response.ok) {
      throw new Error(data.error || "登录失败，请检查账号密码");
    }
    state.user = data.user;
    state.csrf = data.csrf;
    state.meta = data.meta;
    state.filters = null;
    state.borrowFilters = null;
    state.overview = null;
    state.followNow = false;
    toast(`欢迎回来，${data.user.name}`);
    if (location.hash && allowedPages().includes(pageFromHash())) {
      route();
    } else {
      go("overview", true);
    }
  } catch (error) {
    if (errorBox) {
      errorBox.hidden = false;
      errorBox.innerHTML = `${icon("alert")}<span>${esc(error.message)}</span>`;
    }
  } finally {
    setButtonLoading(button, false);
  }
}

async function boot() {
  try {
    const response = await fetch("/api/me", { credentials: "same-origin", headers: { Accept: "application/json" } });
    if (response.ok) {
      const data = await response.json();
      state.user = data.user;
      state.csrf = data.csrf;
      await loadMeta();
      if (!location.hash || !allowedPages().includes(pageFromHash())) {
        history.replaceState(null, "", "#/overview");
      }
      route();
      return;
    }
    renderLoginPage();
  } catch (error) {
    console.error(error);
    renderLoginPage("无法连接服务器，请确认服务已启动");
  }
}

window.addEventListener("hashchange", () => {
  if (state.user) route();
});
window.addEventListener("auth-expired", () => {
  renderLoginPage("登录状态已失效，请重新登录");
});
window.addEventListener("keydown", (event) => {
  if (event.key === "Escape") {
    $(".app-shell")?.classList.remove("sidebar-open");
  }
});

boot();
`````


## `static/js/pages/overview.js`

`````javascript
/* 教室实时总览。 */

import {
  $,
  $$,
  state,
  icon,
  esc,
  api,
  openModal,
  roomStateMeta,
  formatDateCN,
  formatShortDate,
  weekdayLabel,
  minutesToTime,
  timestampLabel,
  emptyState,
  pageSkeleton,
  suggestedBorrowRange,
  timeToMinutes as toMin,
} from "../core.js";
import { openBorrowModal } from "./booking.js";

const WINDOW_START = 480;
const WINDOW_END = 1320;
const WINDOW_SPAN = WINDOW_END - WINDOW_START;

function initFilters() {
  if (state.filters) return;
  const now = new Date();
  const current = now.getHours() * 60 + now.getMinutes();
  let at = "10:10";
  let followNow = false;
  if (current >= WINDOW_START && current < WINDOW_END) {
    at = minutesToTime(current);
    followNow = true;
  } else if (current >= WINDOW_END) {
    at = "19:00";
  }
  state.filters = {
    date: `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, "0")}-${String(now.getDate()).padStart(2, "0")}`,
    at,
    building: "",
    capacity: "",
    status: "all",
    q: "",
  };
  state.followNow = followNow;
}

export async function renderOverviewPage(content) {
  initFilters();
  const filters = state.filters;

  content.innerHTML = `
    <div class="page-heading">
      <div>
        <div class="eyebrow">REAL-TIME CAMPUS</div>
        <h1>教室实时总览</h1>
        <p>查看各教学楼教室的占用情况与今日安排，数据每 10 秒自动刷新。</p>
      </div>
      <div class="heading-actions">
        <span class="live-pill"><span class="live-dot"></span>实时同步 <b id="last-update">--:--:--</b></span>
      </div>
    </div>

    <section class="filter-card">
      <div class="filter-grid">
        <label class="compact-field">
          <span>查看日期</span>
          <input type="date" id="filter-date" value="${filters.date}">
        </label>
        <label class="compact-field">
          <span>查询时间</span>
          <input type="time" id="filter-at" step="300" value="${filters.at}">
        </label>
        <label class="compact-field">
          <span>教学楼</span>
          <select id="filter-building">
            <option value="">全部楼栋</option>
            ${(state.meta.buildings || []).map((building) => `<option value="${esc(building)}" ${filters.building === building ? "selected" : ""}>${esc(building)}</option>`).join("")}
          </select>
        </label>
        <label class="compact-field">
          <span>容量</span>
          <select id="filter-capacity">
            <option value="">不限容量</option>
            <option value="40" ${filters.capacity === "40" ? "selected" : ""}>40 人以上</option>
            <option value="60" ${filters.capacity === "60" ? "selected" : ""}>60 人以上</option>
            <option value="100" ${filters.capacity === "100" ? "selected" : ""}>100 人以上</option>
          </select>
        </label>
        <label class="compact-field search-field">
          <span>搜索教室</span>
          <div class="search-input">${icon("search")}<input type="search" id="filter-q" placeholder="教室名称 / 楼栋 / 类型" value="${esc(filters.q)}"></div>
        </label>
        <button type="button" class="btn btn-ghost btn-now" id="btn-now">${icon("clock")}回到现在</button>
      </div>
      <div class="filter-bottom">
        <div class="segmented" id="status-segments" role="group" aria-label="教室状态筛选">
          <button type="button" class="segment ${filters.status === "all" ? "is-active" : ""}" data-status="all">全部</button>
          <button type="button" class="segment ${filters.status === "free" ? "is-active" : ""}" data-status="free">空闲中</button>
          <button type="button" class="segment ${filters.status === "busy" ? "is-active" : ""}" data-status="busy">使用中</button>
          <button type="button" class="segment ${filters.status === "maintenance" ? "is-active" : ""}" data-status="maintenance">维护中</button>
        </div>
        <div class="filter-count" id="result-count">正在读取教室数据…</div>
      </div>
    </section>

    <div id="overview-notice"></div>
    <section id="stats-area"></section>

    <div class="section-head">
      <div>
        <h2>教室状态 <span class="section-badge" id="week-badge"></span></h2>
        <p id="grid-subtitle">点击卡片查看当日完整时间轴</p>
      </div>
      <div class="legend">
        <span><i class="legend-dot course"></i>课程</span>
        <span><i class="legend-dot booking"></i>借用</span>
        <span><i class="legend-dot now"></i>查询时刻</span>
      </div>
    </div>
    <section class="room-grid" id="room-grid">${roomSkeleton(4)}</section>`;

  bindControls(content);
  await loadOverview(true);
  startPolling();
}

function bindControls(content) {
  const dateInput = $("#filter-date", content);
  const timeInput = $("#filter-at", content);
  const building = $("#filter-building", content);
  const capacity = $("#filter-capacity", content);
  const q = $("#filter-q", content);

  dateInput.addEventListener("change", () => {
    state.filters.date = dateInput.value || state.filters.date;
    state.followNow = false;
    loadOverview(true);
  });
  timeInput.addEventListener("change", () => {
    state.filters.at = timeInput.value || state.filters.at;
    state.followNow = false;
    loadOverview(false);
  });
  building.addEventListener("change", () => {
    state.filters.building = building.value;
    renderRooms();
  });
  capacity.addEventListener("change", () => {
    state.filters.capacity = capacity.value;
    renderRooms();
  });
  q.addEventListener("input", () => {
    state.filters.q = q.value.trim();
    renderRooms();
  });
  $("#btn-now", content).addEventListener("click", () => {
    const now = new Date();
    const current = now.getHours() * 60 + now.getMinutes();
    state.filters.date = localDate(now);
    state.filters.at = minutesToTime(current);
    state.followNow = true;
    dateInput.value = state.filters.date;
    timeInput.value = state.filters.at;
    loadOverview(false);
  });
  $$("#status-segments .segment", content).forEach((button) => {
    button.addEventListener("click", () => {
      state.filters.status = button.dataset.status;
      $$("#status-segments .segment", content).forEach((item) => item.classList.toggle("is-active", item === button));
      renderRooms();
    });
  });
}

function localDate(date) {
  return `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, "0")}-${String(date.getDate()).padStart(2, "0")}`;
}

function roomSkeleton(count = 4) {
  return `<div class="skeleton-grid">${Array.from({ length: count }, () => '<div class="skeleton sk-room"></div>').join("")}</div>`;
}

async function loadOverview(showSkeleton) {
  try {
    if (showSkeleton) {
      const statsArea = $("#stats-area");
      if (statsArea) {
        statsArea.innerHTML = `<div class="skeleton-stats">${Array.from({ length: 4 }, () => '<div class="skeleton sk-card"></div>').join("")}</div>`;
      }
      const grid = $("#room-grid");
      if (grid) grid.innerHTML = roomSkeleton(4);
    }
    const data = await api(`/overview?date=${encodeURIComponent(state.filters.date)}&at=${encodeURIComponent(state.filters.at)}`);
    if (state.page !== "overview") return;
    state.overview = data;
    state.filters.at = data.at;
    state.filters.date = data.date;
    $("#filter-at") && ($("#filter-at").value = data.at);
    $("#filter-date") && ($("#filter-date").value = data.date);
    renderStats();
    renderRooms();
    renderNotice();
    const last = $("#last-update");
    if (last) last.textContent = timestampLabel(data.server_time);
    const badge = $("#week-badge");
    if (badge) badge.textContent = data.week > 0 && data.week <= 30 ? `第 ${data.week} 教学周` : "非教学周";
  } catch (error) {
    const grid = $("#room-grid");
    if (grid) {
      grid.innerHTML = emptyState("alert", "读取失败", error.message, `<button class="btn btn-ghost" data-action="retry">${icon("refresh")}重新加载</button>`);
      grid.querySelector("[data-action='retry']")?.addEventListener("click", () => loadOverview(true));
    }
  }
}

function startPolling() {
  clearInterval(state.pollTimer);
  state.pollTimer = setInterval(async () => {
    if (state.page !== "overview" || document.hidden) return;
    if (state.followNow) {
      const now = new Date();
      const current = now.getHours() * 60 + now.getMinutes();
      if (current >= WINDOW_START && current < WINDOW_END) {
        state.filters.at = minutesToTime(current);
        const input = $("#filter-at");
        if (input && document.activeElement !== input) input.value = state.filters.at;
      }
    }
    await loadOverview(false);
  }, 10000);
}

function renderNotice() {
  const data = state.overview;
  const holder = $("#overview-notice");
  if (!data || !holder) return;
  const notices = [];
  const today = localDate(new Date());
  const current = new Date().getHours() * 60 + new Date().getMinutes();
  if (data.date !== today) {
    notices.push({
      tone: "info",
      icon: "calendar",
      text: `正在查看 ${formatDateCN(data.date)} 的教室安排，状态按所选时间 ${data.at} 计算。`,
    });
  } else if (data.outside_window) {
    if (state.followNow) {
      notices.push({
        tone: "warning",
        icon: "clock",
        text: `当前时间 ${data.at} 不在教学开放时段（08:00-22:00），教室状态按此刻计算。`,
      });
    } else {
      notices.push({
        tone: "info",
        icon: "clock",
        text: `所选时间 ${data.at} 不在教学开放时段（08:00-22:00），教室状态按该时刻计算。`,
      });
    }
  } else if (current < WINDOW_START || current >= WINDOW_END) {
    notices.push({
      tone: "info",
      icon: "clock",
      text: `当前不在教学开放时段，已为你展示最近教学时段 ${data.at} 的教室安排。点击「回到现在」可查看此刻状态。`,
    });
  }
  if (!data.semester.has_courses) {
    notices.push({
      tone: "warning",
      icon: "info",
      text: "当前日期不在学期教学周范围内，页面仅展示教师借用安排。",
    });
  }
  holder.innerHTML = notices
    .map(
      (notice) => `
      <div class="notice notice-${notice.tone}">
        <span class="notice-icon">${icon(notice.icon)}</span>
        <span>${esc(notice.text)}</span>
      </div>`
    )
    .join("");
}

function renderStats() {
  const data = state.overview;
  if (!data) return;
  const stats = data.stats;
  const freeRatio = stats.total ? Math.round((stats.free / stats.total) * 100) : 0;
  const isLiveNow = data.date === localDate(new Date()) && state.followNow && !data.outside_window;
  $("#stats-area").innerHTML = `
    <div class="stats-grid">
      <article class="stat-card stat-free">
        <div class="stat-icon">${icon("door")}</div>
        <div class="stat-body">
          <span class="stat-label">${isLiveNow ? "当前空闲" : "查询时段空闲"}</span>
          <div class="stat-value">${stats.free}<small>/ ${stats.total} 间</small></div>
          <div class="stat-meter"><span style="width:${freeRatio}%"></span></div>
          <p>可用教室占比 ${freeRatio}%</p>
        </div>
      </article>
      <article class="stat-card stat-busy">
        <div class="stat-icon">${icon("users")}</div>
        <div class="stat-body">
          <span class="stat-label">使用中</span>
          <div class="stat-value">${stats.busy}<small>间</small></div>
          <p>课程与教师借用占用</p>
        </div>
      </article>
      <article class="stat-card stat-book">
        <div class="stat-icon">${icon("clipboard")}</div>
        <div class="stat-body">
          <span class="stat-label">今日借用</span>
          <div class="stat-value">${stats.bookings_today}<small>条</small></div>
          <p>教师提交的有效借用</p>
        </div>
      </article>
      <article class="stat-card stat-muted">
        <div class="stat-icon">${icon("settings")}</div>
        <div class="stat-body">
          <span class="stat-label">维护中</span>
          <div class="stat-value">${stats.maintenance}<small>间</small></div>
          <p>维护教室暂不开放借用</p>
        </div>
      </article>
    </div>`;
}

function filteredRooms() {
  const data = state.overview;
  if (!data) return [];
  const filters = state.filters;
  return data.classrooms.filter((room) => {
    if (filters.building && room.building !== filters.building) return false;
    if (filters.capacity && room.capacity < Number(filters.capacity)) return false;
    if (filters.status !== "all" && room.state !== filters.status) return false;
    if (filters.q) {
      const haystack = `${room.name} ${room.building} ${room.room_type} ${room.equipment.join(" ")}`.toLowerCase();
      if (!haystack.includes(filters.q.toLowerCase())) return false;
    }
    return true;
  });
}

function renderRooms() {
  const data = state.overview;
  const grid = $("#room-grid");
  if (!data || !grid) return;
  const rooms = filteredRooms();
  const count = $("#result-count");
  if (count) {
    count.textContent = `共 ${data.classrooms.length} 间教室 · 当前显示 ${rooms.length} 间`;
  }
  const subtitle = $("#grid-subtitle");
  if (subtitle) {
    subtitle.textContent = `${formatShortDate(data.date)} ${weekdayLabel(data.date)} · ${data.at} 时刻 · 点击卡片查看完整时间轴`;
  }
  if (!rooms.length) {
    grid.innerHTML = emptyState(
      "search",
      "没有匹配的教室",
      "试试调整楼栋、容量或状态筛选条件。",
      `<button type="button" class="btn btn-ghost" data-action="reset-filters">${icon("refresh")}清除筛选</button>`
    );
    grid.querySelector("[data-action='reset-filters']")?.addEventListener("click", resetFilters);
    return;
  }
  grid.innerHTML = rooms.map((room) => roomCard(room, data)).join("");
  grid.onclick = (event) => {
    const button = event.target.closest("[data-action]");
    if (!button) return;
    const roomId = Number(button.dataset.room);
    if (button.dataset.action === "detail" || button.dataset.action === "card") {
      openRoomDetail(roomId);
    }
    if (button.dataset.action === "borrow") {
      const room = data.classrooms.find((item) => item.id === roomId);
      if (room) openBorrowFromOverview(room);
    }
  };
}

function resetFilters() {
  state.filters.building = "";
  state.filters.capacity = "";
  state.filters.status = "all";
  state.filters.q = "";
  $("#filter-building").value = "";
  $("#filter-capacity").value = "";
  $("#filter-q").value = "";
  $$("#status-segments .segment").forEach((item) => item.classList.toggle("is-active", item.dataset.status === "all"));
  renderRooms();
}

function roomCard(room, data) {
  const meta = roomStateMeta(room.state);
  const atMin = toMin(data.at);
  const equipment = room.equipment.slice(0, 2);
  let occupancy = "";
  if (room.state === "maintenance") {
    occupancy = `<div class="occupancy-note is-maintenance"><strong>维护中，暂停开放</strong><span>请联系管理员确认恢复时间</span></div>`;
  } else if (room.state === "busy") {
    occupancy = `
      <div class="occupancy-note is-busy">
        <div class="occupancy-title"><span class="now-badge">使用中</span><strong>${esc(room.current.title)}</strong></div>
        <span>${room.current.start} - ${room.current.end} · ${esc(room.current.subtitle)}</span>
      </div>`;
  } else {
    const nextText = room.upcoming
      ? `<span>下一场 ${room.upcoming.start} · ${esc(room.upcoming.title)}</span>`
      : `<span>今日后续暂无安排</span>`;
    const freeTitle =
      data.outside_window
        ? "非开放时段"
        : data.date === localDate(new Date()) && state.followNow
          ? "现在可以使用"
          : "该时段可以使用";
    occupancy = `
      <div class="occupancy-note is-free">
        <div class="occupancy-title"><span class="now-badge">空闲</span><strong>${freeTitle}</strong></div>
        ${nextText}
      </div>`;
  }

  return `
    <article class="room-card room-${room.state}" data-room-card="${room.id}">
      <div class="room-card-head">
        <div class="room-title-wrap">
          <div class="room-icon">${icon("door")}</div>
          <div>
            <h3>${esc(room.name)}</h3>
            <p>${esc(room.building)} · ${esc(room.room_type)}</p>
          </div>
        </div>
        <span class="status-pill status-${meta.tone}"><i></i>${meta.label}</span>
      </div>
      <div class="room-facts">
        <span>${icon("users")} ${room.capacity} 人</span>
        ${equipment.map((item) => `<span>${icon("monitor")} ${esc(item)}</span>`).join("")}
      </div>
      ${occupancy}
      <div class="mini-timeline-wrap">
        ${timelineMarkup(room, atMin)}
        <div class="timeline-scale"><span>08:00</span><span>15:00</span><span>22:00</span></div>
      </div>
      <div class="room-card-foot">
        <button type="button" class="link-btn" data-action="detail" data-room="${room.id}">查看安排 ${icon("arrow")}</button>
        ${
          state.user.role === "teacher"
            ? `<button type="button" class="btn btn-primary btn-sm" data-action="borrow" data-room="${room.id}" ${room.state === "maintenance" ? "disabled" : ""}>${icon("door")}借用</button>`
            : `<span class="room-id">ROOM ${String(room.id).padStart(2, "0")}</span>`
        }
      </div>
    </article>`;
}

function timelineMarkup(room, atMin, full = false) {
  const segments = room.intervals
    .map((item) => {
      const left = ((Math.max(WINDOW_START, item.start_min) - WINDOW_START) / WINDOW_SPAN) * 100;
      const width = ((Math.min(WINDOW_END, item.end_min) - Math.max(WINDOW_START, item.start_min)) / WINDOW_SPAN) * 100;
      return `<span class="tl-seg tl-${item.kind}" style="left:${left}%;width:${Math.max(width, 0.7)}%" title="${esc(`${item.start}-${item.end} ${item.title}`)}"></span>`;
    })
    .join("");
  const markerLeft = Math.min(100, Math.max(0, ((atMin - WINDOW_START) / WINDOW_SPAN) * 100));
  const marker = atMin >= WINDOW_START && atMin <= WINDOW_END
    ? `<i class="tl-now" style="left:${markerLeft}%" title="查询时刻 ${minutesToTime(atMin)}"></i>`
    : "";
  return `<div class="timeline ${full ? "timeline-full" : ""}"><div class="tl-track">${segments}${marker}</div></div>`;
}

function openRoomDetail(roomId) {
  const data = state.overview;
  const room = data?.classrooms.find((item) => item.id === roomId);
  if (!room) return;
  const meta = roomStateMeta(room.state);
  const atMin = toMin(data.at);
  const intervals = room.intervals || [];
  const list = intervals.length
    ? intervals
        .map(
          (item) => `
          <div class="schedule-row">
            <div class="schedule-time"><strong>${item.start}</strong><span>${item.end}</span></div>
            <div class="schedule-rail"><i class="rail-${item.kind}"></i></div>
            <div class="schedule-info">
              <strong>${esc(item.title)}</strong>
              <span>${esc(item.subtitle)}</span>
            </div>
            <span class="tag tag-${item.kind}">${item.kind === "course" ? "课程" : "借用"}</span>
          </div>`
        )
        .join("")
    : `<div class="detail-empty">${icon("calendar")}当天暂无课程或借用安排</div>`;

  const footer = `
    <button type="button" class="btn btn-ghost" data-close>关闭</button>
    ${
      state.user.role === "teacher" && room.state !== "maintenance"
        ? `<button type="button" class="btn btn-primary" data-borrow>${icon("door")}借用此教室</button>`
        : ""
    }`;

  const modal = openModal({
    title: room.name,
    subtitle: `${formatDateCN(data.date)} · ${data.at} 时刻`,
    size: "modal-md",
    body: `
      <div class="detail-head">
        <div class="detail-badges">
          <span class="status-pill status-${meta.tone}"><i></i>${meta.label}</span>
          <span class="soft-badge">${esc(room.room_type)}</span>
          <span class="soft-badge">${room.capacity} 人</span>
        </div>
        <span class="detail-room-code">ROOM ${String(room.id).padStart(2, "0")}</span>
      </div>
      <div class="detail-timeline-block">
        ${timelineMarkup(room, atMin, true)}
        <div class="timeline-scale"><span>08:00</span><span>12:00</span><span>16:00</span><span>20:00</span><span>22:00</span></div>
      </div>
      <div class="detail-section-head"><h3>当日安排</h3><span>${intervals.length} 项</span></div>
      <div class="schedule-list">${list}</div>
      <div class="detail-equipment">
        <span>教室设备</span>
        <div>${room.equipment.map((item) => `<em>${esc(item)}</em>`).join("") || "<em>暂无设备信息</em>"}</div>
      </div>`,
    footer,
  });

  modal.element.querySelector("[data-close]")?.addEventListener("click", () => modal.close());
  modal.element.querySelector("[data-borrow]")?.addEventListener("click", () => {
    modal.close();
    openBorrowFromOverview(room);
  });
}

function openBorrowFromOverview(room) {
  const data = state.overview;
  const range = suggestedBorrowRange(room, data.at);
  openBorrowModal(room, {
    date: data.date,
    start: range.start,
    end: range.end,
    onSuccess: () => loadOverview(false),
  });
}
`````


## `static/js/pages/booking.js`

`````javascript
/* 教室借用：借用弹窗、查找空闲教室、我的借用与管理员借用记录。 */

import {
  $,
  $$,
  state,
  icon,
  esc,
  api,
  openModal,
  toast,
  confirmDialog,
  emptyState,
  formatDateCN,
  formatShortDate,
  relativeDate,
  weekdayLabel,
  timeToMinutes,
  minutesToTime,
  nowMinutes,
  dateString,
  defaultBorrowFilters,
  rangeOverlaps,
  suggestedBorrowRange,
  roomAvailabilityForRange,
  bookingStateMeta,
  setButtonLoading,
  formError,
  clearFormError,
} from "../core.js";

/* ------------------------------------------------------------------ */
/* 借用弹窗                                                            */
/* ------------------------------------------------------------------ */

export function openBorrowModal(room, options = {}) {
  const today = dateString();
  let date = options.date || today;
  let start = options.start;
  let end = options.end;
  if (date < today) {
    const fallback = defaultBorrowFilters();
    date = fallback.date;
    start = fallback.start;
    end = fallback.end;
  }
  if (!start || !end) {
    const range = suggestedBorrowRange(room, options.at || "10:00");
    start = range.start;
    end = range.end;
  }
  if (timeToMinutes(end) <= timeToMinutes(start)) {
    const range = suggestedBorrowRange(room, "10:00");
    start = range.start;
    end = range.end;
  }

  const body = `
    <form id="borrow-form" class="stack-form" novalidate>
      <div class="form-error" role="alert"></div>
      <div class="borrow-room-summary">
        <div class="room-icon">${icon("door")}</div>
        <div>
          <strong>${esc(room.name)}</strong>
          <span>${esc(room.building)} · ${esc(room.room_type)} · ${room.capacity} 人</span>
        </div>
        <span class="status-pill status-free"><i></i>可借用</span>
      </div>

      <div class="form-grid two">
        <label class="field">
          <span>借用日期</span>
          <input type="date" name="date" id="borrow-date" min="${today}" value="${date}" required>
        </label>
        <label class="field">
          <span>借用时段</span>
          <div class="time-range">
            <input type="time" name="start_time" id="borrow-start" step="300" value="${start}" required>
            <span>至</span>
            <input type="time" name="end_time" id="borrow-end" step="300" value="${end}" required>
          </div>
        </label>
      </div>

      <label class="field">
        <span>借用用途</span>
        <textarea name="purpose" id="borrow-purpose" rows="3" maxlength="100" placeholder="例如：课程组教研会议、毕业设计答疑、学科竞赛辅导" autofocus required></textarea>
        <small class="field-hint" id="duration-hint">借用时长 ${durationLabel(start, end)}</small>
      </label>

      <div class="form-note">${icon("info")}借用提交后立即生效。如与课程或其他借用冲突，系统将自动拦截。</div>
    </form>`;

  const modal = openModal({
    title: "借用教室",
    subtitle: `${room.name} · ${room.building}`,
    size: "modal-md",
    body,
    footer: `
      <button type="button" class="btn btn-ghost" data-cancel>取消</button>
      <button type="submit" class="btn btn-primary" form="borrow-form" data-submit>${icon("check")}确认借用</button>`,
  });

  const form = modal.element.querySelector("#borrow-form");
  const startInput = modal.element.querySelector("#borrow-start");
  const endInput = modal.element.querySelector("#borrow-end");
  const hint = modal.element.querySelector("#duration-hint");
  const submit = modal.element.querySelector("[data-submit]");

  const updateHint = () => {
    if (!hint) return;
    const value = durationLabel(startInput.value, endInput.value);
    hint.textContent = value === "—" ? "请选择有效的借用时段" : `借用时长 ${value}`;
  };
  startInput?.addEventListener("change", updateHint);
  endInput?.addEventListener("change", updateHint);
  modal.element.querySelector("[data-cancel]")?.addEventListener("click", () => modal.close());
  form?.addEventListener("input", () => clearFormError(form));

  form?.addEventListener("submit", async (event) => {
    event.preventDefault();
    clearFormError(form);
    const data = Object.fromEntries(new FormData(form).entries());
    const startMin = timeToMinutes(data.start_time);
    const endMin = timeToMinutes(data.end_time);
    if (!data.date || !data.start_time || !data.end_time) {
      formError(form, "请选择借用日期和时段");
      return;
    }
    if (data.date < today) {
      formError(form, "不能借用过去的日期");
      return;
    }
    if (endMin <= startMin) {
      formError(form, "结束时间必须晚于开始时间");
      return;
    }
    if (startMin < 480 || endMin > 1320) {
      formError(form, "借用时段需在 08:00-22:00 之间");
      return;
    }
    if (!data.purpose.trim()) {
      formError(form, "请填写借用用途");
      return;
    }

    setButtonLoading(submit, true, "提交中...");
    try {
      const result = await api("/bookings", {
        method: "POST",
        body: {
          classroom_id: room.id,
          date: data.date,
          start_time: data.start_time,
          end_time: data.end_time,
          purpose: data.purpose.trim(),
        },
      });
      toast(`已成功借用 ${result.booking.classroom_name}`);
      modal.close(true);
      if (options.onSuccess) options.onSuccess(result.booking);
    } catch (error) {
      formError(form, error.message);
      setButtonLoading(submit, false);
    }
  });
}

function durationLabel(start, end) {
  const minutes = timeToMinutes(end) - timeToMinutes(start);
  if (!Number.isFinite(minutes) || minutes <= 0) return "—";
  const hours = Math.floor(minutes / 60);
  const rest = minutes % 60;
  if (!hours) return `${rest} 分钟`;
  if (!rest) return `${hours} 小时`;
  return `${hours} 小时 ${rest} 分`;
}

/* ------------------------------------------------------------------ */
/* 教师：查找空闲教室                                                  */
/* ------------------------------------------------------------------ */

function initBorrowFilters() {
  const filters = state.borrowFilters || defaultBorrowFilters();
  const today = dateString();
  if (filters.date < today || (filters.date === today && timeToMinutes(filters.end) <= nowMinutes())) {
    Object.assign(filters, defaultBorrowFilters());
  }
  state.borrowFilters = filters;
}

export async function renderBorrowPage(content) {
  initBorrowFilters();
  const filters = state.borrowFilters;

  content.innerHTML = `
    <div class="page-heading">
      <div>
        <div class="eyebrow">ROOM BOOKING</div>
        <h1>借用空闲教室</h1>
        <p>选择日期与时段，系统会为你筛出整段时间都可用的教室。</p>
      </div>
      <div class="heading-actions">
        <button type="button" class="btn btn-ghost" data-action="my-bookings">${icon("clipboard")}我的借用</button>
      </div>
    </div>

    <div class="notice notice-info">
      <span class="notice-icon">${icon("info")}</span>
      <span>教师借用即时生效。提交前请确认时段与用途，管理员可在借用管理中查看记录。</span>
    </div>

    <section class="filter-card borrow-filter-card">
      <div class="borrow-filter-grid">
        <label class="compact-field">
          <span>借用日期</span>
          <input type="date" id="borrow-filter-date" min="${dateString()}" value="${filters.date}">
        </label>
        <label class="compact-field">
          <span>开始时间</span>
          <input type="time" id="borrow-filter-start" step="300" value="${filters.start}">
        </label>
        <label class="compact-field">
          <span>结束时间</span>
          <input type="time" id="borrow-filter-end" step="300" value="${filters.end}">
        </label>
        <label class="compact-field">
          <span>教学楼</span>
          <select id="borrow-filter-building">
            <option value="">全部楼栋</option>
            ${(state.meta.buildings || []).map((building) => `<option value="${esc(building)}" ${filters.building === building ? "selected" : ""}>${esc(building)}</option>`).join("")}
          </select>
        </label>
        <label class="compact-field">
          <span>容量</span>
          <select id="borrow-filter-capacity">
            <option value="">不限容量</option>
            <option value="40" ${filters.capacity === "40" ? "selected" : ""}>40 人以上</option>
            <option value="60" ${filters.capacity === "60" ? "selected" : ""}>60 人以上</option>
            <option value="100" ${filters.capacity === "100" ? "selected" : ""}>100 人以上</option>
          </select>
        </label>
        <button type="button" class="btn btn-primary btn-search-rooms" id="borrow-search">${icon("search")}查找教室</button>
      </div>
      <div class="filter-bottom">
        <div class="segmented" id="borrow-building-tabs">
          <button type="button" class="segment ${!filters.building ? "is-active" : ""}" data-borrow-building="">全部</button>
          ${(state.meta.buildings || []).map((building) => `<button type="button" class="segment ${filters.building === building ? "is-active" : ""}" data-borrow-building="${esc(building)}">${esc(building)}</button>`).join("")}
        </div>
        <div class="filter-count" id="borrow-count">正在查找可用教室…</div>
      </div>
    </section>

    <div id="borrow-notice"></div>
    <div class="section-head">
      <div>
        <h2>可用教室 <span class="section-badge" id="borrow-slot-badge"></span></h2>
        <p>绿色卡片表示所选时段内整段空闲</p>
      </div>
      <div class="legend">
        <span><i class="legend-dot free"></i>可借用</span>
        <span><i class="legend-dot busy"></i>时间冲突</span>
      </div>
    </div>
    <section class="borrow-grid" id="borrow-results"></section>`;

  bindBorrowControls(content);
  await loadBorrowRooms(true);
}

function bindBorrowControls(content) {
  const date = $("#borrow-filter-date", content);
  const start = $("#borrow-filter-start", content);
  const end = $("#borrow-filter-end", content);
  const building = $("#borrow-filter-building", content);
  const capacity = $("#borrow-filter-capacity", content);

  const syncAndLoad = () => {
    state.borrowFilters.date = date.value;
    state.borrowFilters.start = start.value;
    state.borrowFilters.end = end.value;
    loadBorrowRooms(false);
  };
  date.addEventListener("change", syncAndLoad);
  start.addEventListener("change", syncAndLoad);
  end.addEventListener("change", syncAndLoad);
  $("#borrow-search", content).addEventListener("click", syncAndLoad);
  building.addEventListener("change", () => {
    state.borrowFilters.building = building.value;
    $$("#borrow-building-tabs .segment", content).forEach((item) =>
      item.classList.toggle("is-active", item.dataset.borrowBuilding === building.value)
    );
    renderBorrowResults();
  });
  capacity.addEventListener("change", () => {
    state.borrowFilters.capacity = capacity.value;
    renderBorrowResults();
  });
  $$("#borrow-building-tabs .segment", content).forEach((button) => {
    button.addEventListener("click", () => {
      building.value = button.dataset.borrowBuilding;
      state.borrowFilters.building = building.value;
      $$("#borrow-building-tabs .segment", content).forEach((item) => item.classList.toggle("is-active", item === button));
      renderBorrowResults();
    });
  });
  $("[data-action='my-bookings']", content)?.addEventListener("click", () => {
    location.hash = "#/bookings";
  });
}

async function loadBorrowRooms(showLoading) {
  const filters = state.borrowFilters;
  const notice = $("#borrow-notice");
  const results = $("#borrow-results");
  if (!results) return;

  const startMin = timeToMinutes(filters.start);
  const endMin = timeToMinutes(filters.end);
  if (!filters.date || !filters.start || !filters.end) {
    showBorrowNotice("请选择完整的借用日期与时段。");
    return;
  }
  if (endMin <= startMin) {
    showBorrowNotice("结束时间必须晚于开始时间。");
    results.innerHTML = emptyState("clock", "时段不正确", "请调整开始时间和结束时间。");
    return;
  }
  if (startMin < 480 || endMin > 1320) {
    showBorrowNotice("借用时间需在 08:00-22:00 之间。");
    results.innerHTML = emptyState("clock", "不在开放时段", "教室开放时间为每天 08:00-22:00。");
    return;
  }
  if (filters.date < dateString()) {
    showBorrowNotice("不能借用过去的日期。");
    results.innerHTML = emptyState("calendar", "日期已过", "请选择今天或之后的日期。");
    return;
  }
  showBorrowNotice("");

  if (showLoading) {
    results.innerHTML = `<div class="skeleton-grid borrow-skeleton">${Array.from({ length: 6 }, () => '<div class="skeleton sk-borrow"></div>').join("")}</div>`;
  }

  try {
    const data = await api(`/overview?date=${encodeURIComponent(filters.date)}&at=${encodeURIComponent(filters.start)}`);
    if (state.page !== "borrow") return;
    state.borrowData = data;
    renderBorrowResults();
    const badge = $("#borrow-slot-badge");
    if (badge) {
      badge.textContent = `${formatShortDate(filters.date)} ${weekdayLabel(filters.date)} ${filters.start}-${filters.end}`;
    }
  } catch (error) {
    showBorrowNotice(error.message, "error");
    results.innerHTML = emptyState("alert", "查找失败", error.message);
  }
}

function showBorrowNotice(message, tone = "info", actionHtml = "") {
  const holder = $("#borrow-notice");
  if (!holder) return;
  const iconName = tone === "error" ? "alert" : tone === "warning" ? "clock" : "info";
  holder.innerHTML = message
    ? `<div class="notice notice-${tone}"><span class="notice-icon">${icon(iconName)}</span><span>${esc(message)}</span>${actionHtml}</div>`
    : "";
}

function nextTeacherFreeRange(intervals, selectedStart, duration = 90) {
  let start = Math.max(480, timeToMinutes(selectedStart));
  if (state.borrowFilters.date === dateString()) {
    start = Math.max(start, Math.ceil((nowMinutes() + 10) / 5) * 5);
  }
  const busy = [...intervals].sort((a, b) => a.start_min - b.start_min);
  for (let guard = 0; guard < 20; guard += 1) {
    const covering = busy.find((item) => start >= item.start_min && start < item.end_min);
    if (covering) {
      start = covering.end_min + 10;
      continue;
    }
    const upcoming = busy.find((item) => item.start_min > start);
    if (upcoming && upcoming.start_min - start < duration) {
      start = upcoming.end_min + 10;
      continue;
    }
    if (start + duration <= 1320) {
      return { start: minutesToTime(start), end: minutesToTime(start + duration) };
    }
    return null;
  }
  return null;
}

function renderBorrowResults() {
  const data = state.borrowData;
  const holder = $("#borrow-results");
  if (!data || !holder) return;
  const filters = state.borrowFilters;
  const slotPassed =
    filters.date < dateString() ||
    (filters.date === dateString() && timeToMinutes(filters.end) <= nowMinutes());
  const teacherConflicts = (data.teacher_intervals || []).filter((item) =>
    rangeOverlaps(filters.start, filters.end, item.start, item.end)
  );
  if (teacherConflicts.length) {
    const detail = teacherConflicts
      .map((item) => `${item.start}-${item.end} ${item.title}（${item.subtitle}）`)
      .slice(0, 2)
      .join("；");
    const suggestion = slotPassed ? null : nextTeacherFreeRange(data.teacher_intervals || [], filters.start);
    const action = suggestion
      ? `<button type="button" class="notice-btn" data-use-suggestion="${suggestion.start}-${suggestion.end}">采用建议 ${suggestion.start}-${suggestion.end}</button>`
      : "";
    showBorrowNotice(`你在所选时段已有安排：${detail}。请调整开始或结束时间。`, "warning", action);
    const suggestionButton = $("#borrow-notice [data-use-suggestion]");
    if (suggestionButton) {
      suggestionButton.addEventListener("click", () => {
        const [nextStart, nextEnd] = suggestionButton.dataset.useSuggestion.split("-");
        state.borrowFilters.start = nextStart;
        state.borrowFilters.end = nextEnd;
        const startInput = $("#borrow-filter-start");
        const endInput = $("#borrow-filter-end");
        if (startInput) startInput.value = nextStart;
        if (endInput) endInput.value = nextEnd;
        loadBorrowRooms(true);
      });
    }
  }

  let rooms = data.classrooms.filter((room) => {
    if (filters.building && room.building !== filters.building) return false;
    if (filters.capacity && room.capacity < Number(filters.capacity)) return false;
    return true;
  });

  rooms = rooms.map((room) => {
    const range = roomAvailabilityForRange(room, filters.start, filters.end);
    return { room, ...range, passed: slotPassed, teacherConflict: teacherConflicts.length > 0 };
  });
  rooms.sort((a, b) => {
    if (a.available !== b.available) return a.available ? -1 : 1;
    return a.room.name.localeCompare(b.room.name, "zh-CN");
  });

  const availableCount = rooms.filter(
    (item) => item.available && !item.passed && !item.teacherConflict
  ).length;
  const count = $("#borrow-count");
  if (count) count.textContent = `${availableCount} 间可借用 · 共 ${rooms.length} 间教室`;

  if (!rooms.length) {
    holder.innerHTML = emptyState("search", "没有符合条件的教室", "请调整楼栋或容量筛选条件。");
    return;
  }

  holder.innerHTML = rooms.map(({ room, available, conflicts, passed, teacherConflict }) => {
    const disabled = !available || passed || teacherConflict;
    let statusHtml = "";
    if (passed) {
      statusHtml = `<div class="borrow-state is-muted">${icon("clock")}<div><strong>时段已过</strong><span>请选择今天稍后的时间或未来日期</span></div></div>`;
    } else if (teacherConflict) {
      statusHtml = `<div class="borrow-state is-conflict">${icon("clock")}<div><strong>教师时间冲突</strong><span>你在该时段已有课程或借用</span></div></div>`;
    } else if (available) {
      statusHtml = `<div class="borrow-state is-free">${icon("check")}<div><strong>整段时段可借用</strong><span>${filters.start} - ${filters.end} · ${durationLabel(filters.start, filters.end)}</span></div></div>`;
    } else {
      const conflictText = conflicts
        .map((item) => `${item.start}-${item.end} ${item.title}`)
        .slice(0, 2)
        .join("；");
      statusHtml = `<div class="borrow-state is-conflict">${icon("alert")}<div><strong>时间冲突</strong><span>${esc(conflictText || "该时段已有安排")}</span></div></div>`;
    }

    return `
      <article class="borrow-card ${available && !passed && !teacherConflict ? "is-available" : "is-unavailable"}">
        <div class="borrow-card-head">
          <div class="room-title-wrap">
            <div class="room-icon">${icon("door")}</div>
            <div>
              <h3>${esc(room.name)}</h3>
              <p>${esc(room.building)} · ${esc(room.room_type)}</p>
            </div>
          </div>
          <span class="soft-badge">${room.capacity} 人</span>
        </div>
        <div class="borrow-facts">
          ${room.equipment.slice(0, 3).map((item) => `<span>${icon("monitor")} ${esc(item)}</span>`).join("") || "<span>暂无设备信息</span>"}
        </div>
        ${statusHtml}
        <div class="borrow-card-foot">
          <span class="borrow-time">${icon("clock")}${filters.start} - ${filters.end}</span>
          <button type="button" class="btn ${available && !passed && !teacherConflict ? "btn-primary" : "btn-ghost"} btn-sm" data-borrow-room="${room.id}" ${disabled ? "disabled" : ""}>
            ${passed ? "时段已过" : teacherConflict ? "时间冲突" : available ? `${icon("door")}立即借用` : "不可借用"}
          </button>
        </div>
      </article>`;
  }).join("");

  holder.onclick = (event) => {
    const button = event.target.closest("[data-borrow-room]");
    if (!button || button.disabled) return;
    const room = data.classrooms.find((item) => item.id === Number(button.dataset.borrowRoom));
    if (!room) return;
    openBorrowModal(room, {
      date: filters.date,
      start: filters.start,
      end: filters.end,
      onSuccess: () => loadBorrowRooms(false),
    });
  };
}

/* ------------------------------------------------------------------ */
/* 教师：我的借用                                                      */
/* ------------------------------------------------------------------ */

export async function renderMyBookingsPage(content) {
  content.innerHTML = `
    <div class="page-heading">
      <div>
        <div class="eyebrow">MY BOOKINGS</div>
        <h1>我的借用</h1>
        <p>查看已提交的教室借用，并在使用前取消不再需要的安排。</p>
      </div>
      <div class="heading-actions">
        <button type="button" class="btn btn-primary" data-action="go-borrow">${icon("plus")}借用教室</button>
      </div>
    </div>
    <div id="booking-stats" class="stats-grid stats-grid-mini"></div>
    <div id="booking-list" class="booking-list-wrap">
      <div class="page-loading-inline"><span class="spinner"></span>正在载入借用记录...</div>
    </div>`;

  $("[data-action='go-borrow']", content)?.addEventListener("click", () => {
    location.hash = "#/borrow";
  });
  await loadMyBookings();
}

async function loadMyBookings() {
  try {
    const data = await api("/bookings");
    if (state.page !== "bookings") return;
    const upcoming = data.bookings
      .filter((item) => item.state === "active")
      .sort((a, b) => `${a.date} ${a.start_time}`.localeCompare(`${b.date} ${b.start_time}`));
    const history = data.bookings
      .filter((item) => item.state !== "active")
      .sort((a, b) => `${b.date} ${b.start_time}`.localeCompare(`${a.date} ${a.start_time}`));
    renderBookingStats(upcoming, history);
    renderBookingList(upcoming, history);
  } catch (error) {
    $("#booking-list").innerHTML = emptyState("alert", "读取失败", error.message);
  }
}

function renderBookingStats(upcoming, history) {
  const holder = $("#booking-stats");
  if (!holder) return;
  const cancelled = history.filter((item) => item.state === "cancelled").length;
  holder.innerHTML = `
    <article class="stat-card stat-free stat-card-horizontal">
      <div class="stat-icon">${icon("calendar")}</div>
      <div class="stat-body"><span class="stat-label">即将使用</span><div class="stat-value">${upcoming.length}<small>条</small></div></div>
    </article>
    <article class="stat-card stat-muted stat-card-horizontal">
      <div class="stat-icon">${icon("check")}</div>
      <div class="stat-body"><span class="stat-label">历史记录</span><div class="stat-value">${history.length}<small>条</small></div></div>
    </article>
    <article class="stat-card stat-book stat-card-horizontal">
      <div class="stat-icon">${icon("x")}</div>
      <div class="stat-body"><span class="stat-label">已取消</span><div class="stat-value">${cancelled}<small>条</small></div></div>
    </article>`;
}

function renderBookingList(upcoming, history) {
  const holder = $("#booking-list");
  if (!holder) return;
  if (!upcoming.length && !history.length) {
    holder.innerHTML = emptyState(
      "door",
      "还没有借用记录",
      "去借用一间空闲教室，为课程讨论或教研活动预留空间。",
      `<button type="button" class="btn btn-primary" data-action="empty-borrow">${icon("plus")}借用教室</button>`
    );
    holder.querySelector("[data-action='empty-borrow']")?.addEventListener("click", () => {
      location.hash = "#/borrow";
    });
    return;
  }

  holder.innerHTML = `
    ${upcoming.length ? `
      <div class="section-head compact-head">
        <div><h2>即将使用 <span class="section-badge">${upcoming.length}</span></h2><p>借用已生效，可在开始前取消</p></div>
      </div>
      <div class="booking-list">${upcoming.map((item) => bookingItem(item, true)).join("")}</div>` : ""}
    ${history.length ? `
      <div class="section-head compact-head">
        <div><h2>历史记录 <span class="section-badge">${history.length}</span></h2><p>已结束或已取消的借用</p></div>
      </div>
      <div class="booking-list">${history.map((item) => bookingItem(item, false)).join("")}</div>` : ""}`;

  holder.onclick = (event) => {
    const button = event.target.closest("[data-cancel-booking]");
    if (!button) return;
    cancelBooking(Number(button.dataset.cancelBooking), loadMyBookings);
  };
}

function bookingItem(booking, active) {
  const date = new Date(`${booking.date}T00:00:00`);
  const meta = bookingStateMeta(booking.state);
  return `
    <article class="booking-item ${active ? "" : "is-history"}">
      <div class="booking-date">
        <strong>${String(date.getDate()).padStart(2, "0")}</strong>
        <span>${date.getMonth() + 1} 月</span>
        <em>${weekdayLabel(booking.date)}</em>
      </div>
      <div class="booking-main">
        <div class="booking-title">
          <h3>${esc(booking.classroom_name)}</h3>
          <span class="status-pill status-${meta.tone}"><i></i>${meta.label}</span>
          ${active ? `<span class="relative-chip">${relativeDate(booking.date)}</span>` : ""}
        </div>
        <div class="booking-meta">
          <span>${icon("calendar")}${formatDateCN(booking.date)}</span>
          <span>${icon("clock")}${booking.start_time} - ${booking.end_time}</span>
          <span>${icon("pin")}${esc(booking.building)}</span>
        </div>
        <p class="booking-purpose">${esc(booking.purpose)}</p>
      </div>
      <div class="booking-action">
        ${active ? `<button type="button" class="btn btn-ghost btn-sm btn-danger-text" data-cancel-booking="${booking.id}">取消借用</button>` : ""}
      </div>
    </article>`;
}

async function cancelBooking(id, onSuccess) {
  const confirmed = await confirmDialog({
    title: "取消借用",
    message: "取消后该时段将重新开放给其他教师借用，确定继续吗？",
    confirmText: "确认取消",
    danger: true,
  });
  if (!confirmed) return;
  try {
    await api(`/bookings/${id}`, { method: "DELETE" });
    toast("借用已取消");
    if (onSuccess) await onSuccess();
  } catch (error) {
    toast(error.message, "error");
  }
}

/* ------------------------------------------------------------------ */
/* 管理员：借用管理                                                    */
/* ------------------------------------------------------------------ */

export async function renderAdminBookingsPage(content) {
  content.innerHTML = `
    <div class="page-heading">
      <div>
        <div class="eyebrow">BOOKING ADMIN</div>
        <h1>借用管理</h1>
        <p>查看全校教师提交的教室借用，必要时可取消异常记录。</p>
      </div>
      <div class="heading-actions">
        <button type="button" class="btn btn-ghost" data-action="refresh-bookings">${icon("refresh")}刷新</button>
      </div>
    </div>
    <div id="admin-booking-stats" class="stats-grid stats-grid-mini"></div>
    <section class="filter-card">
      <div class="filter-grid admin-booking-filter">
        <label class="compact-field search-field">
          <span>搜索记录</span>
          <div class="search-input">${icon("search")}<input type="search" id="admin-booking-q" placeholder="教师 / 教室 / 用途"></div>
        </label>
        <label class="compact-field">
          <span>状态</span>
          <select id="admin-booking-status">
            <option value="all">全部状态</option>
            <option value="active">已生效</option>
            <option value="ended">已结束</option>
            <option value="cancelled">已取消</option>
          </select>
        </label>
      </div>
      <div class="filter-bottom">
        <div class="filter-count" id="admin-booking-count">正在载入…</div>
      </div>
    </section>
    <section class="table-card" id="admin-booking-table">
      <div class="page-loading-inline"><span class="spinner"></span>正在载入借用记录...</div>
    </section>`;

  $("[data-action='refresh-bookings']", content)?.addEventListener("click", () => loadAdminBookings(true));
  $("#admin-booking-q", content)?.addEventListener("input", renderAdminBookingTable);
  $("#admin-booking-status", content)?.addEventListener("change", renderAdminBookingTable);
  await loadAdminBookings(true);
}

async function loadAdminBookings(showLoading) {
  const table = $("#admin-booking-table");
  if (!table) return;
  if (showLoading) {
    table.innerHTML = `<div class="page-loading-inline"><span class="spinner"></span>正在载入借用记录...</div>`;
  }
  try {
    const data = await api("/bookings");
    if (state.page !== "admin-bookings") return;
    state.adminBookings = data.bookings;
    renderAdminBookingStats(data.bookings);
    renderAdminBookingTable();
  } catch (error) {
    table.innerHTML = emptyState("alert", "读取失败", error.message);
  }
}

function renderAdminBookingStats(bookings) {
  const holder = $("#admin-booking-stats");
  if (!holder) return;
  const active = bookings.filter((item) => item.state === "active").length;
  const ended = bookings.filter((item) => item.state === "ended").length;
  const cancelled = bookings.filter((item) => item.state === "cancelled").length;
  holder.innerHTML = `
    <article class="stat-card stat-free stat-card-horizontal"><div class="stat-icon">${icon("check")}</div><div class="stat-body"><span class="stat-label">已生效</span><div class="stat-value">${active}<small>条</small></div></div></article>
    <article class="stat-card stat-busy stat-card-horizontal"><div class="stat-icon">${icon("clock")}</div><div class="stat-body"><span class="stat-label">已结束</span><div class="stat-value">${ended}<small>条</small></div></div></article>
    <article class="stat-card stat-muted stat-card-horizontal"><div class="stat-icon">${icon("x")}</div><div class="stat-body"><span class="stat-label">已取消</span><div class="stat-value">${cancelled}<small>条</small></div></div></article>`;
}

function renderAdminBookingTable() {
  const holder = $("#admin-booking-table");
  if (!holder) return;
  const all = state.adminBookings || [];
  const q = ($("#admin-booking-q")?.value || "").trim().toLowerCase();
  const status = $("#admin-booking-status")?.value || "all";
  const rows = all.filter((item) => {
    if (status !== "all" && item.state !== status) return false;
    if (q) {
      const text = `${item.teacher_name} ${item.classroom_name} ${item.purpose} ${item.date}`.toLowerCase();
      if (!text.includes(q)) return false;
    }
    return true;
  });

  const count = $("#admin-booking-count");
  if (count) count.textContent = `共 ${all.length} 条记录 · 当前显示 ${rows.length} 条`;

  if (!rows.length) {
    holder.innerHTML = emptyState("clipboard", "暂无借用记录", "当前筛选条件下没有匹配的借用记录。");
    return;
  }

  holder.innerHTML = `
    <div class="table-scroll">
      <table class="data-table booking-admin-table">
        <thead>
          <tr><th>教师</th><th>教室</th><th>日期</th><th>时段</th><th>用途</th><th>状态</th><th></th></tr>
        </thead>
        <tbody>
          ${rows.map((item) => {
            const meta = bookingStateMeta(item.state);
            return `
              <tr>
                <td><div class="person-cell"><span class="avatar avatar-sm">${esc(item.teacher_name.slice(0, 1))}</span><strong>${esc(item.teacher_name)}</strong></div></td>
                <td><div class="room-cell"><strong>${esc(item.classroom_name)}</strong><span>${esc(item.building)}</span></div></td>
                <td><span class="nowrap">${formatDateCN(item.date)}</span></td>
                <td><span class="nowrap">${item.start_time} - ${item.end_time}</span></td>
                <td class="purpose-cell">${esc(item.purpose)}</td>
                <td><span class="status-pill status-${meta.tone}"><i></i>${meta.label}</span></td>
                <td class="actions-cell">${item.state === "active" ? `<button type="button" class="table-action danger" data-admin-cancel="${item.id}">取消</button>` : ""}</td>
              </tr>`;
          }).join("")}
        </tbody>
      </table>
    </div>`;

  holder.querySelectorAll("[data-admin-cancel]").forEach((button) => {
    button.addEventListener("click", () => cancelBooking(Number(button.dataset.adminCancel), () => loadAdminBookings(false)));
  });
}
`````


## `static/js/pages/courses.js`

`````javascript
/* 管理员：课程管理。 */

import {
  $,
  $$,
  state,
  icon,
  esc,
  api,
  openModal,
  toast,
  confirmDialog,
  emptyState,
  formatDateCN,
  timeToMinutes,
  setButtonLoading,
  formError,
  clearFormError,
} from "../core.js";

const WEEKDAYS = [
  { value: 1, label: "周一" },
  { value: 2, label: "周二" },
  { value: 3, label: "周三" },
  { value: 4, label: "周四" },
  { value: 5, label: "周五" },
  { value: 6, label: "周六" },
  { value: 7, label: "周日" },
];

export async function renderCoursesPage(content) {
  state.courseFilters = state.courseFilters || { q: "", weekday: "all" };
  const semester = state.meta.semester;
  const weekLabel = semester.week > 0 && semester.week <= 30 ? `第 ${semester.week} 教学周` : "非教学周";

  content.innerHTML = `
    <div class="page-heading">
      <div>
        <div class="eyebrow">COURSE ADMIN</div>
        <h1>课程管理</h1>
        <p>维护课程、授课教师、教室与上课时间，排课冲突由系统自动校验。</p>
      </div>
      <div class="heading-actions">
        <button type="button" class="btn btn-ghost" data-action="semester">${icon("settings")}学期设置</button>
        <button type="button" class="btn btn-primary" data-action="add-course">${icon("plus")}添加课程</button>
      </div>
    </div>

    <section class="semester-banner">
      <div class="semester-main">
        <span class="semester-kicker">当前学期</span>
        <h2>${esc(semester.name)}</h2>
        <p>${formatDateCN(semester.start)} 开学 · ${weekLabel}</p>
      </div>
      <div class="semester-week">
        <span>教学进度</span>
        <strong>${semester.week > 0 && semester.week <= 30 ? semester.week : 0}<small>/ 18 周</small></strong>
        <div class="term-bar"><span style="width:${Math.min(100, Math.max(2, ((semester.week || 0) / 18) * 100))}%"></span></div>
      </div>
      <div class="semester-facts">
        <div><strong id="course-total">—</strong><span>门课程</span></div>
        <div><strong id="course-classes">—</strong><span>个授课班级</span></div>
      </div>
    </section>

    <section class="filter-card">
      <div class="filter-grid course-filter-grid">
        <label class="compact-field search-field">
          <span>搜索课程</span>
          <div class="search-input">${icon("search")}<input type="search" id="course-q" placeholder="课程名称 / 代码 / 教师 / 教室" value="${esc(state.courseFilters.q)}"></div>
        </label>
        <label class="compact-field">
          <span>上课星期</span>
          <select id="course-weekday">
            <option value="all">全部星期</option>
            ${WEEKDAYS.map((item) => `<option value="${item.value}" ${String(state.courseFilters.weekday) === String(item.value) ? "selected" : ""}>${item.label}</option>`).join("")}
          </select>
        </label>
      </div>
      <div class="filter-bottom">
        <div class="filter-count" id="course-count">正在载入课程…</div>
      </div>
    </section>

    <section class="table-card" id="course-table">
      <div class="page-loading-inline"><span class="spinner"></span>正在载入课程...</div>
    </section>`;

  $("[data-action='add-course']", content)?.addEventListener("click", () => openCourseModal());
  $("[data-action='semester']", content)?.addEventListener("click", openSemesterModal);
  $("#course-q", content)?.addEventListener("input", () => {
    state.courseFilters.q = $("#course-q").value.trim();
    renderCourseTable();
  });
  $("#course-weekday", content)?.addEventListener("change", () => {
    state.courseFilters.weekday = $("#course-weekday").value;
    renderCourseTable();
  });

  await loadCourses(true);
}

async function loadCourses(showLoading) {
  const holder = $("#course-table");
  if (!holder) return;
  if (showLoading) {
    holder.innerHTML = `<div class="page-loading-inline"><span class="spinner"></span>正在载入课程...</div>`;
  }
  try {
    const data = await api("/courses");
    if (state.page !== "courses") return;
    state.courses = data.courses;
    const total = $("#course-total");
    const classes = $("#course-classes");
    if (total) total.textContent = data.courses.length;
    if (classes) classes.textContent = new Set(data.courses.map((item) => item.class_name)).size;
    renderCourseTable();
  } catch (error) {
    holder.innerHTML = emptyState("alert", "读取失败", error.message);
  }
}

function renderCourseTable() {
  const holder = $("#course-table");
  if (!holder) return;
  const all = state.courses || [];
  const q = state.courseFilters.q.toLowerCase();
  const weekday = state.courseFilters.weekday;
  const rows = all.filter((course) => {
    if (weekday !== "all" && String(course.weekday) !== String(weekday)) return false;
    if (q) {
      const haystack = `${course.name} ${course.code} ${course.teacher_name} ${course.classroom_name} ${course.class_name}`.toLowerCase();
      if (!haystack.includes(q)) return false;
    }
    return true;
  });
  const count = $("#course-count");
  if (count) count.textContent = `共 ${all.length} 门课程 · 当前显示 ${rows.length} 门`;

  if (!rows.length) {
    holder.innerHTML = emptyState(
      "book",
      all.length ? "没有匹配的课程" : "还没有课程",
      all.length ? "试试调整搜索关键词或星期筛选。" : "点击右上角「添加课程」开始排课。",
      all.length ? "" : `<button type="button" class="btn btn-primary" data-empty-add>${icon("plus")}添加课程</button>`
    );
    holder.querySelector("[data-empty-add]")?.addEventListener("click", () => openCourseModal());
    return;
  }

  holder.innerHTML = `
    <div class="table-scroll">
      <table class="data-table course-table">
        <thead>
          <tr><th>课程</th><th>授课教师</th><th>班级</th><th>教室</th><th>上课时间</th><th>周次</th><th>操作</th></tr>
        </thead>
        <tbody>
          ${rows.map((course) => `
            <tr>
              <td>
                <div class="course-cell">
                  <span class="course-code">${esc(course.code)}</span>
                  <strong>${esc(course.name)}</strong>
                </div>
              </td>
              <td><div class="person-cell"><span class="avatar avatar-sm">${esc(course.teacher_name.slice(0, 1))}</span>${esc(course.teacher_name)}</div></td>
              <td><span class="soft-badge">${esc(course.class_name)}</span></td>
              <td><div class="room-cell"><strong>${esc(course.classroom_name)}</strong></div></td>
              <td>
                <div class="time-cell"><strong>${course.weekday_label} ${course.start_time} - ${course.end_time}</strong></div>
              </td>
              <td><span class="nowrap">${course.weeks_label}</span></td>
              <td>
                <div class="row-actions">
                  <button type="button" class="table-action" data-course-edit="${course.id}">${icon("edit")}编辑</button>
                  <button type="button" class="table-action danger" data-course-delete="${course.id}">${icon("trash")}删除</button>
                </div>
              </td>
            </tr>`).join("")}
        </tbody>
      </table>
    </div>`;

  holder.querySelectorAll("[data-course-edit]").forEach((button) => {
    button.addEventListener("click", () => {
      const course = all.find((item) => item.id === Number(button.dataset.courseEdit));
      if (course) openCourseModal(course);
    });
  });
  holder.querySelectorAll("[data-course-delete]").forEach((button) => {
    button.addEventListener("click", () => {
      const course = all.find((item) => item.id === Number(button.dataset.courseDelete));
      if (course) deleteCourse(course);
    });
  });
}

function openCourseModal(course = null) {
  const editing = Boolean(course);
  const teachers = state.meta.teachers || [];
  const classrooms = state.meta.classrooms || [];
  const week = state.meta.semester.week || 1;
  const values = course || {
    code: "",
    name: "",
    teacher_id: teachers[0]?.id || "",
    classroom_id: classrooms[0]?.id || "",
    class_name: "",
    weekday: 1,
    start_time: "08:00",
    end_time: "09:35",
    weeks_start: 1,
    weeks_end: 18,
  };

  const body = `
    <form id="course-form" class="stack-form" novalidate>
      <div class="form-error" role="alert"></div>
      <div class="form-section-label">基本信息</div>
      <div class="form-grid two">
        <label class="field">
          <span>课程代码</span>
          <input type="text" name="code" maxlength="24" value="${esc(values.code)}" placeholder="例如 CS201" autofocus required>
        </label>
        <label class="field">
          <span>课程名称</span>
          <input type="text" name="name" maxlength="60" value="${esc(values.name)}" placeholder="例如 数据结构" required>
        </label>
        <label class="field">
          <span>授课教师</span>
          <select name="teacher_id" required>
            ${teachers.map((teacher) => `<option value="${teacher.id}" ${Number(values.teacher_id) === teacher.id ? "selected" : ""}>${esc(teacher.name)} · ${esc(teacher.identifier || "")}</option>`).join("")}
          </select>
        </label>
        <label class="field">
          <span>授课班级</span>
          <input type="text" name="class_name" maxlength="40" value="${esc(values.class_name)}" placeholder="例如 计算机2301" required>
        </label>
      </div>

      <div class="form-section-label">上课安排</div>
      <div class="form-grid two">
        <label class="field">
          <span>教室</span>
          <select name="classroom_id" required>
            ${classrooms.map((room) => `<option value="${room.id}" ${Number(values.classroom_id) === room.id ? "selected" : ""}>${esc(room.name)} · ${room.capacity}人${room.status === "maintenance" ? "（维护中）" : ""}</option>`).join("")}
          </select>
        </label>
        <label class="field">
          <span>上课星期</span>
          <select name="weekday" required>
            ${WEEKDAYS.map((day) => `<option value="${day.value}" ${Number(values.weekday) === day.value ? "selected" : ""}>${day.label}</option>`).join("")}
          </select>
        </label>
        <label class="field">
          <span>开始时间</span>
          <input type="time" name="start_time" step="300" value="${values.start_time}" required>
        </label>
        <label class="field">
          <span>结束时间</span>
          <input type="time" name="end_time" step="300" value="${values.end_time}" required>
        </label>
        <label class="field">
          <span>起始教学周</span>
          <input type="number" name="weeks_start" min="1" max="30" value="${values.weeks_start}" required>
        </label>
        <label class="field">
          <span>结束教学周</span>
          <input type="number" name="weeks_end" min="1" max="30" value="${values.weeks_end}" required>
        </label>
      </div>

      <div class="form-note">${icon("info")}系统会自动检查教室冲突、教师冲突、班级冲突，以及该时段已有的教师借用。当前演示数据为第 ${week} 教学周。</div>
    </form>`;

  const modal = openModal({
    title: editing ? "编辑课程" : "添加课程",
    subtitle: editing ? `${course.name} · ${course.code}` : "新建一门课程并安排固定上课时间",
    size: "modal-lg",
    body,
    footer: `
      <button type="button" class="btn btn-ghost" data-cancel>取消</button>
      <button type="submit" class="btn btn-primary" form="course-form" data-submit>${icon("check")}${editing ? "保存修改" : "创建课程"}</button>`,
  });

  const form = modal.element.querySelector("#course-form");
  const submit = modal.element.querySelector("[data-submit]");
  modal.element.querySelector("[data-cancel]")?.addEventListener("click", () => modal.close());
  form?.addEventListener("input", () => clearFormError(form));

  form?.addEventListener("submit", async (event) => {
    event.preventDefault();
    clearFormError(form);
    const formData = new FormData(form);
    const payload = Object.fromEntries(formData.entries());
    const startMin = timeToMinutes(payload.start_time);
    const endMin = timeToMinutes(payload.end_time);
    if (!payload.code.trim() || !payload.name.trim() || !payload.class_name.trim()) {
      formError(form, "请填写课程代码、课程名称和授课班级");
      return;
    }
    if (endMin <= startMin) {
      formError(form, "结束时间必须晚于开始时间");
      return;
    }
    if (startMin < 480 || endMin > 1320) {
      formError(form, "课程时间需在 08:00-22:00 之间");
      return;
    }
    if (Number(payload.weeks_end) < Number(payload.weeks_start)) {
      formError(form, "结束周不能早于起始周");
      return;
    }

    setButtonLoading(submit, true, "保存中...");
    try {
      const endpoint = editing ? `/courses/${course.id}` : "/courses";
      await api(endpoint, {
        method: editing ? "PUT" : "POST",
        body: {
          code: payload.code.trim(),
          name: payload.name.trim(),
          teacher_id: Number(payload.teacher_id),
          classroom_id: Number(payload.classroom_id),
          class_name: payload.class_name.trim(),
          weekday: Number(payload.weekday),
          start_time: payload.start_time,
          end_time: payload.end_time,
          weeks_start: Number(payload.weeks_start),
          weeks_end: Number(payload.weeks_end),
        },
      });
      modal.close(true);
      toast(editing ? "课程信息已更新" : "课程已创建");
      await loadCourses(false);
    } catch (error) {
      formError(form, error.message);
      setButtonLoading(submit, false);
    }
  });
}

async function deleteCourse(course) {
  const confirmed = await confirmDialog({
    title: "删除课程",
    message: `确定删除《${course.name}》（${course.code}）吗？删除后该时段将重新开放。`,
    confirmText: "删除课程",
    danger: true,
  });
  if (!confirmed) return;
  try {
    await api(`/courses/${course.id}`, { method: "DELETE" });
    toast("课程已删除");
    await loadCourses(false);
  } catch (error) {
    toast(error.message, "error");
  }
}

function openSemesterModal() {
  const semester = state.meta.semester;
  const body = `
    <form id="semester-form" class="stack-form" novalidate>
      <div class="form-error" role="alert"></div>
      <label class="field">
        <span>学期名称</span>
        <input type="text" name="name" maxlength="60" value="${esc(semester.name)}" autofocus required>
      </label>
      <label class="field">
        <span>学期开始日期（所在周的周一）</span>
        <input type="date" name="start" value="${semester.start}" required>
        <small class="field-hint">教学周将根据该日期重新计算，已有课程按周次继续生效。</small>
      </label>
      <div class="form-note">${icon("alert")}调整学期日期会改变当前教学周，请确认后再保存。</div>
    </form>`;

  const modal = openModal({
    title: "学期设置",
    subtitle: "设置当前学期名称与教学周起点",
    size: "modal-md",
    body,
    footer: `
      <button type="button" class="btn btn-ghost" data-cancel>取消</button>
      <button type="submit" class="btn btn-primary" form="semester-form" data-submit>${icon("check")}保存设置</button>`,
  });

  const form = modal.element.querySelector("#semester-form");
  const submit = modal.element.querySelector("[data-submit]");
  modal.element.querySelector("[data-cancel]")?.addEventListener("click", () => modal.close());
  form?.addEventListener("submit", async (event) => {
    event.preventDefault();
    clearFormError(form);
    const data = Object.fromEntries(new FormData(form).entries());
    if (!data.name.trim() || !data.start) {
      formError(form, "请填写学期名称和开始日期");
      return;
    }
    setButtonLoading(submit, true, "保存中...");
    try {
      const result = await api("/settings", {
        method: "PUT",
        body: { name: data.name.trim(), start: data.start },
      });
      state.meta.semester = result.semester;
      toast("学期设置已更新");
      modal.close(true);
      setTimeout(() => location.reload(), 350);
    } catch (error) {
      formError(form, error.message);
      setButtonLoading(submit, false);
    }
  });
}
`````


## `static/js/pages/classrooms.js`

`````javascript
/* 管理员：教室档案管理。 */

import {
  $,
  state,
  icon,
  esc,
  api,
  loadMeta,
  openModal,
  toast,
  emptyState,
  setButtonLoading,
  formError,
  clearFormError,
} from "../core.js";

export async function renderClassroomsPage(content) {
  state.classroomFilters = state.classroomFilters || { q: "", building: "", status: "all" };
  const filters = state.classroomFilters;

  content.innerHTML = `
    <div class="page-heading">
      <div>
        <div class="eyebrow">CLASSROOM ADMIN</div>
        <h1>教室管理</h1>
        <p>维护教室容量、类型与设备信息，并控制教室的开放状态。</p>
      </div>
      <div class="heading-actions">
        <button type="button" class="btn btn-primary" data-action="add-classroom">${icon("plus")}新增教室</button>
      </div>
    </div>

    <div id="classroom-stats" class="stats-grid stats-grid-mini"></div>

    <section class="filter-card">
      <div class="classroom-filter-grid">
        <label class="compact-field search-field">
          <span>搜索教室</span>
          <div class="search-input">${icon("search")}<input type="search" id="classroom-q" placeholder="教室名称 / 房间号 / 设备" value="${esc(filters.q)}"></div>
        </label>
        <label class="compact-field">
          <span>所属楼栋</span>
          <select id="classroom-building">
            <option value="">全部楼栋</option>
            ${(state.meta.buildings || []).map((building) => `<option value="${esc(building)}" ${filters.building === building ? "selected" : ""}>${esc(building)}</option>`).join("")}
          </select>
        </label>
        <label class="compact-field">
          <span>开放状态</span>
          <select id="classroom-status">
            <option value="all">全部状态</option>
            <option value="active" ${filters.status === "active" ? "selected" : ""}>开放中</option>
            <option value="maintenance" ${filters.status === "maintenance" ? "selected" : ""}>维护中</option>
          </select>
        </label>
      </div>
      <div class="filter-bottom">
        <div class="filter-count" id="classroom-count">正在载入教室...</div>
      </div>
    </section>

    <section class="classroom-admin-grid" id="classroom-grid">
      <div class="page-loading-inline"><span class="spinner"></span>正在载入教室...</div>
    </section>`;

  $("[data-action='add-classroom']", content)?.addEventListener("click", () => openClassroomModal());
  $("#classroom-q", content)?.addEventListener("input", () => {
    state.classroomFilters.q = $("#classroom-q").value.trim();
    renderClassroomGrid();
  });
  $("#classroom-building", content)?.addEventListener("change", () => {
    state.classroomFilters.building = $("#classroom-building").value;
    renderClassroomGrid();
  });
  $("#classroom-status", content)?.addEventListener("change", () => {
    state.classroomFilters.status = $("#classroom-status").value;
    renderClassroomGrid();
  });

  await loadClassrooms(true);
}

async function loadClassrooms(showLoading) {
  const holder = $("#classroom-grid");
  if (!holder) return;
  if (showLoading) {
    holder.innerHTML = `<div class="page-loading-inline"><span class="spinner"></span>正在载入教室...</div>`;
  }
  try {
    const data = await api("/classrooms");
    if (state.page !== "classrooms") return;
    state.classrooms = data.classrooms;
    renderClassroomStats();
    renderClassroomGrid();
  } catch (error) {
    holder.innerHTML = emptyState("alert", "读取失败", error.message);
  }
}

function renderClassroomStats() {
  const holder = $("#classroom-stats");
  if (!holder) return;
  const rooms = state.classrooms || [];
  const active = rooms.filter((room) => room.status === "active").length;
  const maintenance = rooms.filter((room) => room.status === "maintenance").length;
  const capacity = rooms.reduce((sum, room) => sum + room.capacity, 0);
  holder.innerHTML = `
    <article class="stat-card stat-free stat-card-horizontal"><div class="stat-icon">${icon("building")}</div><div class="stat-body"><span class="stat-label">教室总数</span><div class="stat-value">${rooms.length}<small>间</small></div></div></article>
    <article class="stat-card stat-busy stat-card-horizontal"><div class="stat-icon">${icon("check")}</div><div class="stat-body"><span class="stat-label">开放中</span><div class="stat-value">${active}<small>间</small></div></div></article>
    <article class="stat-card stat-book stat-card-horizontal"><div class="stat-icon">${icon("users")}</div><div class="stat-body"><span class="stat-label">总容量</span><div class="stat-value">${capacity}<small>座</small></div></div></article>
    <article class="stat-card stat-muted stat-card-horizontal"><div class="stat-icon">${icon("settings")}</div><div class="stat-body"><span class="stat-label">维护中</span><div class="stat-value">${maintenance}<small>间</small></div></div></article>`;
}

function filteredClassrooms() {
  const rooms = state.classrooms || [];
  const filters = state.classroomFilters;
  return rooms.filter((room) => {
    if (filters.building && room.building !== filters.building) return false;
    if (filters.status !== "all" && room.status !== filters.status) return false;
    if (filters.q) {
      const text = `${room.name} ${room.building} ${room.room_no} ${room.room_type} ${room.equipment.join(" ")}`.toLowerCase();
      if (!text.includes(filters.q.toLowerCase())) return false;
    }
    return true;
  });
}

function renderClassroomGrid() {
  const holder = $("#classroom-grid");
  if (!holder) return;
  const rooms = filteredClassrooms();
  const count = $("#classroom-count");
  if (count) count.textContent = `共 ${(state.classrooms || []).length} 间教室 · 当前显示 ${rooms.length} 间`;

  if (!rooms.length) {
    holder.innerHTML = emptyState("building", "没有匹配的教室", "请调整筛选条件，或新增一间教室。");
    return;
  }

  holder.innerHTML = rooms.map((room) => `
    <article class="classroom-admin-card ${room.status === "maintenance" ? "is-maintenance" : ""}">
      <div class="classroom-admin-head">
        <div class="room-icon">${icon("building")}</div>
        <div class="classroom-admin-title">
          <h3>${esc(room.name)}</h3>
          <p>${esc(room.building)} · ${esc(room.room_no)}</p>
        </div>
        <span class="status-pill ${room.status === "active" ? "status-free" : "status-maintenance"}"><i></i>${room.status === "active" ? "开放中" : "维护中"}</span>
      </div>
      <div class="classroom-admin-facts">
        <div><span>类型</span><strong>${esc(room.room_type)}</strong></div>
        <div><span>容量</span><strong>${room.capacity} 人</strong></div>
      </div>
      <div class="classroom-tags">
        ${room.equipment.length ? room.equipment.map((item) => `<span>${esc(item)}</span>`).join("") : "<span>暂无设备信息</span>"}
      </div>
      <div class="classroom-admin-foot">
        <span>ROOM ${String(room.id).padStart(2, "0")}</span>
        <button type="button" class="btn btn-ghost btn-sm" data-edit-room="${room.id}">${icon("edit")}编辑档案</button>
      </div>
    </article>`).join("");

  holder.querySelectorAll("[data-edit-room]").forEach((button) => {
    button.addEventListener("click", () => {
      const room = (state.classrooms || []).find((item) => item.id === Number(button.dataset.editRoom));
      if (room) openClassroomModal(room);
    });
  });
}

function openClassroomModal(room = null) {
  const editing = Boolean(room);
  const values = room || {
    name: "",
    building: "",
    room_no: "",
    capacity: 60,
    room_type: "多媒体教室",
    equipment: [],
    status: "active",
  };
  const roomTypes = Array.from(new Set([...(state.meta.room_types || []), "普通教室", "多媒体教室", "研讨室", "机房", "阶梯教室", "智慧教室"]));
  const body = `
    <form id="classroom-form" class="stack-form" novalidate>
      <div class="form-error" role="alert"></div>
      <div class="form-grid two">
        <label class="field">
          <span>教室名称</span>
          <input type="text" name="name" maxlength="40" value="${esc(values.name)}" placeholder="例如 综合楼 A101" autofocus required>
        </label>
        <label class="field">
          <span>所属楼栋</span>
          <input type="text" name="building" maxlength="30" value="${esc(values.building)}" placeholder="例如 综合楼" required>
        </label>
        <label class="field">
          <span>房间号</span>
          <input type="text" name="room_no" maxlength="20" value="${esc(values.room_no)}" placeholder="例如 A101" required>
        </label>
        <label class="field">
          <span>容纳人数</span>
          <input type="number" name="capacity" min="1" max="2000" value="${values.capacity}" required>
        </label>
        <label class="field">
          <span>教室类型</span>
          <input type="text" name="room_type" maxlength="30" value="${esc(values.room_type)}" list="room-type-options" required>
          <datalist id="room-type-options">${roomTypes.map((type) => `<option value="${esc(type)}"></option>`).join("")}</datalist>
        </label>
        <label class="field">
          <span>开放状态</span>
          <select name="status" required>
            <option value="active" ${values.status === "active" ? "selected" : ""}>开放中</option>
            <option value="maintenance" ${values.status === "maintenance" ? "selected" : ""}>维护中</option>
          </select>
        </label>
      </div>
      <label class="field">
        <span>教室设备</span>
        <input type="text" name="equipment" maxlength="200" value="${esc((values.equipment || []).join(","))}" placeholder="用英文逗号分隔，例如 投影,音响,空调">
        <small class="field-hint">设备信息会显示在教室总览卡片中。</small>
      </label>
      <div class="form-note">${icon("alert")}切换为维护中后，新的借用会被阻止，已生效的借用不会自动取消，请提前通知相关教师。</div>
    </form>`;

  const modal = openModal({
    title: editing ? "编辑教室" : "新增教室",
    subtitle: editing ? `${room.name} · ${room.building}` : "将教室加入可用教学空间",
    size: "modal-md",
    body,
    footer: `
      <button type="button" class="btn btn-ghost" data-cancel>取消</button>
      <button type="submit" class="btn btn-primary" form="classroom-form" data-submit>${icon("check")}${editing ? "保存修改" : "创建教室"}</button>`,
  });

  const form = modal.element.querySelector("#classroom-form");
  const submit = modal.element.querySelector("[data-submit]");
  modal.element.querySelector("[data-cancel]")?.addEventListener("click", () => modal.close());
  form?.addEventListener("input", () => clearFormError(form));

  form?.addEventListener("submit", async (event) => {
    event.preventDefault();
    clearFormError(form);
    const data = Object.fromEntries(new FormData(form).entries());
    if (!data.name.trim() || !data.building.trim() || !data.room_no.trim() || !data.room_type.trim()) {
      formError(form, "请完整填写教室名称、楼栋、房间号和类型");
      return;
    }
    const capacity = Number(data.capacity);
    if (!Number.isFinite(capacity) || capacity <= 0) {
      formError(form, "容纳人数必须大于 0");
      return;
    }
    setButtonLoading(submit, true, "保存中...");
    try {
      const endpoint = editing ? `/classrooms/${room.id}` : "/classrooms";
      await api(endpoint, {
        method: editing ? "PUT" : "POST",
        body: {
          name: data.name.trim(),
          building: data.building.trim(),
          room_no: data.room_no.trim(),
          capacity,
          room_type: data.room_type.trim(),
          equipment: data.equipment.trim(),
          status: data.status,
        },
      });
      modal.close(true);
      toast(editing ? "教室档案已更新" : "教室已创建");
      await loadMeta();
      const content = $("#page-content");
      if (content && state.page === "classrooms") {
        await renderClassroomsPage(content);
      }
    } catch (error) {
      formError(form, error.message);
      setButtonLoading(submit, false);
    }
  });
}
`````


## `static/js/pages/schedule.js`

`````javascript
/* 学生 / 教师：本周课表。 */

import {
  $,
  state,
  icon,
  esc,
  api,
  openModal,
  emptyState,
  formatDateCN,
  minutesToTime,
} from "../core.js";

const START_MIN = 480;
const END_MIN = 1320;
const SPAN = END_MIN - START_MIN;

export async function renderSchedulePage(content) {
  const isTeacher = state.user.role === "teacher";
  content.innerHTML = `
    <div class="page-heading">
      <div>
        <div class="eyebrow">WEEKLY SCHEDULE</div>
        <h1>${isTeacher ? "我的授课课表" : "我的课表"}</h1>
        <p>${isTeacher ? "查看本周授课安排与教室分布。" : "查看本班本周课程与上课教室。"}</p>
      </div>
      <div class="heading-actions">
        <span class="live-pill live-pill-static">${icon("calendar")}${esc(state.meta.semester.name)}</span>
      </div>
    </div>
    <div id="schedule-stats" class="stats-grid stats-grid-mini"></div>
    <div id="schedule-body">
      <div class="page-loading-inline"><span class="spinner"></span>正在生成课表...</div>
    </div>`;

  await loadSchedule();
}

async function loadSchedule() {
  try {
    const data = await api("/schedule");
    if (state.page !== "schedule") return;
    state.schedule = data;
    renderScheduleStats(data);
    renderSchedule(data);
  } catch (error) {
    $("#schedule-body").innerHTML = emptyState("alert", "课表读取失败", error.message);
  }
}

function renderScheduleStats(data) {
  const holder = $("#schedule-stats");
  if (!holder) return;
  holder.innerHTML = `
    <article class="stat-card stat-free stat-card-horizontal">
      <div class="stat-icon">${icon("book")}</div>
      <div class="stat-body"><span class="stat-label">本周课程</span><div class="stat-value">${data.stats.course_count}<small>门</small></div></div>
    </article>
    <article class="stat-card stat-busy stat-card-horizontal">
      <div class="stat-icon">${icon("calendar")}</div>
      <div class="stat-body"><span class="stat-label">今日课程</span><div class="stat-value">${data.stats.today_count}<small>节</small></div></div>
    </article>
    <article class="stat-card stat-book stat-card-horizontal">
      <div class="stat-icon">${icon("clock")}</div>
      <div class="stat-body"><span class="stat-label">本周课时</span><div class="stat-value">${data.stats.weekly_hours}<small>小时</small></div></div>
    </article>
    <article class="stat-card stat-muted stat-card-horizontal">
      <div class="stat-icon">${icon("pin")}</div>
      <div class="stat-body"><span class="stat-label">教学周</span><div class="stat-value">${data.semester.week > 0 ? data.semester.week : "—"}<small>/ 18 周</small></div></div>
    </article>`;
}

function renderSchedule(data) {
  const holder = $("#schedule-body");
  if (!holder) return;
  const courses = data.courses || [];
  const todayDay = data.days.find((day) => day.is_today)?.weekday || null;
  const todayCourses = (todayDay ? courses.filter((course) => course.weekday === todayDay) : [])
    .sort((a, b) => a.start_time.localeCompare(b.start_time));

  const side = `
    <aside class="schedule-side">
      <section class="side-card">
        <div class="side-card-head">
          <h3>今日课程</h3>
          <span>${formatDateCN(data.today)}</span>
        </div>
        ${todayCourses.length ? `
          <div class="today-list">
            ${todayCourses.map((course) => `
              <button type="button" class="today-item" data-course-id="${course.id}">
                <span class="today-time">${course.start_time}<em>${course.end_time}</em></span>
                <span class="today-line"></span>
                <span class="today-copy">
                  <strong>${esc(course.name)}</strong>
                  <small>${esc(course.classroom_name)} · ${esc(course.teacher_name)}</small>
                </span>
              </button>`).join("")}
          </div>` : `<div class="side-empty">${icon("clock")}<p>今天没有课程安排</p><span>享受一段自主安排的时间</span></div>`}
      </section>

      <section class="side-card term-side-card">
        <span class="semester-kicker">CURRENT SEMESTER</span>
        <h3>${esc(data.semester.name)}</h3>
        <p>当前为第 ${data.semester.week > 0 ? data.semester.week : "—"} 教学周</p>
        <div class="term-progress"><span style="width:${Math.min(100, Math.max(2, ((data.semester.week || 0) / 18) * 100))}%"></span></div>
        <div class="term-side-foot"><span>学期开始</span><strong>${formatDateCN(data.semester.start, false)}</strong></div>
      </section>
    </aside>`;

  if (!courses.length) {
    holder.innerHTML = `
      <div class="schedule-empty-layout">
        ${emptyState("calendar", "本周暂无课程", "当前教学周内没有排课安排，可前往教室总览查看空闲教室。")}
        ${side}
      </div>`;
    bindScheduleEvents(holder, data);
    return;
  }

  const hours = [8, 10, 12, 14, 16, 18, 20, 22];
  const board = `
    <section class="calendar-card">
      <div class="calendar-head">
        <div>
          <h2>本周课表</h2>
          <p>${esc(data.semester.name)} · 第 ${data.semester.week > 0 ? data.semester.week : "—"} 教学周</p>
        </div>
        <div class="calendar-legend">
          <span><i class="legend-dot course"></i>课程</span>
          <span><i class="legend-dot now"></i>当前时间</span>
        </div>
      </div>
      <div class="week-board-scroll">
        <div class="week-board">
          <div class="week-head">
            <div class="week-corner">时间</div>
            ${data.days.map((day) => `
              <div class="week-day-head ${day.is_today ? "is-today" : ""}">
                <span>${day.label}</span>
                <strong>${day.month}/${day.day}</strong>
              </div>`).join("")}
          </div>
          <div class="week-grid-body">
            <div class="week-time-axis">
              ${hours.map((hour) => `<span style="top:${((hour * 60 - START_MIN) / SPAN) * 100}%">${String(hour).padStart(2, "0")}:00</span>`).join("")}
            </div>
            ${data.days.map((day) => {
              const dayCourses = courses.filter((course) => course.weekday === day.weekday);
              const now = new Date();
              const nowLine = day.is_today && now.getHours() * 60 + now.getMinutes() >= START_MIN && now.getHours() * 60 + now.getMinutes() < END_MIN
                ? `<i class="week-now-line" style="top:${((now.getHours() * 60 + now.getMinutes() - START_MIN) / SPAN) * 100}%"><span>${minutesToTime(now.getHours() * 60 + now.getMinutes())}</span></i>`
                : "";
              return `
                <div class="week-day-lane ${day.is_today ? "is-today" : ""}">
                  ${dayCourses.map((course) => courseBlock(course)).join("")}
                  ${nowLine}
                </div>`;
            }).join("")}
          </div>
        </div>
      </div>
    </section>`;

  holder.innerHTML = `<div class="schedule-layout"><div class="schedule-main">${board}</div>${side}</div>`;
  bindScheduleEvents(holder, data);
}

function courseBlock(course) {
  const start = Number(course.start_time.split(":")[0]) * 60 + Number(course.start_time.split(":")[1]);
  const end = Number(course.end_time.split(":")[0]) * 60 + Number(course.end_time.split(":")[1]);
  const top = ((Math.max(START_MIN, start) - START_MIN) / SPAN) * 100;
  const height = ((Math.min(END_MIN, end) - Math.max(START_MIN, start)) / SPAN) * 100;
  const compact = end - start < 75;
  return `
    <button type="button" class="timetable-block ${compact ? "is-compact" : ""}" style="top:${top}%;height:${height}%" data-course-id="${course.id}" title="${esc(`${course.name} ${course.start_time}-${course.end_time}`)}">
      <span class="block-time">${course.start_time} - ${course.end_time}</span>
      <strong>${esc(course.name)}</strong>
      <span class="block-room">${icon("pin")}${esc(course.classroom_name)}</span>
      ${compact ? "" : `<small>${esc(course.teacher_name)} · ${esc(course.class_name)}</small>`}
    </button>`;
}

function bindScheduleEvents(holder, data) {
  holder.addEventListener("click", (event) => {
    const target = event.target.closest("[data-course-id]");
    if (!target) return;
    const course = (data.courses || []).find((item) => item.id === Number(target.dataset.courseId));
    if (course) openCourseDetail(course, data);
  });
}

function openCourseDetail(course, data) {
  openModal({
    title: course.name,
    subtitle: `${course.code} · ${course.weekday_label}`,
    size: "modal-sm",
    body: `
      <div class="course-detail-hero">
        <span class="course-detail-code">${esc(course.code)}</span>
        <h3>${esc(course.name)}</h3>
        <p>${esc(course.weeks_label)} · ${course.start_time} - ${course.end_time}</p>
      </div>
      <div class="detail-info-list">
        <div><span>${icon("user")}授课教师</span><strong>${esc(course.teacher_name)}</strong></div>
        <div><span>${icon("users")}授课班级</span><strong>${esc(course.class_name)}</strong></div>
        <div><span>${icon("pin")}上课教室</span><strong>${esc(course.classroom_name)}</strong></div>
        <div><span>${icon("calendar")}上课时间</span><strong>${course.weekday_label} ${course.start_time} - ${course.end_time}</strong></div>
      </div>
      <div class="form-note">${icon("info")}课程信息由教务管理员维护，如有调整请联系管理员。</div>`,
    footer: `<button type="button" class="btn btn-ghost" data-close>关闭</button>`,
    onMount(element, close) {
      element.querySelector("[data-close]").addEventListener("click", () => close());
    },
  });
}
`````


## `tests/test_services.py`

`````python
"""业务层测试：认证、空闲计算、借用与课程冲突规则。

运行：
    python -m unittest discover -s tests -v
"""

from __future__ import annotations

import shutil
import sqlite3
import unittest
from datetime import date, timedelta
from pathlib import Path

import database
import services


class ServiceTests(unittest.TestCase):
    @classmethod
    def setUpClass(cls) -> None:
        project_root = Path(__file__).resolve().parent.parent
        cls.temp_path = project_root / ".test_data"
        shutil.rmtree(cls.temp_path, ignore_errors=True)
        cls.temp_path.mkdir(parents=True, exist_ok=True)
        database.set_db_path(cls.temp_path / "test.db")
        database.init_db()

    @classmethod
    def tearDownClass(cls) -> None:
        shutil.rmtree(cls.temp_path, ignore_errors=True)

    def connect(self) -> sqlite3.Connection:
        return database.connect()

    def test_01_authentication(self) -> None:
        conn = self.connect()
        try:
            admin = services.authenticate(conn, "admin", "123456")
            self.assertEqual(admin["role"], "admin")
            self.assertEqual(admin["name"], "周明")
            with self.assertRaises(services.ApiError) as context:
                services.authenticate(conn, "admin", "wrong-password")
            self.assertEqual(context.exception.status, 401)
        finally:
            conn.close()

    def test_02_overview_and_room_order(self) -> None:
        conn = self.connect()
        try:
            data = services.get_overview(conn, date.today().isoformat(), "10:10")
            self.assertEqual(len(data["classrooms"]), 12)
            self.assertEqual(data["classrooms"][0]["name"], "综合楼 A101")
            self.assertGreaterEqual(sum(1 for room in data["classrooms"] if room["state"] == "busy"), 1)
            self.assertTrue(1 <= data["week"] <= 18)
            self.assertIn("intervals", data["classrooms"][0])
        finally:
            conn.close()

    def test_03_schedule_and_semester(self) -> None:
        conn = self.connect()
        try:
            student = services.authenticate(conn, "student", "123456")
            schedule = services.get_schedule(conn, student)
            self.assertGreater(schedule["stats"]["course_count"], 0)
            self.assertEqual(len(schedule["days"]), 7)
            self.assertTrue(all(course["class_name"] == student["class_name"] for course in schedule["courses"]))
            semester = services.semester_info(conn)
            self.assertGreaterEqual(semester["week"], 1)
        finally:
            conn.close()

    def test_04_booking_conflicts(self) -> None:
        conn = self.connect()
        try:
            teacher = services.authenticate(conn, "teacher", "123456")
            other = services.authenticate(conn, "lina", "123456")
            tomorrow = (date.today() + timedelta(days=1)).isoformat()
            payload = {
                "classroom_id": 4,
                "date": tomorrow,
                "start_time": "12:00",
                "end_time": "13:00",
                "purpose": "单元测试借用",
            }
            booking = services.create_booking(conn, teacher, payload)
            self.assertEqual(booking["classroom_name"], "综合楼 A202")
            with self.assertRaises(services.ApiError) as context:
                services.create_booking(conn, teacher, payload)
            self.assertIn("时间冲突", context.exception.message)
            with self.assertRaises(services.ApiError) as context:
                services.create_booking(conn, other, payload)
            self.assertIn("时间冲突", context.exception.message)
            cancelled = services.cancel_booking(conn, teacher, booking["id"])
            self.assertEqual(cancelled["state"], "cancelled")
        finally:
            conn.close()

    def test_05_course_conflicts_and_crud(self) -> None:
        conn = self.connect()
        try:
            admin = services.authenticate(conn, "admin", "123456")
            base = {
                "code": "TST101",
                "name": "测试课程",
                "teacher_id": 3,
                "classroom_id": 1,
                "class_name": "测试班级",
                "weekday": 2,
                "start_time": "10:05",
                "end_time": "11:40",
                "weeks_start": 1,
                "weeks_end": 18,
            }
            with self.assertRaises(services.ApiError) as context:
                services.create_course(conn, admin, base)
            self.assertIn("教室冲突", context.exception.message)

            class_conflict = {**base, "classroom_id": 12, "class_name": "计算机2301"}
            with self.assertRaises(services.ApiError) as context:
                services.create_course(conn, admin, class_conflict)
            self.assertIn("班级冲突", context.exception.message)

            created = services.create_course(
                conn,
                admin,
                {
                    **base,
                    "code": "TST102",
                    "classroom_id": 12,
                    "class_name": "测试班级",
                    "weekday": 6,
                    "start_time": "18:00",
                    "end_time": "19:30",
                },
            )
            self.assertEqual(created["weekday_label"], "周六")
            updated = services.update_course(conn, admin, created["id"], {**base, "name": "测试课程（修改）", "classroom_id": 12, "class_name": "测试班级", "weekday": 6, "start_time": "18:00", "end_time": "19:30"})
            self.assertIn("修改", updated["name"])
            services.delete_course(conn, admin, created["id"])
            with self.assertRaises(services.ApiError) as context:
                services.update_course(conn, admin, created["id"], base)
            self.assertEqual(context.exception.status, 404)
        finally:
            conn.close()

    def test_06_classroom_and_settings(self) -> None:
        conn = self.connect()
        try:
            admin = services.authenticate(conn, "admin", "123456")
            meta = services.get_meta(conn)
            self.assertEqual(meta["buildings"][0], "综合楼")
            room = services.create_classroom(
                conn,
                admin,
                {
                    "name": "测试楼 T101",
                    "building": "测试楼",
                    "room_no": "T101",
                    "capacity": 30,
                    "room_type": "研讨室",
                    "equipment": "投屏,白板",
                    "status": "active",
                },
            )
            self.assertEqual(room["building"], "测试楼")
            room = services.update_classroom(
                conn,
                admin,
                room["id"],
                {
                    "name": room["name"],
                    "building": room["building"],
                    "room_no": room["room_no"],
                    "capacity": room["capacity"],
                    "room_type": room["room_type"],
                    "equipment": "投屏,白板",
                    "status": "maintenance",
                },
            )
            self.assertEqual(room["status"], "maintenance")
            semester = services.update_semester(conn, admin, {"start": "2026-09-01", "name": "测试学期"})
            self.assertEqual(semester["start"], "2026-08-31")
        finally:
            conn.close()


if __name__ == "__main__":
    unittest.main()
`````


## `README.md`

`````markdown
# 知行教室 · 教室课程管理系统

面向 **学生、教师、教务管理员** 的轻量级教室课程协同平台。学生和教师可以实时查看教室空闲情况；教师可以直接借用空闲教室；管理员可以维护课程、教室档案、借用记录和学期设置。

系统使用 **Python 标准库 + SQLite + 原生 JavaScript** 构建，无第三方运行依赖，下载后即可启动。

![教室实时总览](docs/screenshots/overview.png)

## 功能一览

### 学生端
- 实时查看全校教室的空闲 / 使用中 / 维护中状态
- 按日期、时间、教学楼、容量、教室类型筛选
- 查看每间教室的当日课程与借用时间轴
- 查看本班本周课表、今日课程和上课教室

### 教师端
- 查看教室实时状态与当日安排
- 按日期和时段查找整段时间可借用的教室
- 一键借用空闲教室，自动校验教室、教师、班级与借用冲突
- 教师已有课程 / 借用冲突时给出最近可用时段建议
- 查看和取消自己的借用记录
- 查看本周授课课表

### 管理员端
- 课程增删改查：课程代码、名称、教师、班级、教室、星期、时间、周次
- 自动拦截教室冲突、教师冲突、班级冲突和已有借用冲突
- 教室档案维护：容量、类型、设备、开放 / 维护状态
- 查看全校教师借用记录，取消异常借用
- 设置当前学期名称与教学周起始日期

### 通用能力
- 学生 / 教师 / 管理员三种角色，按角色显示导航和操作权限
- 教室总览每 10 秒自动刷新，显示最后同步时间
- 响应式布局，支持手机、平板和桌面
- 密码 PBKDF2 加盐存储、签名会话 Cookie、CSRF 校验
- SQLite 自动建库与演示数据初始化

## 快速启动

环境要求：**Python 3.10 及以上**，无需安装任何第三方库。

```bash
python app.py
```

Windows 用户也可以直接双击 `start.bat`，脚本会自动寻找可用的 Python。

启动后访问：

```text
http://127.0.0.1:8000
```

指定端口或允许局域网访问：

```bash
python app.py --port 8001
python app.py --host 0.0.0.0 --port 8000
```

## 演示账号

登录页提供三个一键登录卡片，密码均为 `123456`。

| 角色 | 账号 | 姓名 | 身份 |
| --- | --- | --- | --- |
| 学生 | `student` | 陈晨 | 计算机2301 |
| 教师 | `teacher` | 张伟 | 计算机学院 |
| 管理员 | `admin` | 周明 | 教务管理 |

其他演示账号：教师 `lina` / `wangqiang`，学生 `liuyang` / `zhaomin`，密码同上。

> 首次启动会自动生成 12 间教室、18 门课程和今日借用数据。数据库文件位于 `data/classroom.db`。

## 页面预览

| 登录 | 借用教室 |
| --- | --- |
| ![登录](docs/screenshots/login.png) | ![借用教室](docs/screenshots/borrow.png) |

| 本周课表 | 课程管理 |
| --- | --- |
| ![课表](docs/screenshots/schedule.png) | ![课程管理](docs/screenshots/courses.png) |

<p align="center">
  <img src="docs/screenshots/mobile.png" alt="移动端教室总览" width="360">
</p>

## 业务规则

1. **教学时间**：教室开放时间为每天 `08:00-22:00`，课程与借用均限定在该区间。
2. **教学周**：课程按星期 + 周次循环生效；当前教学周由学期开始日期计算。
3. **课程冲突**：同一教室、同一教师或同一班级在同一时间不能安排两门课程。
4. **借用冲突**：借用时间不能与该教室已有课程、其他教师借用冲突，也不能与自己已有的课程 / 借用冲突。
5. **课程与借用交叉校验**：新增或修改课程时，会检查该课程在全部周次内是否与已生效借用冲突。
6. **即时生效**：教师借用提交成功后立即生效，可在使用前取消；管理员可取消异常借用。
7. **权限边界**：学生只读；教师只能管理自己的借用；课程、教室和学期设置仅管理员可修改。

## 项目结构

```text
.
├── app.py                  # 启动入口、HTTP 服务器
├── database.py             # SQLite 建表、密码工具、演示数据
├── services.py             # 认证、空闲计算、课程 / 借用业务规则
├── web.py                  # JSON API、静态资源、Cookie / CSRF
├── static/
│   ├── index.html
│   ├── styles.css
│   └── js/
│       ├── core.js         # 通用工具、API、弹窗、格式化
│       ├── app.js          # 登录、路由、应用外壳
│       └── pages/          # 总览、借用、课表、课程、教室
├── tests/test_services.py  # 业务层单元测试
├── docs/screenshots/       # 项目截图
├── start.bat               # Windows 启动脚本
└── data/                   # 运行时自动生成（已忽略）
```

## 测试

```bash
python -m unittest discover -s tests -v
```

覆盖登录认证、教室排序与空闲计算、课表、教师借用冲突、课程教室 / 班级冲突、课程增删改、教室档案和学期设置。

## API 摘要

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `POST` | `/api/login` | 登录并写入会话 Cookie |
| `GET` | `/api/me` | 当前用户与 CSRF Token |
| `GET` | `/api/overview?date=&at=` | 某日某时刻全部教室状态与时间轴 |
| `GET` | `/api/schedule` | 当前用户本周课表 |
| `GET/POST` | `/api/courses` | 课程列表 / 新增课程 |
| `PUT/DELETE` | `/api/courses/{id}` | 修改 / 删除课程 |
| `GET/POST` | `/api/classrooms` | 教室列表 / 新增教室 |
| `PUT` | `/api/classrooms/{id}` | 修改教室档案 |
| `GET/POST` | `/api/bookings` | 借用列表 / 提交借用 |
| `DELETE` | `/api/bookings/{id}` | 取消借用 |
| `GET/PUT` | `/api/settings` | 学期设置 |

所有非 GET 请求都需要登录会话和 `X-CSRF-Token` 请求头。

## 数据重置

停止服务后删除 `data/classroom.db`（以及同目录下的 `-wal`、`-shm` 文件），再次启动即可重新生成演示数据。`data/session.key` 可以保留，用于继续签发会话 Cookie。

## 部署提示

- 本项目定位为课程设计 / 小型院系内部工具，默认使用 HTTP 和 SQLite，适合单机或小规模并发。
- 如需局域网访问，使用 `--host 0.0.0.0`，并建议放在 Nginx / Caddy 反向代理和 HTTPS 之后。
- 正式使用前请替换演示账号密码，并接入学校统一身份认证；当前版本未提供账号管理界面。
- SQLite 已启用 WAL 模式，满足轻量并发；更大规模部署可将数据层替换为 PostgreSQL / MySQL。
- 管理员调整学期开始日期会重新计算当前教学周，请在新学期开始时统一设置。
`````


## `start.bat`

`````bat
@echo off
setlocal
cd /d "%~dp0"

set "DSH_PY=%USERPROFILE%\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies\python\python.exe"
if exist "%DSH_PY%" (
    "%DSH_PY%" app.py %*
    goto :end
)

where py >nul 2>nul
if not errorlevel 1 (
    py -3 app.py %*
    goto :end
)

where python >nul 2>nul
if not errorlevel 1 (
    python app.py %*
    goto :end
)

echo.
echo [ERROR] Python 3.10+ was not found.
echo Install Python from https://www.python.org/downloads/ and try again.
echo.
pause

:end
endlocal
`````


## `.gitignore`

`````text
__pycache__/
*.py[cod]
data/
.tmp/
.test_data/
.test-*/
*.db
*.db-wal
*.db-shm
*.zip
*.tar.gz
*.log
.DS_Store
Thumbs.db
`````
