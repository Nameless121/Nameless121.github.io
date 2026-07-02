# Eliran's House of Teaching — Project Brief for Claude Code

> Read this fully before touching any file. Everything Claude needs to continue this project is here.

---

## Who is the user?

**Mr. Eliran** — ESL & Maths private teacher/tutor working with Hebrew-speaking students (primarily ages 7–16). Not a developer. All file deployments go through Claude using the GitHub REST API via Python scripts or browser fetch. The user has no terminal access on their end.

- GitHub username: `Nameless121`
- Site URL: `https://nameless121.github.io`
- Email: admin@clardyventures.com

---

## What this project is

A **static GitHub Pages site** called "Eliran's House of Teaching" — the student-facing tool layer for Mr. Eliran's broader teaching platform (called **Mind** / Bluebook Teacher). Students visit the site to use study tools; Mr. Eliran uses the Teacher Hub to generate and publish lesson content.

**No backend. No build step.** Everything is plain HTML/CSS/JS. All GitHub file operations use the GitHub REST API (authenticated with a Personal Access Token).

---

## Repository

`Nameless121/Nameless121.github.io` (public GitHub Pages repo)

All files at the root map to `https://nameless121.github.io/[path]`.

### File structure
```
/                           ← root
├── CLAUDE.md               ← this file (project brief)
├── index.html              ← homepage (student-facing)
├── assets/
│   └── style.css           ← shared CSS (may be sparse; each page is mostly self-contained)
├── vocabulary/
│   └── index.html          ← Quizlet-style vocab practice tool (A1 + A2)
├── teacher-hub/
│   └── index.html          ← password-locked teacher dashboard
└── little-prince/          ← exists on GitHub, not cloned locally
    └── index.html          ← Little Prince study guide (chapters 1–12)
```

Also exists on GitHub: `Clardy_Caller_Academy.html` (a standalone page, ignore).

---

## GitHub API — how to push files

Since the user has no terminal, Claude pushes files using Python + GitHub REST API. Pattern:

```python
import urllib.request, json, base64, pathlib

TOKEN = "<current session token>"   # user generates a new Classic token each session
REPO  = "Nameless121/Nameless121.github.io"
BASE  = f"https://api.github.com/repos/{REPO}/contents"
HEADERS = {
    "Authorization": f"token {TOKEN}",
    "Accept": "application/vnd.github.v3+json",
    "Content-Type": "application/json",
    "User-Agent": "EliranTeacher/1.0"
}

def get_sha(path):
    req = urllib.request.Request(f"{BASE}/{path}", headers=HEADERS)
    try:
        with urllib.request.urlopen(req) as r:
            return json.loads(r.read())["sha"]
    except:
        return None

def upload(path, local_path, message):
    content = pathlib.Path(local_path).read_bytes()
    b64     = base64.b64encode(content).decode()
    sha     = get_sha(path)   # needed for updates; None for new files
    body    = {"message": message, "content": b64}
    if sha:
        body["sha"] = sha
    data = json.dumps(body).encode()
    req  = urllib.request.Request(f"{BASE}/{path}", data=data, headers=HEADERS, method="PUT")
    with urllib.request.urlopen(req) as r:
        result = json.loads(r.read())
        print(f"OK: {path} → {result['commit']['sha'][:7]}")

upload("path/on/github.html", "/local/path/file.html", "commit message")
```

**Token security rule**: GitHub Personal Access Tokens (Classic, `repo` scope) must be **revoked after every session**. Remind the user at the end of every session that uploads files. Path: GitHub → Profile photo → Settings → Developer settings → Personal access tokens → Tokens (classic) → delete.

**Write-before-write rule**: The Write tool requires the file to have been Read first in the same session, even if doing a full rewrite. Always `Read` 5 lines of an existing file before `Write`.

---

## Design system

All pages use inline CSS (no external stylesheets except Google Fonts). Consistent design tokens:

```css
/* Fonts */
font-family: 'Poppins', sans-serif;          /* English */
font-family: 'Noto Sans Hebrew', sans-serif; /* Hebrew (dir="rtl") */

/* Brand colours */
--blue:    #2563EB;   /* primary */
--blue-d:  #1D4ED8;   /* hover */
--blue-lt: #EFF6FF;   /* tint */
--green:   #16A34A;
--red:     #DC2626;
--gold:    #D97706;
--purple:  #7C3AED;
--bg:      #F1F5F9;   /* page bg */
--white:   #FFFFFF;
--border:  #E2E8F0;
--text:    #0F172A;
--muted:   #64748B;

/* Section colours (homepage) */
--c-lit:   #7C3AED;   /* Literature / study guides */
--c-voc:   #059669;   /* Vocabulary */
--c-act:   #0891B2;   /* Activities */
--c-mat:   #DC2626;   /* Maths */
```

Homepage hero: gradient `#1E3A8A → #3B82F6`. Header/footer: `#1E3A8A`. Nav bar: sticky, dark blue.

Hebrew text always uses `font-family: 'Noto Sans Hebrew'; direction: rtl;`. Class `.he` applies both.

---

## Homepage (`index.html`)

**Current state**: Spiced-up hero with:
- Sticky dark-blue header (🏫 brand + ESL pill)
- Gradient hero section with stats pills: `2 Tools | 124 Vocab Words | עב Hebrew Support`
- Wave SVG separator
- Three content sections: **Study Guides** (purple), **Vocabulary** (green), **Activities** (teal/coming soon)
- Maths section commented out (coming later)
- Footer matching header

**Cards currently live:**
1. Little Prince Study Guide → `href="little-prince/index.html"`
2. English Vocabulary Practice → `href="vocabulary/index.html"` (tags: A1 + A2, 124 Words, 4 Modes)

**When adding new tools**: copy the card template in the relevant section. Update the hero stat count. Each card uses CSS variables `--ca`, `--ca-bg`, `--ca-border` for its colour.

---

## Vocabulary Tool (`vocabulary/index.html`)

Full Quizlet-style vocabulary practice tool. Completely self-contained HTML (~1100 lines).

### Word data
```javascript
const VOCAB = {
  A1: [ /* 49 words */ ],  // 7 categories
  A2: [ /* 75 words */ ],  // 8 categories
}
// Total: 124 words

// Word object shape:
{ e: "happy",  h: "שמח",  c: "Adjectives" }
// e = English, h = Hebrew, c = category
```

**A1 categories** (49 words): Pronouns (7), Verbs (10), People (6), Home (4), Food & Drink (5), Adjectives (8), Function Words (9)

**A2 categories** (75 words): Character (13), Describing (12), How & When (7), Story & Biblical (12), Movement (10), Thinking (11), Everyday (5), Nature (5)

### Four study modes
| Mode | Description |
|---|---|
| 🃏 **Flashcards** | 3D flip cards, Shuffle + Direction toggle, Know/Still Learning, keyboard shortcuts (Space=flip, ←→=nav, ↑↓=respond) |
| 📖 **Learn** | Adaptive multiple choice, EN↔HE direction randomised, wrong answers form next round, auto-advance on correct |
| 🎯 **Match** | 12 tiles (6 random pairs), click-to-match, live timer, best time saved per level in localStorage |
| 📝 **Test** | 10-question MCQ, mixed EN→HE and HE→EN, score ring at end, wrong-answer review list |

### State
```javascript
const SK = 'ehe-vocab-v2'; // localStorage key
// Match best time keys: 'ehe-match-best-A1', 'ehe-match-best-A2', 'ehe-match-best-ALL'
```

Level tabs: A1 (blue), A2 (purple), All (teal). Set via `setLevel('A1'|'A2'|'ALL')`.

---

## Teacher Hub (`teacher-hub/index.html`)

Password-locked teacher dashboard for AI-powered lesson generation. Completely self-contained (~600 lines).

### Access
URL: `https://nameless121.github.io/teacher-hub/`

**Auth flow:**
- First visit → password setup screen → SHA-256 hash stored in `localStorage['th-pw-hash']`
- Subsequent visits → login screen → hash comparison
- Session stored in `sessionStorage['th-authed']`
- "Lock" button in sidebar logs out

### localStorage keys
```javascript
const SK = {
  pwHash:   'th-pw-hash',    // SHA-256 of password
  apiKey:   'th-api-key',    // Anthropic API key (sk-ant-...)
  ghToken:  'th-gh-token',   // GitHub PAT (ghp_...)
  students: 'th-students',   // JSON array of student profiles
  history:  'th-history',    // JSON array of past generations (last 50)
};
```

### Student profile shape
```javascript
{
  name: "Yael Cohen",
  level: "A1",           // A1 | A2 | B1
  age: "12",
  interests: "animals, Israel, sports",
  strengths: "good vocabulary",
  weaknesses: "grammar, writing",
  notes: "any freeform notes"
}
```

### Generation modes
| Tab | What it generates | Output format |
|---|---|---|
| 📄 Reading Passage | Graded reading text with bold vocab | ===TITLE=== ===TEXT=== ===VOCAB=== |
| 📚 Vocabulary List | EN\|HE\|example\|category table | ===VOCAB=== |
| 📦 Full Lesson | Reading + vocab + Bloom's questions + writing prompt | ===TITLE=== ===TEXT=== ===VOCAB=== ===QUESTIONS=== ===WRITING=== |
| ❓ Questions | Bloom's-labelled comprehension questions | ===QUESTIONS=== |

### API call (Claude)
- Model: `claude-sonnet-4-6`
- Endpoint: `https://api.anthropic.com/v1/messages`
- Streaming SSE
- Required header: `anthropic-dangerous-direct-browser-access: true`
- System prompt: includes Mind pedagogical framework, CEFR levels, Hebrew-speaker awareness

### Push to site
"Push to Site" button creates `/readings/[slug].html` via GitHub API (PUT to `https://api.github.com/repos/Nameless121/Nameless121.github.io/contents/readings/[slug].html`). Uses the stored GitHub token. The pushed page is a standalone formatted reading page with the same site design.

---

## Mind Platform (Pedagogical Framework)

The site is the **student-facing tool layer** of a bigger AI tutoring platform called **Mind** (also: Bluebook Teacher). All content generation should align with these principles:

**Core pedagogy:**
- Retrieval practice + spaced repetition (Leitner system)
- Bloom's Taxonomy: Remember → Understand → Apply → Analyze → Evaluate → Create
- Formative assessment, not just summative
- Teacher-in-the-loop (AI drafts, teacher approves)
- Learner profile drives all content decisions

**For ESL / Hebrew speakers:**
- CEFR levels: A1 (very basic), A2 (elementary), B1 (intermediate)
- Hebrew interference patterns (common errors)
- Vocabulary: form + meaning + use framework
- Reading: 3-pass (gist → structure → deep analysis)
- Writing: RACE framework (Restate, Answer, Cite, Explain)

**UTLS** (Universal Tutor Level System): standardised placement and progress tracking using IRT-friendly items, Leitner boxes, and CBM micro-assessments.

---

## Pending work / ideas

- [ ] Add `readings/` section to homepage once first reading is pushed
- [ ] Little Prince study guide exists on GitHub but is not tracked locally — if editing, pull from API first
- [ ] Maths section on homepage (currently commented out) — future work
- [ ] Activities section placeholder — needs first tool
- [ ] Teacher hub: "Push Vocab" mode — currently only reading/lesson push is wired; vocab push would create a mini vocab tool at `/vocabulary/[lesson-slug].html`
- [ ] Teacher hub: student-specific "lesson history" view
- [ ] Homepage: auto-populate cards from a `materials.json` manifest when pushed content grows

---

## Session startup checklist

1. User provides a fresh GitHub PAT (Classic, `repo` scope)
2. Claude works locally on files in `/Users/elirana/Desktop/untitled folder/` (or cloned path on new laptop)
3. Read a file before Write (even 5 lines)
4. After uploads: remind user to revoke the token

---

## Working style preferences

- Direct, high-density output. No filler.
- No trailing summaries after code — the diff speaks for itself
- Ask only when the answer is necessary to proceed
- One Python upload block at the end of a session, not per-file unless files are independent
