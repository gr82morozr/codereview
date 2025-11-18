"""

~~~

#!/usr/bin/env python3
"""Generate an HTML report for Siebel validation messages based on keyword hits."""

import argparse
import csv
import html
import json
import os
import pathlib
import re
import subprocess
import sys
import tempfile
from dataclasses import dataclass
from datetime import datetime
from typing import Dict, List, Sequence


def load_keywords(path):
    """Read keywords from a text file, skipping blanks and comments."""
    path = pathlib.Path(path)
    keywords = []
    for line in path.read_text(encoding="utf-8").splitlines():
        value = line.strip()
        if not value or value.startswith("#"):
            continue
        keywords.append(value)
    return keywords


def load_config(path):
    """Load the json config that stores DB credentials and sqlplus path."""
    path = pathlib.Path(path)
    with path.open("r", encoding="utf-8") as handle:
        data = json.load(handle)
    for key in ("username", "password", "dsn"):
        if not data.get(key):
            raise ValueError(f"Config file is missing required '{key}' value.")
    return data


def sql_literal(value):
    """Escape a Python string so it can be safely embedded in SQL."""
    value = "" if value is None else str(value)
    return "'" + value.replace("'", "''") + "'"


class SQLTemplateRenderer:
    """Utility for loading SQL template files and injecting runtime values."""

    def __init__(self, templates_dir):
        self.templates_dir = pathlib.Path(templates_dir)
        self._cache = {}

    def render(self, template_name, **params):
        template = self._get_template(template_name)
        rendered = template
        for key, value in params.items():
            rendered = rendered.replace(f"{{{{{key}}}}}", value)
        return rendered

    def _get_template(self, template_name):
        if template_name not in self._cache:
            template_path = self.templates_dir / template_name
            if not template_path.exists():
                msg = f"Template '{template_name}' not found in {self.templates_dir}."
                raise FileNotFoundError(msg)
            self._cache[template_name] = template_path.read_text(encoding="utf-8")
        return self._cache[template_name]


class SQLPlusClient:
    """Small helper around sqlplus."""

    def __init__(self, username, password, dsn, sqlplus_path="sqlplus", nls_lang=None):
        self.username = username
        self.password = password
        self.dsn = dsn
        self.sqlplus_path = sqlplus_path
        self.nls_lang = nls_lang

    def run_query(self, sql_text):
        """Execute SQL via sqlplus and parse CSV output into dictionaries."""
        script_text = "WHENEVER SQLERROR EXIT SQL.SQLCODE;\n" + sql_text.strip() + "\n"
        with tempfile.NamedTemporaryFile("w", suffix=".sql", delete=False, encoding="utf-8") as tmp:
            tmp.write(script_text)
            tmp_path = pathlib.Path(tmp.name)
        env = os.environ.copy()
        if self.nls_lang:
            env["NLS_LANG"] = self.nls_lang
        connect_str = f"{self.username}/{self.password}@{self.dsn}"
        cmd = [self.sqlplus_path, "-S", connect_str, f"@{tmp_path}"]
        try:
            completed = subprocess.run(
                cmd,
                capture_output=True,
                text=True,
                check=False,
                env=env,
            )
        except FileNotFoundError as exc:
            tmp_path.unlink(missing_ok=True)
            raise RuntimeError(f"Unable to execute sqlplus command '{self.sqlplus_path}'.") from exc
        finally:
            tmp_path.unlink(missing_ok=True)
        if completed.returncode != 0:
            raise RuntimeError(f"sqlplus returned {completed.returncode}: {completed.stderr.strip()}")
        stdout = completed.stdout.strip()
        if "SP2-" in stdout or "ORA-" in stdout:
            raise RuntimeError(f"sqlplus error:\n{stdout}")
        return parse_csv_output(stdout)


@dataclass
class ReportSection:
    """Bundle the three datasets rendered for a single keyword."""

    keyword: str
    messages: List[Dict[str, str]]
    msg_type_rows: List[Dict[str, str]]
    field_map_rows: List[Dict[str, str]]
    msg_types: List[str]


def parse_csv_output(raw_output):
    """Parse sqlplus CSV output."""
    lines = []
    for line in raw_output.splitlines():
        stripped = line.strip()
        if not stripped or stripped.lower().startswith("disconnected from"):
            continue
        if stripped.lower() == "no rows selected":
            return []
        lines.append(line)
    if not lines:
        return []
    reader = csv.reader(lines)
    rows = list(reader)
    if not rows:
        return []
    headers = rows[0]
    data_rows = rows[1:]
    parsed = []
    for row in data_rows:
        if len(row) < len(headers):
            row.extend([""] * (len(headers) - len(row)))
        parsed.append(dict(zip(headers, row)))
    return parsed


def build_highlight_specs(keyword, msg_types):
    """Precompile keyword/msg-type regex specs for HTML highlighting."""
    specs = []
    if keyword:
        specs.append((re.compile(re.escape(keyword), re.IGNORECASE), "kw"))
    if not isinstance(msg_types, Sequence):
        msg_types = []
    msg_terms = [term for term in msg_types if term]
    if msg_terms:
        msg_pattern = re.compile("|".join(re.escape(term) for term in msg_terms), re.IGNORECASE)
        specs.append((msg_pattern, "msg-type"))
    return specs


def highlight_text(value, highlight_specs):
    """Highlight keyword and msg type occurrences inside a single table cell."""
    text = "" if value is None else str(value)
    if not highlight_specs:
        return html.escape(text)
    segments = [(text, None)]
    for pattern, css_class in highlight_specs:
        new_segments = []
        for segment_text, active_class in segments:
            if active_class is not None:
                new_segments.append((segment_text, active_class))
                continue
            last = 0
            for match in pattern.finditer(segment_text):
                if match.start() > last:
                    new_segments.append((segment_text[last:match.start()], None))
                new_segments.append((match.group(0), css_class))
                last = match.end()
            if last < len(segment_text):
                new_segments.append((segment_text[last:], None))
        segments = new_segments
    parts = []
    for segment, css_class in segments:
        escaped = html.escape(segment)
        if css_class:
            parts.append(f'<mark class="{css_class}">{escaped}</mark>')
        else:
            parts.append(escaped)
    return "".join(parts)


def render_table(rows, highlight_specs):
    """Render an HTML table (or placeholder message) for a query result."""
    if not rows:
        return '<p class="empty">No rows returned.</p>'
    headers = list(rows[0].keys())
    body_rows = []
    for row in rows:
        cells = "".join(
            f"<td>{highlight_text(row.get(header), highlight_specs)}</td>"
            for header in headers
        )
        body_rows.append(f"<tr>{cells}</tr>")
    header_cells = "".join(f"<th>{html.escape(header)}</th>" for header in headers)
    table = [
        "<table>",
        f"<thead><tr>{header_cells}</tr></thead>",
        "<tbody>",
        "\n".join(body_rows),
        "</tbody>",
        "</table>",
    ]
    return "\n".join(table)


def build_report(sections):
    """Construct the final HTML document."""
    generated_at = datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S UTC")
    section_html = []
    for section in sections:
        keyword_value = section.keyword
        keyword = keyword_value if isinstance(keyword_value, str) else str(keyword_value)
        keyword_display = f'<mark class="kw">{html.escape(keyword)}</mark>'
        msg_types = [value for value in section.msg_types if isinstance(value, str)]
        highlight_specs = build_highlight_specs(keyword, msg_types)
        section_html.append(
            "\n".join(
                [
                    "<section>",
                    f"<h2>Keyword: {keyword_display}</h2>",
                    '<div class="panel">',
                    "<h3>S_ISS_VALDN_MSG matches</h3>",
                    render_table(section.messages, highlight_specs),
                    "</div>",
                    '<div class="panel">',
                    "<h3>MSG_TYPE details</h3>",
                    render_table(section.msg_type_rows, highlight_specs),
                    "</div>",
                    '<div class="panel">',
                    "<h3>S_INT_FLDMAP matches</h3>",
                    render_table(section.field_map_rows, highlight_specs),
                    "</div>",
                    "</section>",
                ]
            )
        )
    style_block = """
body { font-family: Segoe UI, Arial, sans-serif; margin: 1.5rem; background: #f8f9fb; color: #1f1f1f; }
h1 { font-size: 1.75rem; margin-bottom: 0.25rem; }
h2 { margin-top: 2rem; border-bottom: 2px solid #d0d7de; padding-bottom: 0.25rem; }
.panel { margin-top: 0.75rem; background: #fff; padding: 0.75rem; border-radius: 8px; box-shadow: 0 1px 3px rgba(15,23,42,0.1); }
table { border-collapse: collapse; width: 100%; margin-top: 0.5rem; font-size: 0.9rem; }
th, td { border: 1px solid #d0d7de; padding: 0.35rem 0.5rem; text-align: left; vertical-align: top; }
th { background: #f0f4f8; }
mark.kw { background: #fff59d; }
mark.msg-type { background: #c8e6c9; }
.empty { font-style: italic; color: #555; }
section { margin-bottom: 2.5rem; }
"""
    html_doc = f"""<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <title>Siebel Validation Message Report</title>
    <style>{style_block}</style>
</head>
<body>
    <h1>Siebel Validation Message Report</h1>
    <p>Generated at {generated_at} for {len(sections)} keyword(s).</p>
    {"".join(section_html) if section_html else '<p>No keywords produced any results.</p>'}
</body>
</html>
"""
    return html_doc


def fetch_msg_type_rows(msg_types, renderer, client, cache):
    """Retrieve MSG_TYPE rows while caching results across keywords."""
    rows = []
    for msg_type in msg_types:
        cached = cache.get(msg_type)
        if cached is None:
            query = renderer.render("message_type_details.sql", msg_type_cd=sql_literal(msg_type))
            cached = client.run_query(query)
            cache[msg_type] = cached
        rows.extend(cached)
    return rows


def collect_results_for_keyword(keyword, renderer, client, msg_type_cache):
    like_literal = sql_literal(f"%{keyword}%")
    messages_query = renderer.render("validation_messages.sql", like_keyword=like_literal)
    messages = client.run_query(messages_query)
    msg_types = sorted(
        {
            candidate
            for row in messages
            for candidate in [str(row.get("MSG_TYPE_CD", "")).strip()]
            if candidate
        }
    )

    msg_type_rows = fetch_msg_type_rows(msg_types, renderer, client, msg_type_cache)

    if msg_types:
        type_filter = ", ".join(sql_literal(value) for value in msg_types)
        msg_type_condition = f"MSG_TYPE_CD IN ({type_filter})"
    else:
        msg_type_condition = "0 = 1"
    field_map_query = renderer.render(
        "field_map.sql",
        like_keyword=like_literal,
        msg_type_condition=msg_type_condition,
    )
    field_map_rows = client.run_query(field_map_query)
    return ReportSection(
        keyword=keyword,
        messages=messages,
        msg_type_rows=msg_type_rows,
        field_map_rows=field_map_rows,
        msg_types=msg_types,
    )


def parse_args(argv):
    parser = argparse.ArgumentParser(description="Generate Siebel error code report.")
    parser.add_argument("--keywords", default="keywords.txt", help="Path to keywords text file.")
    parser.add_argument("--config", default="config.json", help="Path to DB config json.")
    parser.add_argument("--templates", default="templates", help="Directory containing SQL templates.")
    parser.add_argument("--output", default="report.html", help="HTML file to write.")
    return parser.parse_args(argv)


def main(argv=None):
    args = parse_args(argv or sys.argv[1:])
    base_path = pathlib.Path.cwd()
    keywords_path = (base_path / args.keywords).resolve()
    templates_path = (base_path / args.templates).resolve()
    config_path = (base_path / args.config).resolve()
    output_path = (base_path / args.output).resolve()

    keywords = load_keywords(keywords_path)
    if not keywords:
        raise SystemExit(f"No keywords were found in '{keywords_path}'.")
    config = load_config(config_path)
    renderer = SQLTemplateRenderer(templates_path)
    client = SQLPlusClient(
        username=config["username"],
        password=config["password"],
        dsn=config["dsn"],
        sqlplus_path=config.get("sqlplus_path", "sqlplus"),
        nls_lang=config.get("nls_lang"),
    )

    sections = []
    msg_type_cache: Dict[str, List[Dict[str, str]]] = {}
    for keyword in keywords:
        print(f"Processing keyword '{keyword}'...")
        sections.append(collect_results_for_keyword(keyword, renderer, client, msg_type_cache))

    html_report = build_report(sections)
    output_path.write_text(html_report, encoding="utf-8")
    print(f"Report written to {output_path}")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())




~~~

"""
