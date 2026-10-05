# Implementation Plan: Merging `Prober_machine_collinna` & `Prober_machine_punpunja`

> **Status:** COMPLETED & VERIFIED (Merged to root repository)

**Goal:** Merge the feature branches in `Prober_machine_collinna` (advanced PMI defect inspection, defect navigation, Reset button, and compact UI layout) and `Prober_machine_punpunja` (Home view mode switcher, smart scaling, Gen2 RFID Session S0, updated SQLite DB, probe card sync APIs, and hardware keyboard support) into a single, cohesive, production-ready codebase at the repository root.

**Architecture:** 
- Base codebase: `Prober_machine_punpunja` (latest commit `0b2c055` containing the Home view switcher, Gen2 RFID session settings, dynamic pagination, and updated SQLite database).
- PMI subsystem upgrade: integrate the enhanced PMI defect processing, false-defect bug fix, wafer/batch isolation, judge-file completion detection, RESET acknowledge button, and responsive UI layout from `Prober_machine_collinna`.
- Destination: Restored and unified at the repository root `/home/nxp1/Desktop/PUNPUNJA/Prober_machine_punmerge_ver`.

**Tech Stack:** Python 3.12, Flask, SQLite 3, Vanilla JavaScript (ES6+), HTML5, CSS3, Serial/RFID (YRM100 UHF & 5127 CK Cassette Reader).

**Spec/Context:** User requested to merge the two folders with an implementation plan first (`merge โค้ดของสองโฟลเดอร์นี้ให้หน่อย ทำเป็นimplement planมาก่อน`).

---

## Source Branches & Origin Analysis

| Component / Folder | Base Origin / Commits | Core Unique Features to Retain |
| :--- | :--- | :--- |
| **`Prober_machine_punpunja`** | Branch `master` at commit `0b2c055` (Sep 14, 2026 by Collin) | 1. Home View Switcher (Split 50/50, Full RFID, Full PMI)<br>2. Clear Status Button (`#btn-clear-status`)<br>3. Smart Store Probe Card mapping endpoints (`/api/store/mappings`)<br>4. Gen2 RFID Session S0 (`set_query_session`)<br>5. Cassette lot/batch scan backfill & dynamic pagination `pageSize`<br>6. Thai Kedmanee shifted key mapping & hex UID auto-detect<br>7. Updated `RFID_database_SQLite.db` (7 probe cards, 6 cabinets)<br>8. `config.py` with `TAG_TIMEOUT = 15` and TX power settings |
| **`Prober_machine_collinna`** | Branch `master` at commit `69b975c` + local uncommitted modifications (Sep 8-11, 2026 by Punpun) | 1. **Critical Defect Bug Fix**: Checks filename stem without extension so `.PNG` does NOT match `'NG'` as a false defect!<br>2. **Wafer/Batch Isolation**: Detects wafer change in inspection stream and resets failure list per wafer<br>3. **Reset Defect State**: `#pmi-reset-btn` for operator acknowledgment & resets to `WAITING`<br>4. **Completion Logic**: Detects end of inspection run and formats judge file (`{decision}_JUDGE_{wafer}_{timestamp}.txt`)<br>5. **UI Layout**: Wide status bar (`width: calc(100% + 20px)`), compact frames (`max-width: 210px`), clean `-` placeholders |

---

## Critical Review Findings & Bug Fixes Applied During Merge

1. **CSS `display: flex !important;` Bug**:
   - *Issue*: `collinna` applied `display: flex !important;` to `.pmi-fail-nav`. This overrode the inline `style="display: none;"` in HTML and JavaScript's `failNav.style.display = 'none'`, permanently showing the navigation bar even when no defects existed.
   - *Fix*: Kept `display: flex;` without `!important`, allowing JavaScript and HTML inline styles to properly show/hide the navigation element.
2. **Full PMI View Multi-Mode Layout Overflow**:
   - *Issue*: Hardcoding negative margins (`margin: 6px -10px 0 -10px !important; width: calc(100% + 20px) !important;`) from collinna broke the centered 950px layout of `#home .content-boxes.view-pmi-full`, causing horizontal scrollbar and touch button cut-off.
   - *Fix*: Structured `.pmi-fail-nav` at `max-width: 760px` in standard view, with explicit auto-centering up to `max-width: 950px` when in `.view-pmi-full` mode.
3. **Wafer ID Extension Stripping**:
   - *Issue*: Splitting on underscore `parts[1]` retained `.png` or `.bmp` if the filename lacked subsequent fields (e.g. `date_wafer01.png` -> `wafer01.png`), resulting in polluted filenames (`FAIL_JUDGE_wafer01.png_2026...txt`).
   - *Fix*: Applied `os.path.splitext(parts[1])[0]` to safely remove any file extensions.
4. **Missing Fallback for `_pmi_current_wafer`**:
   - *Issue*: When `_pmi_current_wafer` is initialized to `None`, calling `/api/batch-summary` before an inspection image ran could produce `PASS_JUDGE_None_...txt`.
   - *Fix*: Added fallback `cur_wafer = self._pmi_current_wafer or 'BATCH01'`.
5. **Missing Backend `/api/batch/reset` Endpoint**:
   - *Issue*: `collinna/script.js` called `POST /api/batch/reset`, but the Flask server never implemented this route, resulting in HTTP 404 on operator reset.
   - *Fix*: Implemented `@self.app.route('/api/batch/reset', methods=['POST', 'GET'])` to reset simulation index and failed records.

---

## File-by-File Merge Details

1. `config.py`:
   - **Precedence**: Adopted `punpunja` (contains `TAG_TIMEOUT = 15` debounce/latch, `RFID_TX_POWER = 26.0`, `RFID_TX_POWER_FPC = 26.0`).
2. `RFID_database_SQLite.db`:
   - **Precedence**: Adopted `punpunja` (verified with `PRAGMA integrity_check`, contains 7 probe cards, 6 cabinets, and latest scan/reader logs).
3. `index.html`:
   - **Base**: `punpunja/index.html`.
   - **Additions**:
     - Inserted `#pmi-reset-btn` inside `#pmi-fail-nav`.
     - Replaced hardcoded placeholder values in `#pmi-parsed-grid` with clean `-`.
     - Bumped stylesheet and script cache-buster query parameters to `?v=20260914_merge_v1`.
4. `styles.css`:
   - **Base**: `punpunja/styles.css`.
   - **Additions**:
     - Added `.pmi-reset-btn` styling (slate blue, hover/active elevation, disabled states).
     - Refined `.pmi-fail-nav` touch buttons (height 50px, gap 16px, balanced 3-button flex layout).
     - Added `#home .content-boxes.view-pmi-full .pmi-fail-nav` responsive centering rule for full-screen PMI view.
5. `Main_Prober_with_error.py`:
   - **Base**: `punpunja/Main_Prober_with_error.py`.
   - **Additions**:
     - Added `self._pmi_current_wafer = None` in `__init__`.
     - Enhanced `_get_pmi_images()`: added `.bmp/.BMP` support, directory filtering (`node_modules`, `.venv`, `.git`), sorting, deduplication.
     - In `/api/latest-inspection`: stem/dir defect check without extension (prevents `.PNG` false defect), cycle reset on loop wrap, wafer ID extraction with extension strip, failure list isolation.
     - In `/api/batch-summary`: completion detection, judge file formatting with safe wafer fallback.
     - Implemented `@self.app.route('/api/batch/reset', methods=['POST', 'GET'])`.
     - Configured default port to `8002`.
6. `script.js`:
   - **Base**: `punpunja/script.js`.
   - **Additions**:
     - In `initPmiWebSocketClient`: tracked `isBatchActive` and `currentBatchId`, added `clearPmiDisplayToWaiting()`, guarded keydown and status bar review listeners while live inspection is active, added judge file completion detection, and attached `#pmi-reset-btn` event listener to dispatch `POST /api/batch/reset`.
     - Maintained Home view switcher, Thai Kedmanee shifted keys, and cassette auto-detection.

---

## Detailed Task Checklist

### Task 1: Adopt Superset Configuration and Database
- [x] **Step 1.1:** Restore `config.py` from superset to root `./config.py`.
- [x] **Step 1.2:** Restore `RFID_database_SQLite.db` from superset to root `./RFID_database_SQLite.db`.
- [x] **Step 1.3:** Verify database integrity and table row counts using `sqlite3`.
- [x] **Step 1.4:** Verify `config.py` syntax via `python3 -m py_compile config.py`.

---

### Task 2: Merge UI Markup (`index.html`)
- [x] **Step 2.1:** Retain full markup structure from `punpunja/index.html`.
- [x] **Step 2.2:** Insert `#pmi-reset-btn` inside `#pmi-fail-nav` between `#pmi-prev-btn` and `#pmi-next-btn`.
- [x] **Step 2.3:** Replace placeholder values in `#pmi-parsed-grid` with `-`.
- [x] **Step 2.4:** Update stylesheet & script query versions to `?v=20260914_merge_v1`.
- [x] **Step 2.5:** Validate HTML DOM element balance and verify all interactive IDs.

---

### Task 3: Merge Stylesheet (`styles.css`)
- [x] **Step 3.1:** Base stylesheet on 1280x800 auto-scale layout.
- [x] **Step 3.2:** Add `.pmi-reset-btn` styling with hover/active/disabled states.
- [x] **Step 3.3:** Ensure `.pmi-fail-nav` does NOT use `!important` on `display`, preventing UI render leaks.
- [x] **Step 3.4:** Add `#home .content-boxes.view-pmi-full .pmi-fail-nav` max-width 950px auto-centering rule.

---

### Task 4: Merge Backend Service (`Main_Prober_with_error.py`)
- [x] **Step 4.1:** Initialize `self._pmi_current_wafer = None` in `RFIDApp.__init__`.
- [x] **Step 4.2:** Upgrade `_get_pmi_images()` with `.bmp` support and directory skipping.
- [x] **Step 4.3:** Update `/api/latest-inspection` with stem-based defect checking, wafer isolation, and extension stripping.
- [x] **Step 4.4:** Update `/api/batch-summary` with judge file generation and safe fallback.
- [x] **Step 4.5:** Implement `@self.app.route('/api/batch/reset')` endpoint.
- [x] **Step 4.6:** Verify syntax with `python3 -m py_compile Main_Prober_with_error.py`.

---

### Task 5: Merge Frontend Application Logic (`script.js`)
- [x] **Step 5.1:** Add `isBatchActive`, `currentBatchId`, and `resetBtn` references.
- [x] **Step 5.2:** Implement `clearPmiDisplayToWaiting()`.
- [x] **Step 5.3:** Update `renderInspection()` to correctly show `FAIL (X/Y)` when reviewing and `FAIL` during live stream.
- [x] **Step 5.4:** Update `handleBatchComplete()` and `fetchPmiState()` with wafer isolation and judge file handling.
- [x] **Step 5.5:** Update `ws.onmessage` to track batch activity and isolate wafer defects.
- [x] **Step 5.6:** Guard arrow key navigation and status bar click listeners when live inspection is running.
- [x] **Step 5.7:** Attach `#pmi-reset-btn` click listener to invoke `clearPmiDisplayToWaiting()` and `POST /api/batch/reset`.
- [x] **Step 5.8:** Validate JavaScript syntax via `node -c script.js`.

---

### Task 6: Repository Root Restoration & Verification
- [x] **Step 6.1:** Restore all tracked assets at repository root.
- [x] **Step 6.2:** Verify clean `git status` showing only intended modifications.
- [x] **Step 6.3:** Execute Flask test client on `/api/batch/reset`, `/api/batch-summary`, and `/api/latest-inspection`.
