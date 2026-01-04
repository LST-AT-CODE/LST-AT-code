===============================
プロジェクト構成
===============================
## Hi there 👋

<!--
**LST-AT-CODE/LST-AT-code** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

Here are some ideas to get you started:

- 🔭 I’m currently working on ...
- 🌱 I’m currently learning ...
- 👯 I’m looking to collaborate on ...
- 🤔 I’m looking for help with ...
- 💬 Ask me about ...
- 📫 How to reach me: ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...
-->
===============================
main.py
===============================
import sys
from app.config import AppConfig
from app.db.connection import Database
from app.db.repositories import EventRepo, TagStatusRepo
from app.services.rfid.stub_reader import StubRfidReader
from app.domain.logic import process_reads
from app.ui.cli import show_status, show_events, confirm_present, confirm_missing

def main():
    cfg = AppConfig()
    db = Database(cfg.db_path)
    db.init_schema(cfg.schema_path)

    event_repo = EventRepo(db)
    status_repo = TagStatusRepo(db)

    if len(sys.argv) >= 2:
        cmd = sys.argv[1]
        if cmd == "status":
            show_status(status_repo)
            return
        if cmd == "events":
            show_events(event_repo)
            return
        if cmd == "confirm-present":
            confirm_present(event_repo, sys.argv[2])
            return
        if cmd == "confirm-missing":
            confirm_missing(event_repo, sys.argv[2])
            return

    reader = StubRfidReader()
    reader.start(lambda reads: process_reads(reads, event_repo, status_repo))
    print("PHASE1 running. Ctrl+C to stop.")
    try:
        reader.join()
    except KeyboardInterrupt:
        reader.stop()

if __name__ == "__main__":
    main()

===============================
requirements.txt
===============================
# standard library only

===============================
app/config.py
===============================
class AppConfig:
    db_path = "data/app.db"
    schema_path = "app/db/schema.sql"

===============================
app/utils/logging.py
===============================
import logging

def setup():
    logging.basicConfig(level=logging.INFO)

===============================
app/db/schema.sql
===============================
CREATE TABLE IF NOT EXISTS tag_status (
  tag_id TEXT PRIMARY KEY,
  first_seen TEXT,
  last_seen TEXT,
  dwell_seconds INTEGER
);

CREATE TABLE IF NOT EXISTS events (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  ts TEXT,
  event_type TEXT,
  tag_id TEXT,
  note TEXT
);

===============================
app/db/connection.py
===============================
import sqlite3
from pathlib import Path

class Database:
    def __init__(self, path):
        Path(path).parent.mkdir(parents=True, exist_ok=True)
        self.conn = sqlite3.connect(path)
        self.conn.row_factory = sqlite3.Row

    def init_schema(self, schema_path):
        with open(schema_path, "r", encoding="utf-8") as f:
            self.conn.executescript(f.read())
        self.conn.commit()

    def close(self):
        self.conn.close()

===============================
app/db/repositories.py
===============================
from datetime import datetime

class EventRepo:
    def __init__(self, db):
        self.db = db

    def add(self, event_type, tag_id, note=""):
        self.db.conn.execute(
            "INSERT INTO events(ts,event_type,tag_id,note) VALUES(?,?,?,?)",
            (datetime.now().isoformat(), event_type, tag_id, note),
        )
        self.db.conn.commit()

    def latest(self, limit=100):
        cur = self.db.conn.execute(
            "SELECT * FROM events ORDER BY id DESC LIMIT ?", (limit,)
        )
        return cur.fetchall()

class TagStatusRepo:
    def __init__(self, db):
        self.db = db

    def get(self, tag_id):
        cur = self.db.conn.execute(
            "SELECT * FROM tag_status WHERE tag_id=?", (tag_id,)
        )
        return cur.fetchone()

    def upsert(self, tag_id, first_seen, last_seen, dwell):
        self.db.conn.execute(
            """
            INSERT INTO tag_status(tag_id,first_seen,last_seen,dwell_seconds)
            VALUES(?,?,?,?)
            ON CONFLICT(tag_id) DO UPDATE SET
              last_seen=excluded.last_seen,
              dwell_seconds=excluded.dwell_seconds
            """,
            (tag_id, first_seen, last_seen, dwell),
        )
        self.db.conn.commit()

    def all(self):
        cur = self.db.conn.execute(
            "SELECT * FROM tag_status ORDER BY last_seen DESC"
        )
        return cur.fetchall()

===============================
app/domain/logic.py
===============================
from datetime import datetime

def process_reads(reads, event_repo, status_repo):
    for r in reads:
        now = datetime.now()
        prev = status_repo.get(r["tag_id"])

        if prev is None:
            first = now
            dwell = 0
        else:
            first = datetime.fromisoformat(prev["first_seen"])
            dwell = int((now - first).total_seconds())

        status_repo.upsert(
            r["tag_id"],
            first.isoformat(),
            now.isoformat(),
            dwell,
        )

        event_repo.add("read", r["tag_id"])

===============================
app/services/rfid/interface.py
===============================
class RfidReader:
    def start(self, callback):
        raise NotImplementedError

    def stop(self):
        raise NotImplementedError

    def join(self):
        raise NotImplementedError

===============================
app/services/rfid/stub_reader.py
===============================
import threading
import time
import random

class StubRfidReader:
    def __init__(self):
        self._stop = False
        self._thread = None
        self.tags = ["TAG-001", "TAG-002", "TAG-003"]

    def start(self, callback):
        def run():
            while not self._stop:
                reads = [{"tag_id": random.choice(self.tags)}]
                callback(reads)
                time.sleep(2)
        self._thread = threading.Thread(target=run, daemon=True)
        self._thread.start()

    def stop(self):
        self._stop = True

    def join(self):
        while self._thread.is_alive():
            time.sleep(0.5)

===============================
app/ui/cli.py
===============================
def show_status(repo):
    for r in repo.all():
        print(
            r["tag_id"],
            r["dwell_seconds"],
            r["last_seen"],
        )

def show_events(repo):
    for r in repo.latest():
        print(
            r["ts"],
            r["event_type"],
            r["tag_id"],
            r["note"],
        )

def confirm_present(repo, tag_id):
    repo.add("manual_present", tag_id)
    print("confirmed present:", tag_id)

def confirm_missing(repo, tag_id):
    repo.add("manual_missing", tag_id)
    print("confirmed missing:", tag_id)
