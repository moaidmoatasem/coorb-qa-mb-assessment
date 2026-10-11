# Task 4 — Automation Design: Bank Operations Console

> **Note:** the reference page `Bank_Admin_Console_Demo.html` was not attached to the email I received. I designed from the written description in the brief. If the real page differs (for example, it has no stable attribute on the wrappers), the design still holds — section 2 explains the fallback.

## The problem in one sentence

Bank staff change sensitive settings (limits, fees, KYC thresholds) on a console where the field IDs change while you use it, the page re-draws when you switch tabs, and the test environment is shared with other teams — so the tests must find fields by **meaning**, not by ID, and must **never leave the environment changed**.

## My approach in three ideas

1. **Find a field the way a person does — by its label.** Every setting sits in a stable wrapper with a visible label. I locate the wrapper by its label (or its descriptive attribute), then the control inside it. The changing `slot="field-145"` ID is never used.
2. **One "field" building block for every setting.** Text box, checkbox or dropdown — the test always does the same two things: `set(value)` and `get()`. The building block handles the difference.
3. **Settings are data, not code.** The list of tabs and fields lives in a simple data file. Adding a field means adding one line of data, not writing new code.

Everything below is these three ideas applied to the six areas the brief asks for.

---

## 1. Project structure and Page Object Model

```
ops-console-tests/
├── tests/
│   └── test_settings_persist.py     # the scenarios — short, readable, no locators
├── pages/
│   ├── console_page.py              # the shell: open a tab, Save, reload
│   └── tab.py                       # one generic class used for every tab
├── components/
│   └── field.py                     # finds a field by label; set() / get() for all 3 control types
├── data/
│   ├── fields.yaml                  # every tab and its fields: label + control type
│   └── test_values.yaml             # which 12+ fields to change, across 4+ tabs
├── support/
│   ├── settings_backup.py           # save current values before, restore after
│   └── config.py                    # environment URL, users — from environment variables
└── reports/                         # created by the run, not stored in git
```

**Three layers, each with one job:**

| Layer | Job | Knows about locators? |
| --- | --- | --- |
| Test | *What* to change and what to check | No |
| Page (console, tab) | *Where* things are: tabs, Save, reload | Only for tabs and Save |
| Component (field) | *How* to work one field of any type | Yes — the only place |

**How I avoid one 900-line class:** there is no `LimitsTab`, `FeesTab`, `KycTab`… with a method per field. There is **one** `Tab` class and **one** `Field` component, and the fields come from the data file. If a tab one day has something truly different (e.g. an editable table), only that tab gets its own small class.

## 2. Locator strategy

**Step 1 — find the wrapper (stable):**
- First choice: the wrapper's unique descriptive attribute (e.g. `data-setting="daily-limit"`).
- Fallback: the wrapper that contains the **exact label text**, searched **only inside the open tab** — so a label like "Enabled" on two tabs can't be confused.
- If a field has neither, I ask the developers to add a test attribute. It's a small change for them and saves a lot of maintenance for us.

**Step 2 — find the control inside the wrapper (by type, not by ID):** each wrapper holds one control, so I look for "the checkbox", "the dropdown" or "the text box" inside it.

**Step 3 — one interface for three control types:**

| Control type | `set(value)` does | `get()` returns |
| --- | --- | --- |
| Text box | Clear, then type the value | The text in the box |
| Checkbox | Tick or untick **to the wanted state** (never a blind click, which just toggles) | Ticked or not |
| Dropdown | Choose the option by its visible text | The chosen option's text |

So every test line looks the same, whatever the control:

```
console.tab("Limits").field("Daily transfer limit").set("25000")
```

**A small sample** (illustration only, not a framework). The data file says which fields exist:

```yaml
# data/fields.yaml
Limits:
  - { label: "Daily transfer limit",  type: text }
  - { label: "Per-transaction limit", type: text }
Notifications:
  - { label: "Send SMS on transfer",  type: checkbox }
  - { label: "Default channel",       type: dropdown }
```

The `Field` building block (Playwright, Python) is the only place that knows about control types:

```python
class Field:
    def __init__(self, page, tab_panel, label, kind):
        # 1. Find the stable wrapper by its label, only inside the open tab.
        #    (".setting" stands for the wrapper's real class or attribute.)
        self.box = tab_panel.locator(".setting",
                                     has=page.get_by_text(label, exact=True))
        self.kind = kind

    def set(self, value):
        # 2. Find the control inside the wrapper by its role, never by id.
        if self.kind == "text":
            self.box.get_by_role("textbox").fill(value)
        elif self.kind == "checkbox":
            self.box.get_by_role("checkbox").set_checked(value)  # to a state, not a click
        elif self.kind == "dropdown":
            self.box.get_by_role("combobox").select_option(label=value)

    def get(self):
        if self.kind == "text":
            return self.box.get_by_role("textbox").input_value()
        if self.kind == "checkbox":
            return self.box.get_by_role("checkbox").is_checked()
        if self.kind == "dropdown":
            return self.box.locator("option:checked").inner_text()
```

And the test reads like the requirement:

```python
def test_settings_persist_after_reload(console, changes):   # 12+ changes, 4+ tabs
    for c in changes:
        console.tab(c.tab).field(c.label).set(c.value)
    console.save()
    console.reload()
    for c in changes:
        assert console.tab(c.tab).field(c.label).get() == c.value
```

## 3. Handling the dynamic DOM

**What I do:**
- **Find the field fresh every time I use it.** I never store an element and reuse it later. When the page re-draws after a tab switch or scroll, the next action finds the new element. This removes "stale element" errors at the root.
- **Wait for a condition, not for time:** after switching tab → wait until that tab is marked selected and its content is visible. Before typing → wait until the field is visible and enabled. After Save → wait for the save confirmation (or the save response from the server).
- **Scroll the field into view** before using it.

**What I don't do:**
- No fixed waits (`sleep(3)`).
- No dynamic IDs (`slot="field-…"`), no positions like "the 7th input", no long paths copied from the page structure.
- No automatic re-runs to hide a timing problem. A flaky test is fixed or set aside with an owner — not re-run until it passes.

## 4. Test data strategy

**Where data lives:** field definitions and test values in the data files above. Environment URL and users in environment variables / the CI secret store — never in the code.

**Values unique per run:** each run gets a run ID (date + time + short random). Text values use it, e.g. a sender name `QA-20261011-0930-k3f`. For numbers and options, I read the current value first and set a **different** valid one (current limit 20,000 → set 20,500). If the new value equals the old one, the test proves nothing.

**Shared environment, restricted data, no database reset — how this shapes the design:**

| Constraint | What I do |
| --- | --- |
| Can't reset the database | **Back up, then restore.** Before the test, save the current value of every field I'll change. After the test — pass or fail — put them all back. The environment ends exactly as I found it. |
| Other teams use the same settings | Agree with them on which fields are safe to change in this environment, and keep values in a safe range. A changed KYC threshold could break their tests. |
| Two runs could change the same setting | Tests that change settings run **one at a time** per environment (a lock in CI). Tests that only read can run in parallel. |
| Restricted data and audit | Use a dedicated automation user, so the audit log clearly shows the changes came from tests. |

## 5. Reusability and scalability

**What a new engineer writes:**
- **A 20th field** on an existing tab: one line in `fields.yaml` (label + type). No code.
- **A 5th tab:** a new section in `fields.yaml`. The generic `Tab` class already works for it. Code only if there's a new *type* of control (e.g. a date picker) — one new control type, then every tab can use it.

**The required scenario, written once and driven by data:**

```
for each change in test_values:            # 12+ fields, 4+ tabs
    open the tab, set the field
save and wait for confirmation
reload the page
for each change in test_values:
    open the tab, check the field shows the new value
also check the fields I did NOT change are still the same
```

**Growing from 7 tabs to 20:**
- **Speed:** check every field through the UI only where it matters; use the backend config API (if available) for fast bulk checks. Keep one full UI journey per tab.
- **Parallel runs:** read-only tests run in parallel; tests that change settings run in sequence (see section 4).
- **CI:** a small smoke set on every pull request (one field of each type + save + reload). The full suite nightly and before every release.
- **Reporting:** for every failure — screenshot, step trace, and the field's old → new → after-reload value, so a developer sees the problem without re-running.

## 6. Tooling

**Choice:** Playwright with Python and pytest; Allure (or Playwright's HTML report) for results; GitHub Actions or GitLab CI.

**Why:**
- Playwright finds elements fresh and waits automatically — this directly fits the dynamic-DOM problem.
- Finding "the wrapper with this label" and "the checkbox inside it" is short and readable in Playwright.
- Python + pytest is easy to read for QA engineers who aren't full-time developers, and pytest makes "back up before, restore after" simple.
- Screenshots and traces on failure make debugging fast.

**What the tool already does, and what my design adds:**

| Need | Playwright already gives it | My design adds |
| --- | --- | --- |
| Find by meaning, not id | `get_by_label`, `get_by_role`, `get_by_text` | Searching only inside the open tab, so the same label on two tabs can't clash |
| No stale elements | Finds the element again on every action | Nothing extra; I just never store elements |
| Waiting | Auto-waits until visible, enabled and stable | Waiting for business states: tab selected, save confirmed |
| Three control types | Different methods: `fill`, `set_checked`, `select_option` | One `set` / `get` for all of them |
| 20 tabs, hundreds of fields | — | Fields kept in a data file, back-up-and-restore, CI split |

If the console links labels to controls properly (accessible HTML), `page.get_by_label("Daily transfer limit")` can find the control directly, and the wrapper step becomes even simpler. The design works either way.

**Trade-offs I accept:**
- Playwright's newest features usually reach TypeScript first. I accept that for a stack the whole QA team can read and maintain.
- Fields in a data file means mistakes in that file are found when tests run, not while typing. I accept it because it keeps the code small; a quick format check on the data file in CI catches most mistakes.
- Restoring settings and running setting-changing tests one at a time makes the suite slower. In a shared environment, I choose slower and safe over fast and breaking other teams' work.
