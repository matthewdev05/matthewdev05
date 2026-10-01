"""
Builds dark_mode.svg and light_mode.svg: ASCII portrait on the left,
neofetch-style info + live GitHub stats on the right.

Usage:
    GH_TOKEN=xxx python build_profile.py      # real stats
    python build_profile.py --offline         # preview with fake stats
"""
import json
import os
import sys
import urllib.request
from datetime import date, datetime, timedelta
from xml.sax.saxutils import escape

# ============ EDIT THIS SECTION ============
USERNAME = "matthewdev05"
BIRTHDAY = None  # e.g. date(2005, 1, 31) — then add ("kv", "Uptime", "{uptime}") below

# ("header", title) draws a section line, ("kv", key, value) an info row, ("blank",) a gap.
# {uptime}, {repos}, {contributed}, {stars}, {commits}, {followers} are filled in live.
INFO = [
    ("header", "matthew@fiakpornu"),
    ("kv", "Host", "Colgate University"),
    ("kv", "Kernel", "Computer Science, Physics Minor"),
    ("kv", "Focus", "Backend, Systems, Developer Tools"),
    ("kv", "IDE", "VS Code"),
    ("blank",),
    ("kv", "Languages.Programming", "Python, Java"),
    ("kv", "Languages.Backend", "Flask, REST APIs, HTTP & JSON"),
    ("kv", "Languages.Database", "SQLite, SQL"),
    ("kv", "Tools", "Git, GitHub, Unix/Linux, Postman"),
    ("blank",),
    ("kv", "Exploring.Systems", "Systems Design, Low-Level"),
    ("kv", "Exploring.Core", "Networking, Memory Management"),
    ("blank",),
    ("header", "Contact"),
    ("kv", "Email", "mfiakpornu@colgate.edu"),
    ("kv", "LinkedIn", "matthew-fiakpornu"),
    ("kv", "Location", "Hamilton, NY"),
    ("blank",),
    ("header", "GitHub Stats"),
    ("kv", "Repos", "{repos} (Contributed: {contributed})"),
    ("kv", "Stars", "{stars}"),
    ("kv", "Commits", "{commits}"),
    ("kv", "Followers", "{followers}"),
]
# ===========================================

LINE_CHARS = 58          # width of each info row in characters
FONT_SIZE = 16
LINE_H = 20
CHAR_W = FONT_SIZE * 0.6  # monospace character width
ASCII_X = 15

THEMES = {
    "dark": dict(bg="#161b22", text="#c9d1d9", key="#ffa657", value="#a5d6ff", dots="#616e7f"),
    "light": dict(bg="#f6f8fa", text="#24292f", key="#953800", value="#0a3069", dots="#c2cfde"),
}


# ---------- GitHub API ----------
def gql(query, variables, token):
    req = urllib.request.Request(
        "https://api.github.com/graphql",
        data=json.dumps({"query": query, "variables": variables}).encode(),
        headers={"Authorization": f"bearer {token}", "Content-Type": "application/json"},
    )
    with urllib.request.urlopen(req) as r:
        data = json.load(r)
    if "errors" in data:
        sys.exit(f"GitHub API error: {data['errors']}")
    return data["data"]


def fetch_stats(token):
    stars, repos, cursor = 0, 0, None
    while True:
        d = gql("""
        query($login: String!, $cursor: String) {
          user(login: $login) {
            followers { totalCount }
            repositoriesContributedTo(contributionTypes: [COMMIT, PULL_REQUEST, REPOSITORY]) { totalCount }
            contributionsCollection { contributionYears }
            repositories(ownerAffiliations: OWNER, first: 100, after: $cursor) {
              totalCount
              pageInfo { hasNextPage endCursor }
              nodes { stargazerCount }
            }
          }
        }""", {"login": USERNAME, "cursor": cursor}, token)["user"]
        repos = d["repositories"]["totalCount"]
        stars += sum(n["stargazerCount"] for n in d["repositories"]["nodes"])
        if not d["repositories"]["pageInfo"]["hasNextPage"]:
            break
        cursor = d["repositories"]["pageInfo"]["endCursor"]

    commits = 0
    for year in d["contributionsCollection"]["contributionYears"]:
        c = gql("""
        query($login: String!, $from: DateTime!, $to: DateTime!) {
          user(login: $login) {
            contributionsCollection(from: $from, to: $to) { totalCommitContributions }
          }
        }""", {"login": USERNAME, "from": f"{year}-01-01T00:00:00Z",
               "to": f"{year}-12-31T23:59:59Z"}, token)
        commits += c["user"]["contributionsCollection"]["totalCommitContributions"]

    return dict(
        repos=f"{repos:,}",
        contributed=f"{d['repositoriesContributedTo']['totalCount']:,}",
        stars=f"{stars:,}",
        commits=f"{commits:,}",
        followers=f"{d['followers']['totalCount']:,}",
    )


def uptime(born):
    t = date.today()
    y, m, d = t.year - born.year, t.month - born.month, t.day - born.day
    if d < 0:
        m -= 1
        d += (t.replace(day=1) - timedelta(days=1)).day
    if m < 0:
        y -= 1
        m += 12
    s = lambda n, w: f"{n} {w}{'' if n == 1 else 's'}"
    return f"{s(y, 'year')}, {s(m, 'month')}, {s(d, 'day')}"


# ---------- SVG drawing ----------
def render_rows(stats):
    """Returns a list of rows; each row is a list of (text, color_key) pieces."""
    rows = []
    for item in INFO:
        if item[0] == "blank":
            rows.append([])
        elif item[0] == "header":
            title = item[1]
            prefix = "" if not rows else "- "
            fill = "—" * max(3, LINE_CHARS - len(prefix + title) - 1)
            rows.append([(prefix + title + " ", "text"), (fill, "dots")])
        else:
            key, value = item[1], item[2].format(**stats)
            dots = "." * max(2, LINE_CHARS - len(key) - len(value) - 4)
            rows.append([(". ", "dots"), (key, "key"), (": ", "text"),
                         (dots + " ", "dots"), (value, "value")])
    return rows


def build_svg(theme, ascii_lines, rows):
    c = THEMES[theme]
    ascii_w = max((len(l) for l in ascii_lines), default=0)
    info_x = ASCII_X + ascii_w * CHAR_W + 25
    width = int(info_x + LINE_CHARS * CHAR_W + 20)
    n = max(len(ascii_lines), len(rows))
    height = 30 + n * LINE_H

    out = [
        f'<svg xmlns="http://www.w3.org/2000/svg" width="{width}" height="{height}" '
        f'font-family="ConsolasFallback,Consolas,\'Courier New\',monospace" font-size="{FONT_SIZE}px">',
        f'<rect width="{width}" height="{height}" fill="{c["bg"]}" rx="15"/>',
        f'<text x="{ASCII_X}" y="30" fill="{c["text"]}" xml:space="preserve">',
    ]
    for i, line in enumerate(ascii_lines):
        out.append(f'<tspan x="{ASCII_X}" y="{30 + i * LINE_H}">{escape(line)}</tspan>')
    out.append("</text>")

    out.append(f'<text x="{info_x:.0f}" y="30" fill="{c["text"]}" xml:space="preserve">')
    for i, row in enumerate(rows):
        if not row:
            continue
        spans = "".join(f'<tspan fill="{c[col]}">{escape(t)}</tspan>' for t, col in row)
        out.append(f'<tspan x="{info_x:.0f}" y="{30 + i * LINE_H}">{spans}</tspan>')
    out.append("</text></svg>")
    return "\n".join(out)


if __name__ == "__main__":
    if "--offline" in sys.argv:
        stats = dict(repos="12", contributed="20", stars="34", commits="567", followers="89")
    else:
        token = os.environ.get("GH_TOKEN")
        if not token:
            sys.exit("Set GH_TOKEN (or run with --offline to preview).")
        stats = fetch_stats(token)
    stats["uptime"] = uptime(BIRTHDAY) if BIRTHDAY else ""

    try:
        with open("ascii.txt", encoding="utf-8") as f:
            ascii_lines = f.read().splitlines()
    except FileNotFoundError:
        ascii_lines = ["(run make_ascii.py first)"]

    rows = render_rows(stats)
    for theme in THEMES:
        with open(f"{theme}_mode.svg", "w", encoding="utf-8") as f:
            f.write(build_svg(theme, ascii_lines, rows))
    print(f"Updated dark_mode.svg and light_mode.svg at {datetime.now():%Y-%m-%d %H:%M}")
