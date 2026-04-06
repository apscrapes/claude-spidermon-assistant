---
name: spidermon-assistant
description: >
  The all-in-one assistant for Spidermon, the open-source monitoring framework for
  Scrapy spiders. Use this skill whenever a user mentions Spidermon, spider monitoring,
  scrapy monitors, item validation, data quality monitoring for web scraping, field
  coverage, scraping alerts, JSON schema for scraped items, or monitor suites. Also
  trigger when a user wants to: generate a validation schema from sample data, set up
  monitoring for a Scrapy spider, review or improve existing Spidermon configuration,
  create expression monitors from plain English rules, or debug Spidermon failures from
  log output. Even if the user does not mention "Spidermon" by name, trigger this skill
  when they discuss monitoring Scrapy spiders, validating scraped items, or setting up
  alerts for web scraping jobs. This skill interactively gathers project name, spider
  name, item type (dict/dataclass/attrs/Scrapy Item), and scrapy-poet usage, then
  generates fully copy-pasteable code with zero placeholders.
---

# Spidermon Assistant

A unified assistant for every Spidermon task: generating schemas, bootstrapping monitors,
reviewing configuration, building expression monitors, and troubleshooting failures.

**Key principle:** Before generating any code, always gather the user's project context
interactively (project name, spider name, item type, scrapy-poet usage) so that ALL
output is directly copy-pasteable with zero modifications needed.

## Skill structure

```
spidermon-assistant/
├── SKILL.md                              ← Routing logic and workflows (you are here)
└── references/
    ├── spidermon-knowledge-base.md       ← All settings, monitors, actions, schema patterns
    └── output-templates.md               ← Code templates for generated files
```

**When to read what:**

- `references/spidermon-knowledge-base.md` — always read before generating any output.
  Contains the complete reference for settings, built-in monitors, actions, JSON schema
  patterns, expression monitor syntax, and common Scrapy stats. This is the source of truth.
- `references/output-templates.md` — read when generating `monitors.py`, `settings.py`,
  or JSON schema files. Contains validated, copy-pasteable templates with variants for:
  plain dict items, typed items (dataclass/attrs/Scrapy Item), and scrapy-poet projects.
  **Select the correct template variant** based on the user's answers from Step 0.

## Step 0: Interactive Elicitation (ALWAYS DO THIS FIRST)

Before starting any workflow, you MUST gather the user's project context so that all
generated output is directly copy-pasteable with zero edits. Use the `ask_user_input_v0`
tool to ask these questions interactively — do NOT skip this step.

### What to gather

Ask the following using `ask_user_input_v0`. You can combine into 1-3 tool calls
(respecting the 3-question-per-call limit). Adjust based on what the user has already
provided in their message — don't re-ask things you can infer.

**Round 1** (always ask unless already provided):

| Question                          | Options                                                   | Why it matters                                                         |
|-----------------------------------|-----------------------------------------------------------|------------------------------------------------------------------------|
| What is your Scrapy project name? | *(free text — ask in prose before/after the tool call)*   | Used in `SPIDERMON_SPIDER_CLOSE_MONITORS` path: `"<project>.monitors.SpiderCloseMonitorSuite"` |
| What is your spider name?         | *(free text — ask in prose before/after the tool call)*   | Used in docstrings, file headers, and monitor naming                   |
| What item type do you use?        | `Plain dict`, `dataclass`, `attrs`, `Scrapy Item`         | Determines field coverage prefix and import style                      |
| What is your Scrapy version?      | *(free text — ask user to run `pip show scrapy` or `uv pip show scrapy`)* | **Critical for compatibility.** Scrapy 2.13+ deprecated the `spider` argument from pipeline `process_item()`, which breaks Spidermon ≤1.25.0's `ItemValidationPipeline` ([tracked issue](https://github.com/scrapinghub/spidermon/issues/473)). If user is on Scrapy 2.13+, proactively warn them and recommend downgrading Scrapy (see Cross-workflow behavior section for full details and downgrade steps). |
| How many items do you expect from a successful scrape? | *(free text — ask in prose)* | Used to set the `minimum_threshold` in `ItemCountMonitor`. Do NOT default to 100 — always ask. If user is unsure, suggest they run the spider once without Spidermon and check `item_scraped_count` in the stats. |

**Round 2** (conditional — ask only what's needed):

| Condition                             | Question                                    | Options                     | Why it matters                                                    |
|---------------------------------------|---------------------------------------------|-----------------------------|-------------------------------------------------------------------|
| Item type is NOT "Plain dict"         | What is your item class name?               | *(free text)*               | Used as field coverage prefix: `"ProductItem/name"` vs `"dict/name"` |
| Always                                | Are you using scrapy-poet?                  | `Yes`, `No`                 | Changes imports, item handling, and pipeline config                |
| If using scrapy-poet                  | What is your Page Object / item model name? | *(free text)*               | For import paths and type references                              |
| Always                                | Do you want an HTML report after each run?  | `Yes`, `No`                 | If yes, include `CreateFileReport` action in the monitor suite, add `jinja2` to required packages, and generate report settings (`SPIDERMON_REPORT_TEMPLATE`, `SPIDERMON_REPORT_CONTEXT`, `SPIDERMON_REPORT_FILENAME`). The report is a styled HTML page showing monitor pass/fail results, spider stats, and validation errors. |

### How to ask

Use `ask_user_input_v0` for the multiple-choice questions (item type, scrapy-poet yes/no)
and ask the free-text questions (project name, spider name, class name) conversationally
in the surrounding message. Example flow:

```
Assistant: "I'd love to help you set up Spidermon! Before I generate anything, I need
a few details so the code is ready to paste directly into your project.

What is your Scrapy **project name** (the top-level Python package, e.g. `myproject`),
your **spider name**, and roughly **how many items** do you expect from a successful
scrape?

Also, what **Scrapy version** are you on? You can check with:
- pip: `pip show scrapy`
- uv: `uv pip show scrapy`"

[ask_user_input_v0 with item type question]
```

Then based on answers, ask follow-ups (class name, scrapy-poet).

### Context object

After gathering answers, mentally construct this context (do not show it to the user):

```
project_name:         e.g. "ecommerce_scraper"
spider_name:          e.g. "products"
item_type:            "dict" | "dataclass" | "attrs" | "scrapy_item"
item_class_name:      e.g. "ProductItem" (or "dict" if plain dict)
uses_scrapy_poet:     true | false
page_object_name:     e.g. "ProductPage" (only if scrapy-poet)
scrapy_version:       e.g. "2.12.0" — if >= 2.13, warn about compatibility and recommend downgrade (see Cross-workflow behavior)
expected_item_count:  e.g. 50 — used as ItemCountMonitor threshold
wants_html_report:    true | false — if true, include CreateFileReport action and jinja2 dependency
```

Use this context to fill ALL templates — replace every placeholder. The user should be
able to copy-paste the output directly into their project with zero modifications.

### Scrapy-poet implications

When `uses_scrapy_poet` is true:

1. **Imports change**: Items come from the user's page object module, not a traditional
   `items.py`. The item is typically a `@attrs.define` or `@dataclass` class defined
   alongside or returned by the Page Object.

2. **Item validation pipeline** still works the same way — Spidermon validates whatever
   items the spider yields, regardless of how they're produced.

3. **Field coverage prefix** uses the item class name (e.g., `ProductItem/name`), same
   as any typed item. Scrapy-poet items are almost never plain dicts.

4. **Schema mapping** for multiple item types uses the **actual imported class** as the
   dict key — never a string path. Using a string causes `AttributeError: 'str' object
   has no attribute '__name__'`:
   ```python
   from myproject.items import ProductItem
   SPIDERMON_VALIDATION_SCHEMAS = {
       ProductItem: "./schemas/product_schema.json",  # class object, NOT a string
   }
   ```

5. **Common pattern**: scrapy-poet users often use `attrs` classes. If the user says
   they use scrapy-poet, default to asking if their items are `attrs` or `dataclass`.

---

## Step 1: Prerequisites Overview (ALWAYS PRESENT BEFORE GENERATING CODE)

After completing Step 0 elicitation and before generating any files, ALWAYS present a
clear prerequisites summary to the user. This tells them upfront exactly what they need
to install and what files/folders they need to create. This avoids surprises mid-setup.

### What to include

**1. Required packages** — list every package the user needs to install, with both pip
and uv commands. The list is conditional on the user's answers from Step 0:

| Package       | Always needed? | Condition                        |
|---------------|----------------|----------------------------------|
| `spidermon`   | Yes            | —                                |
| `jsonschema`  | Yes            | —                                |
| `jinja2`      | No             | Only if `wants_html_report` is true |

Format as:
```
**Packages to install:**
# pip
pip install spidermon jsonschema [jinja2 if needed]
# uv
uv pip install spidermon jsonschema [jinja2 if needed]
```

**2. Files and folders to create** — list every file and folder the user will need,
with the exact paths relative to their project root. Use the actual `project_name`,
`spider_name`, and `item_class_name` from Step 0. Example:

```
**Files and folders you'll need:**
├── {project_name}/
│   ├── schemas/
│   │   └── {spider_name}_schema.json   ← item validation schema
│   ├── monitors.py                      ← monitor suite definitions
│   └── settings.py                      ← add Spidermon settings here (existing file)
```

If `wants_html_report` is true, also include:
```
│   └── reports/                         ← HTML reports will be saved here (create this folder)
```

**3. Quick summary sentence** — one sentence describing what the setup will achieve.
Example: "This will set up item validation, field coverage monitoring, and finish reason
checks for your `{spider_name}` spider, with an HTML report generated after each run."
Omit the HTML report part if `wants_html_report` is false.

### Presentation

Present this as a compact overview at the start of your response, BEFORE diving into
the actual file contents. Keep it scannable — the user should be able to glance at it,
run the install commands, create the folders, and then follow along as you present each
file. Do NOT repeat these install commands in the post-output section — reference them
here instead and skip the duplicate in post-output.

---

## How routing works

This skill serves five workflows. Detect which one the user needs based on their input,
then follow the corresponding workflow section. If the input is ambiguous, ask one
clarifying question. If a conversation spans multiple workflows (e.g., user generates a
schema, then asks for monitors), carry context forward naturally.

**IMPORTANT:** Always complete Step 0 (Interactive Elicitation) before entering any
workflow. All output must use the gathered project context — no placeholders like
`myproject` or `YourItem` should ever appear in final output.

### Detection rules

Read the user's message and match against these patterns, checked in order:

| # | User provides...                                               | Workflow to follow        |
|---|----------------------------------------------------------------|---------------------------|
| 1 | A sample item: JSON object, Python dict, Scrapy Item class,   | **Schema Generator**      |
|   | or plain English description of fields                         |                           |
| 2 | A spider description: what it scrapes, expected volumes,       | **Monitor Bootstrapper**  |
|   | which fields matter, preferred notification channel            |                           |
| 3 | Existing `settings.py` and/or `monitors.py` code asking for   | **Config Advisor**        |
|   | review, improvements, or gap analysis                          |                           |
| 4 | Plain English monitoring rules like "fail if fewer than 100    | **Expression Builder**    |
|   | items" or "alert if error rate exceeds 1%"                     |                           |
| 5 | Spidermon log output containing `[Spidermon]`, validation      | **Troubleshooter**        |
|   | stats, FAILED/PASSED lines, or error tracebacks                |                           |

**Ambiguous cases:**

- If the user says something general like "help me set up Spidermon," run Step 0 first,
  then ask what they have so far: sample items? An existing spider? Current config? This
  determines the workflow.
- If the user provides both a sample item AND a spider description, run Schema Generator
  first, then chain into Monitor Bootstrapper using the generated schema.
- If detection is still unclear after one question, default to Monitor Bootstrapper as it
  provides the broadest value.
- **Step 0 always runs first**, regardless of which workflow is detected.

---

## Workflow 1: Schema Generator

**Goal:** Take any representation of a scraped item and produce a complete, validated
JSON schema plus the `settings.py` configuration to enable Spidermon validation.

### Input handling

The user may provide items in several formats. Normalize before generating:

| Input format              | How to handle                                        |
|---------------------------|------------------------------------------------------|
| JSON object               | Infer types directly from values                     |
| Python dict               | Parse as JSON (handle single quotes, True/False/None)|
| Scrapy Item class         | Extract field names, infer types from field classes   |
| Plain English description | Ask for field names and expected types if ambiguous   |
| List of items             | Union all keys, infer types from all values           |

### Generation rules

For each field in the sample item:

1. **Infer the JSON schema type** from the value:
   - String → `{"type": "string"}`; add `"minLength": 1` if the value is non-empty
   - Number (int or float) → `{"type": "number"}`; add `"minimum": 0` if value is positive
   - Boolean → `{"type": "boolean"}`
   - Array → `{"type": "array", "items": {...}}` with inferred item type
   - Null → `{"type": ["string", "null"]}` (default to nullable string; ask user)
   - Nested object → `{"type": "object", "properties": {...}}` (recurse)

2. **Detect string-encoded numbers:** If a field's value is a string that looks like a
   number (e.g., `"419.99"`, `"$29.99"`, `"1,200"`), **always flag this to the user**.
   This is a very common issue in web scraping — prices and quantities often come through
   as strings. Explain that the schema will define the field as `type: number`, which
   means their spider needs to convert the string to a float/int before yielding the item.
   Provide the fix: `item["price"] = float(item["price"].replace(",", ""))`. If the
   string contains currency symbols like `$` or `£`, include those in the replace chain.

3. **Apply smart patterns** based on field name heuristics:
   - Field name contains "url", "link", "href" → add `"pattern": "^https?://"`
   - Field name contains "email" → add `"pattern": "^[^@]+@[^@]+\\.[^@]+$"`
   - Field name contains "price", "cost" and **value is a number** → ensure `"type": "number", "exclusiveMinimum": 0`
   - Field name contains "price", "cost" and **value is a string** → use `"type": ["string", "null"]` with pattern `"^\\d{1,3}(,\\d{3})*(\\.\\d+)?$|^\\d+(\\.\\d+)?$"`. This handles both plain decimals (`"664.00"`) and comma-formatted thousands (`"1,100.00"`). Web scrapers commonly pull from `og:price:amount` meta tags which return the display value with thousand separators before any stripping logic runs — the broader pattern prevents false validation failures on those items.
   - Field name contains "date" → add `"pattern": "^\\d{4}-\\d{2}-\\d{2}"`
   - Field name contains "image" → add URL pattern
   - Field name contains "rating", "score" → add reasonable min/max

4. **Mark required fields:** By default, mark all fields with non-null values as
   required. Ask the user if they want to adjust.

5. **Output three things:**
   - The JSON schema file content
   - The `settings.py` lines to add (MUST include `SPIDERMON_MAX_ITEM_VALIDATION_ERRORS = 0`
     — without it, `ItemValidationMonitor` crashes with `NotConfigured` at runtime)
   - A brief explanation of what each validation rule catches

6. **Use the gathered item class for field coverage:** Use the `item_class_name` from
   Step 0 to set the correct field coverage prefix. If `item_type` is `"dict"`, use
   `"dict/field"`. Otherwise use the class name: `"ProductItem/field"`. Never output
   placeholder prefixes — always use the actual class name gathered in Step 0.

Consult `references/output-templates.md` for the schema file template and
`references/spidermon-knowledge-base.md` for all available JSON schema keywords.

### Example interaction

> **User:** Here is a sample item: `{"title": "Blue Widget", "price": 9.99, "url":
> "https://shop.com/widget", "in_stock": true, "tags": ["sale", "new"]}`
>
> **Assistant:** *(generates schema with string/minLength for title, number/minimum:0
> for price, string/pattern for url, boolean for in_stock, array of strings for tags;
> all marked required; outputs settings.py config)*

---

## Workflow 2: Monitor Bootstrapper

**Goal:** Generate a complete, production-ready `monitors.py` and `settings.py` based
on the user's spider description.

### Information to gather

Most critical info (project name, spider name, item type) is already collected in Step 0.
If not already provided, ask the user for the remaining details:

| Field                 | Required | Default if not provided              | Source         |
|-----------------------|----------|--------------------------------------|----------------|
| What the spider scrapes| Yes     | —                                    | Ask user       |
| Expected item count   | Yes      | From Step 0 elicitation              | **Always from Step 0** |
| Critical fields       | No       | All fields from schema if available  | Ask or default |
| Notification channel  | No       | Log-only (no notification action)    | Ask or default |
| Error tolerance       | No       | 10 errors max                        | Ask or default |

**All generated code must use the actual `project_name`, `spider_name`, and
`item_class_name` from Step 0 — no placeholders.**

### Generation rules

1. **Always include these monitors:**
   - `ItemCountMonitor` (custom, with user's threshold from Step 0 — never default to 100)
   - `FinishReasonMonitor` (built-in)
   - `FieldCoverageMonitor` — include by default, not just conditionally

2. **For error count checking, prefer expression monitors over ErrorCountMonitor.**
   The built-in `ErrorCountMonitor` relies on the `log_count/ERROR` Scrapy stat, which
   does not exist when there are zero errors. This causes the monitor to SKIP rather
   than PASS, which confuses users. Instead, generate an expression monitor that uses
   `stats.get('log_count/ERROR', 0)` with a default of 0, which handles the missing
   stat gracefully:
   ```python
   SPIDERMON_SPIDER_CLOSE_EXPRESSION_MONITORS = [
       {"name": "ErrorChecks", "tests": [
           {"name": "error_count_within_threshold",
            "expression": "stats.get('log_count/ERROR', 0) < 10"},
       ]},
   ]
   ```
   **Important:** When generating expression monitors alongside a MonitorSuite, always
   test them separately first. Expression monitors with syntax errors or missing
   settings can crash the Spidermon extension entirely. Recommend the user add them
   after confirming the base suite works.

3. **Conditionally include:**
   - `DownloaderExceptionMonitor` — if user mentions anti-bot or network concerns
   - `PeriodicItemCountMonitor` — if user mentions long-running or large crawls
   - `ItemValidationMonitor` — if a JSON schema has been generated or is referenced.
     **IMPORTANT:** When including `ItemValidationMonitor`, ALWAYS add
     `SPIDERMON_MAX_ITEM_VALIDATION_ERRORS = 0` to the generated settings. Without this
     setting, `ItemValidationMonitor` will crash with `NotConfigured` at runtime. Default
     to `0` (zero tolerance) and add a comment telling the user to adjust if needed:
     ```python
     # Maximum number of validation errors before the monitor fails.
     # Set to 0 for zero tolerance. Increase if some errors are acceptable.
     SPIDERMON_MAX_ITEM_VALIDATION_ERRORS = 0
     ```

4. **Wire up notifications** based on user preference:
   - Slack → `SendSlackMessageSpiderFinished` + settings
   - Telegram → `SendTelegramMessageSpiderFinished` + settings
   - Discord → `SendDiscordMessageSpiderFinished` + settings
   - Email → `SendSESEmail` + settings
   - Sentry → `SendSentryMessage` + settings
   - None specified → log-only, but suggest Slack as a next step

5. **If `wants_html_report` is true**, add `CreateFileReport` to the monitor suite and
   generate the corresponding settings. This produces a styled HTML report after each run
   showing monitor results, spider stats, and validation errors.

   **In `monitors.py`:**
   - Add import: `from spidermon.contrib.actions.reports.files import CreateFileReport`
   - Add `monitors_finished_actions = [CreateFileReport]` to the `MonitorSuite` class.
     Use `monitors_finished_actions` (runs on every suite completion — pass or fail) as
     the default. Mention in a code comment that `monitors_failed_actions` is available
     if the user only wants reports on failures.

   **In `settings.py`:**
   - Add these three settings:
     ```python
     # --- Spidermon HTML Report ---
     # Uses Spidermon's built-in Jinja2 template (shows monitor results, stats, errors).
     SPIDERMON_REPORT_TEMPLATE = "reports/email/monitors/result.jinja"
     SPIDERMON_REPORT_CONTEXT = {
         "report_title": "Spidermon Report — {spider_name} spider"
     }
     SPIDERMON_REPORT_FILENAME = "reports/{spider_name}_report.html"
     ```
   - Replace `{spider_name}` with the actual spider name from Step 0.
   - The template path `reports/email/monitors/result.jinja` is a built-in Spidermon
     template — the user does NOT need to create it. It ships with the `spidermon` package.
   - The output path `reports/{spider_name}_report.html` is relative to the working
     directory where `scrapy crawl` is run.

   **In the prerequisites overview (Step 1):**
   - Include `jinja2` in the install command.
   - Include `reports/` in the folder listing.

   **In the post-output section:**
   - Tell the user to create the `reports/` folder: `mkdir -p reports`
   - Tell the user where to find the report after a run.
   - Provide the command to open the report:
     ```
     open reports/{spider_name}_report.html        # macOS
     xdg-open reports/{spider_name}_report.html    # Linux
     ```

6. **Output three files worth of content:**
   - `monitors.py` — complete, importable file
   - `settings.py` additions — all required settings with comments
   - A summary explaining what each monitor does and when it fires

Consult `references/output-templates.md` for the monitors.py and settings.py templates.

### Field coverage prefix: item class detection

Field coverage stat keys use the item type name as a prefix. This is the most common
source of FieldCoverageMonitor failures. The rules are:

| Item type | Prefix | Example |
|-----------|--------|---------|
| Plain Python `dict` | `dict/` | `"dict/name": 1.0` |
| Scrapy `Item` subclass | Class name | `"ProductItem/name": 1.0` |
| Python `dataclass` | Class name | `"ScanItem/name": 1.0` |
| `attrs` class | Class name | `"Product/name": 1.0` |

If the user provided an item class definition (e.g., in the Schema Generator step or in
their spider code), use the class name. If not, ask: "What is your item class name? If
you use plain dicts, the prefix is `dict/`. If you use a dataclass or Scrapy Item, it's
the class name (e.g., `ProductItem/`)."

**Never assume `dict/` as the default without asking.** Getting this wrong causes
FieldCoverageMonitor to fail even when the data is perfect.

### Chaining from Schema Generator

If Workflow 1 was run earlier in the conversation, automatically:
- Reference the generated schema path in `SPIDERMON_VALIDATION_SCHEMAS`
- Add `ItemValidationMonitor` to the suite
- Set `FieldCoverageMonitor` rules based on the schema's required fields (1.0 coverage)
  and optional fields (0.7 coverage as default)
- Use the item class name from Workflow 1 for coverage rule prefixes

---

## Workflow 3: Config Advisor

**Goal:** Review existing Spidermon configuration and suggest improvements.

### Input

The user pastes one or more of:
- `settings.py` (full or partial)
- `monitors.py`
- `schema.json`

### Review checklist

Check the configuration against this ordered list. For each gap found, explain what
it means and provide the fix:

1. **Is Spidermon enabled?**
   - Look for `SPIDERMON_ENABLED = True` and the extension in `EXTENSIONS`
   - If missing: this is critical, nothing works without it

2. **Is validation set up?**
   - Look for `ItemValidationPipeline` in `ITEM_PIPELINES`
   - Look for `SPIDERMON_VALIDATION_SCHEMAS` or `SPIDERMON_VALIDATION_MODELS`
   - If missing: recommend adding it as the highest-impact improvement

3. **Is field coverage tracked?**
   - Look for `SPIDERMON_ADD_FIELD_COVERAGE = True`
   - Look for `SPIDERMON_FIELD_COVERAGE_RULES`
   - If missing: recommend with sensible defaults based on any schema present

4. **Which monitors are configured?**
   - Check `SPIDERMON_SPIDER_CLOSE_MONITORS`
   - Compare against built-in monitors they are not using
   - Flag if `FinishReasonMonitor` is missing (catches silent failures)
   - Flag if `ErrorCountMonitor` is missing (catches runaway errors)

5. **Are notifications configured?**
   - Check for any action classes in monitor suites
   - If log-only: suggest adding at least one notification channel

6. **Expression monitor opportunities:**
   - If simple custom monitors could be replaced with expression monitors, suggest it
   - If no expression monitors exist, suggest quick wins

7. **Best practices:**
   - Validation pipeline should be the last pipeline (highest number)
   - `SPIDERMON_EXPECTED_FINISH_REASONS` should be set (default: `["finished"]`)
   - Consider `SPIDERMON_FIELD_COVERAGE_SKIP_IF_NO_ITEM = True` for optional spiders

### Output format

Structure the review as:

```
## Spidermon configuration review

### What is working well
[List what is already correctly configured]

### Recommended improvements
[Ordered by impact, each with: what to add, why it matters, the code to paste]

### Optional enhancements
[Lower-priority suggestions]
```

---

## Workflow 4: Expression Builder

**Goal:** Convert plain English monitoring rules into Spidermon expression monitor
configuration.

### Input

The user describes rules in natural language. Examples:
- "Fail if fewer than 100 items"
- "Alert if error rate is above 1%"
- "Make sure the spider finishes normally"
- "Check that at least 90% of items have a price"

### Translation rules

Map natural language to expression syntax using these patterns:

| User says...                                   | Expression                                                    |
|------------------------------------------------|---------------------------------------------------------------|
| "at least N items"                             | `stats.get('item_scraped_count', 0) >= N`                     |
| "fewer/less than N items"                      | `stats.get('item_scraped_count', 0) < N`                      |
| "fewer/less than N errors"                     | `stats.get('log_count/ERROR', 0) < N`                         |
| "error rate below X%"                          | `stats.get('log_count/ERROR', 0) < stats.get('item_scraped_count', 1) * X/100` |
| "finishes normally"                            | `stats.get('finish_reason') == 'finished'`                    |
| "no critical errors"                           | `stats.get('log_count/CRITICAL', 0) == 0`                     |
| "field X coverage above Y%"                    | `stats.get('spidermon_field_coverage/dict/X', 0) >= Y/100`    |
| "no downloader exceptions"                     | `stats.get('downloader/exception_count', 0) == 0`             |
| "runs in under N minutes"                      | `stats.get('elapsed_time_seconds', 0) < N * 60`               |

### Output

Generate the complete settings dictionary and explain each expression:

```python
SPIDERMON_SPIDER_CLOSE_EXPRESSION_MONITORS = [
    {
        "name": "CustomChecks",
        "tests": [
            {
                "name": "descriptive_test_name",
                "expression": "the expression here",
            },
            # ... more tests
        ],
    },
]
```

**Always remind the user** that expression monitors have access to three objects:
`stats` (spider stats dict), `spider` (the spider instance), and `job` (Scrapy Cloud
job info, if running on Scrapy Cloud).

---

## Workflow 5: Troubleshooter

**Goal:** Parse Spidermon log output, explain failures in plain language, and suggest
concrete fixes.

### Input

The user pastes Spidermon log output. This typically contains blocks like:

```
INFO: [Spidermon] -------------------- MONITORS --------------------
INFO: [Spidermon] Item count/Minimum number of items... FAIL
INFO: [Spidermon] 1 monitor in 0.001s
INFO: [Spidermon] FAILED (failures=1)
```

Or validation stats:
```
'spidermon/validation/fields/errors': 150,
'spidermon/validation/items/errors': 45,
```

### Analysis steps

1. **Parse the log** for:
   - Which monitors passed and which failed
   - Failure messages and tracebacks
   - Validation stats (fields/errors, items/errors breakdowns)
   - Spider stats (item_scraped_count, finish_reason, error counts)

2. **Explain each failure** in plain language:
   - What the monitor was checking
   - What value it found vs what it expected
   - Why this matters for data quality

3. **Suggest fixes**, categorized as:
   - **Spider code fixes** (e.g., "your spider is yielding items with missing URLs,
     check the CSS selector for the URL field")
   - **Monitor config fixes** (e.g., "your threshold is set to 1000 but this spider
     typically returns 500 items — adjust SPIDERMON_MIN_ITEMS")
   - **Schema fixes** (e.g., "field 'price' is failing because some items have the
     price as a string like '$9.99' — either clean it in the spider or change the
     schema type to string with a pattern")

4. **If validation errors dominate**, break them down by field and suggest which
   fields to fix first (highest error count = highest priority).

### Output format

```
## Spidermon failure analysis

### Summary
[One sentence: X monitors ran, Y passed, Z failed]

### Failures
[For each failed monitor: what it checked, what went wrong, how to fix it]

### Validation breakdown (if applicable)
[Per-field error counts and suggested fixes]

### Recommended next steps
[Ordered action items]
```

---

## Cross-workflow behavior

**Context chaining:** When workflows run sequentially in a conversation, carry forward
all generated artifacts AND the Step 0 context. For example:

1. User provides sample item → Schema Generator creates `schema.json`
2. User asks for monitors → Monitor Bootstrapper references the schema, sets field
   coverage rules based on its required/optional fields
3. User pastes logs after running → Troubleshooter uses knowledge of the schema and
   monitors to give more precise suggestions

**Zero-placeholder output rule:** Every piece of generated code MUST use the real values
from Step 0 elicitation. Specifically:
- `project_name` replaces all instances of `myproject`
- `spider_name` replaces generic spider references
- `item_class_name` replaces `dict` in field coverage rules (unless items ARE plain dicts)
- Monitor suite paths use the real project name: `"{project_name}.monitors.SpiderCloseMonitorSuite"`
- Schema paths use the real spider/item name: `"./schemas/{spider_name}_schema.json"`
- If using scrapy-poet, item imports reference the actual module and class names

The user should be able to copy-paste every code block directly into their project.

**Always be concrete:** Every suggestion should include copy-pasteable code. Never
say "you should add a monitor for X" without providing the actual code.

**Spidermon version awareness:** The current version is 1.25.0. Schematics-based
validation is deprecated — always recommend JSON Schema. If a user's code uses
schematics, suggest migrating to JSON Schema and offer to convert their model.

**Scrapy version compatibility:** Scrapy 2.13+ deprecated the `spider` argument from
pipeline `process_item()` calls. Spidermon ≤1.25.0's `ItemValidationPipeline` still
expects this argument, causing a `TypeError: process_item() missing 1 required
positional argument: '_'`. This is a confirmed bug tracked at:
**https://github.com/scrapinghub/spidermon/issues/473**

If the user's `scrapy_version` from Step 0 is ≥2.13, **proactively warn them** before
generating any code. Always share the issue link so they can track the fix. Present the
following two options:

**Option A (recommended): Downgrade Scrapy below 2.13**

This is the safest approach until Spidermon ships a fix. Walk the user through these
steps:

```bash
# Step 1: Check your current Scrapy version
pip show scrapy   # or: uv pip show scrapy

# Step 2: Downgrade Scrapy to the latest 2.12.x release
# pip
pip install "scrapy>=2.12,<2.13"
# uv
uv add "scrapy>=2.12,<2.13"

# Step 3: Verify the downgrade
pip show scrapy   # Should show 2.12.x
```

If the user uses a `requirements.txt`, remind them to pin it there too:
```
scrapy>=2.12,<2.13
```

If they use `pyproject.toml`:
```toml
dependencies = [
    "scrapy>=2.12,<2.13",
]
```

**Option B: Patched pipeline (if user cannot downgrade Scrapy)**

Some users may need to stay on Scrapy 2.13+ (e.g., they depend on new Scrapy features).
In that case, instruct them to add this to their `pipelines.py` and use it in
`ITEM_PIPELINES` instead of the original Spidermon pipeline:

```python
from spidermon.contrib.scrapy.pipelines import ItemValidationPipeline

class PatchedItemValidationPipeline(ItemValidationPipeline):
    """Wrapper that accepts process_item with or without the spider arg,
    fixing the Scrapy 2.13+ / Spidermon <=1.25.0 incompatibility.
    Track the fix: https://github.com/scrapinghub/spidermon/issues/473"""

    def process_item(self, item, _=None):
        return super().process_item(item, _)
```

Then in settings: `"<project>.pipelines.PatchedItemValidationPipeline": 900`

**IMPORTANT:** The patched pipeline MUST use `_=None` as the default (not `*args`).
Using `*args` does NOT work because `super().process_item(item)` still requires the
second positional argument — it just won't receive it from the empty `*args`.

**When presenting the warning**, use this format:

```
⚠️ **Scrapy 2.13+ Compatibility Issue**

Your Scrapy version ({scrapy_version}) is affected by a known bug in Spidermon ≤1.25.0
where `ItemValidationPipeline` crashes with:

    TypeError: ItemValidationPipeline.process_item() missing 1 required positional argument: '_'

This is tracked at: https://github.com/scrapinghub/spidermon/issues/473

**Recommended fix:** Downgrade Scrapy to 2.12.x:
    pip install "scrapy>=2.12,<2.13"

If you can't downgrade, I'll include a patched pipeline workaround in the generated code.
```

This applies to ANY workflow that uses `ItemValidationPipeline` (Schema Generator,
Monitor Bootstrapper). Check this BEFORE generating files so the user doesn't hit the
error after setting everything up.

**SPIDERMON_VALIDATION_SCHEMAS key format:** When using `SPIDERMON_VALIDATION_SCHEMAS`
as a dict (for multiple item types or typed items), the keys MUST be the actual imported
class objects, NOT string paths. Using a string like `"myproject.items.MyItem"` causes
`AttributeError: 'str' object has no attribute '__name__'`. Always generate the schema
config with an explicit import:
```python
from myproject.items import MyItem
SPIDERMON_VALIDATION_SCHEMAS = {
    MyItem: "./schemas/my_schema.json",
}
```
Never use a string key like `"myproject.items.MyItem": "./schemas/my_schema.json"`.
This applies to ALL item types (dataclass, attrs, Scrapy Item). Plain dict items use
a list format instead: `SPIDERMON_VALIDATION_SCHEMAS = ["./schemas/my_schema.json"]`.

**Spidermon extension class name:** In Spidermon 1.25.0 the Scrapy extension is
registered as `Spidermon`, **not** `SpiderMonitor`. Using the wrong name causes a
`NameError` at startup before the spider even runs:

    NameError: Module 'spidermon.contrib.scrapy.extensions' doesn't define any object named 'SpiderMonitor'

Always generate:
```python
EXTENSIONS = {
    "spidermon.contrib.scrapy.extensions.Spidermon": 500,
}
```

---

## Post-output: Installation and next steps (ALWAYS INCLUDE)

After generating all files, ALWAYS append the following sections to your response. These
help users who may not be deeply familiar with Spidermon understand what just happened
and how to verify it works.

### 1. Install Spidermon

If you already presented the install commands in the Step 1 Prerequisites Overview
(which you should have), do NOT repeat them here. Instead, write a brief reminder:

```
"Make sure you've installed the packages listed at the top of this response."
```

If for some reason Step 1 was skipped (e.g., you're in Workflow 3 or 5 and didn't
generate new files), provide the install commands here:

```
**Install Spidermon:**
# pip
pip install spidermon jsonschema
# uv
uv pip install spidermon jsonschema
```

If `wants_html_report` is true, append `jinja2` to both commands.

### 2. Plain English explanation (under 300 words)

After presenting the generated files, include a brief explanation of what each file does
and what the overall setup achieves. Write in plain English — minimal developer jargon.
Keep it under 300 words. Example tone:

> "Here's what each piece does: The **schema file** describes what a valid item looks
> like — which fields are required, what type each field should be, and any patterns
> (like URLs must start with http). The **monitors file** contains checks that run
> automatically every time your spider finishes — it verifies you scraped enough items,
> the spider didn't crash, and your fields are consistently populated. The **settings**
> wire everything together — they tell Scrapy to load Spidermon, validate every item
> against your schema, and run your monitors at the end of each crawl."

If `wants_html_report` is true, append:

> "The **HTML report** is generated automatically after each run. It's a styled page
> showing which monitors passed or failed, spider stats, and any validation errors.
> You'll find it at `reports/{spider_name}_report.html` (relative to where you ran
> `scrapy crawl`). The template is built into Spidermon — you don't need to create it."

Adjust based on what was actually generated.

### 3. Run command with log file

Always provide the command to run the spider with logs saved to a file.

If `wants_html_report` is true, prepend a `mkdir -p reports` command so the output
folder exists:

```
**Create the reports folder (first time only):**
mkdir -p reports

**Run your spider and save the logs:**
scrapy crawl {spider_name} -O {output_file}.jsonl 2>&1 | tee {spider_name}_crawl.log
```

If `wants_html_report` is true, also provide the command to open the report:

```
**Open the HTML report:**
open reports/{spider_name}_report.html        # macOS
xdg-open reports/{spider_name}_report.html    # Linux
```

### 4. Verify Spidermon results

Provide these three commands to check the Spidermon output, along with an explanation
of what each shows and an example of what healthy output looks like. Use the actual
`{spider_name}` from Step 0 in all commands.

**Command 1 — Monitor scorecard (pass/fail at a glance):**
```bash
grep "\[Spidermon\]" {spider_name}_crawl.log | grep -E "(OK|FAIL|SKIP)"
```
Expected output when everything is healthy:
```
ItemCountMonitor/Minimum number of items... OK
Finish Reason Monitor/Should have the expected finished reason(s)... OK
Field Coverage Monitor/test_check_if_field_coverage_rules_are_met... OK
Item Validation Monitor/No item validation errors... OK
```
Any line showing `FAIL` means that monitor's check did not pass. `SKIP` means the
monitor couldn't run (usually a missing setting or no data).

**Command 2 — Catch errors and misconfigurations:**
```bash
grep -E "\[Spidermon\]|NotConfigured|spidermon.*ERROR" {spider_name}_crawl.log
```
If everything is configured correctly, this should show only the Spidermon monitor
lines (same as above). If you see `NotConfigured` or `ERROR` lines, a setting is
missing or a monitor crashed — share the output for debugging.

**Command 3 — Validation stats breakdown (schema errors per field):**
```bash
grep "spidermon/validation" {spider_name}_crawl.log
```
Expected output when all items pass validation:
```
'spidermon/validation/fields': 30,
'spidermon/validation/items': 10,
```
If there are errors, you'll also see lines like:
```
'spidermon/validation/fields/errors': 5,
'spidermon/validation/fields/errors/price': 5,
'spidermon/validation/items/errors': 5,
```
The field-level breakdown (e.g. `errors/price`) tells you exactly which field is
failing — fix the highest-count fields first.

### 5. Debug offer

Always end with an invitation to share logs for debugging:

```
"If anything looks off in the Spidermon output, feel free to share the relevant log
lines and I'll help you debug."
```

This closing line ensures the user knows they can come back for Workflow 5
(Troubleshooter) without having to figure out what to provide.
