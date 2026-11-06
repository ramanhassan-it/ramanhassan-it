#!/usr/bin/env python3
"""
Génère profile-card.svg : une réplique du design CheckMyGit pour un profil GitHub.
Lancé automatiquement chaque jour par GitHub Actions (voir .github/workflows/profile-card.yml).
"""
import base64
import datetime as dt
import html
import json
import math
import os
import sys
import urllib.request

# ===================== À PERSONNALISER =====================
TAGS = ["Cybersécurité", "Réseau", "Futur CFC", "Genève"]
EMAIL = "raman.hassan.it@gmail.com"
# Technologies affichées en plus des langages détectés dans tes projets
EXTRA_TECH = [
    ("Kali Linux", "#557C94"),
    ("Linux", "#FCC624"),
    ("Python", "#3572A5"),
    ("Bash", "#89E051"),
    ("Wireshark", "#1679A7"),
]
WELCOME_TITLE = "Bienvenue sur le Hub de {name}"
WELCOME_SUB = "Découvrez ses contributions open source et ses projets"
WELCOME_RIGHT = "Cybersécurité · Réseau"
OUTPUT = "profile-card.svg"
# ===========================================================

FONT = "Inter, -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif"
BG, CARD, BORDER = "#000000", "#0a0a0a", "#262626"
TEXT, MUTED, DIM = "#fafafa", "#a3a3a3", "#737373"
GREEN = "#3fb950"
LEVELS = {
    "NONE": "#161616",
    "FIRST_QUARTILE": "#0e4429",
    "SECOND_QUARTILE": "#006d32",
    "THIRD_QUARTILE": "#26a641",
    "FOURTH_QUARTILE": "#39d353",
}
MONTHS = ["Jan", "Fév", "Mar", "Avr", "Mai", "Juin", "Juil", "Août", "Sep", "Oct", "Nov", "Déc"]
MONTHS_LONG = ["janvier", "février", "mars", "avril", "mai", "juin", "juillet",
               "août", "septembre", "octobre", "novembre", "décembre"]

QUERY = """
query($login: String!) {
  user(login: $login) {
    name login avatarUrl(size: 240) location company createdAt
    followers { totalCount }
    following { totalCount }
    allRepos: repositories(ownerAffiliations: OWNER, privacy: PUBLIC) { totalCount }
    repositories(first: 100, ownerAffiliations: OWNER, isFork: false, privacy: PUBLIC,
                 orderBy: {field: STARGAZERS, direction: DESC}) {
      nodes { name description stargazerCount forkCount primaryLanguage { name color } }
    }
    contributionsCollection {
      contributionCalendar {
        totalContributions
        weeks { contributionDays { date contributionCount contributionLevel } }
      }
    }
  }
}"""

# ---------- Icônes (style Lucide, 24x24) ----------
ICONS = {
    "folder": '<path d="M20 20a2 2 0 0 0 2-2V8a2 2 0 0 0-2-2h-7.9a2 2 0 0 1-1.69-.9L9.6 3.9A2 2 0 0 0 7.93 3H4a2 2 0 0 0-2 2v13a2 2 0 0 0 2 2Z"/>',
    "star": '<path d="M11.525 2.295a.53.53 0 0 1 .95 0l2.31 4.679a2.123 2.123 0 0 0 1.595 1.16l5.166.756a.53.53 0 0 1 .294.904l-3.736 3.638a2.123 2.123 0 0 0-.611 1.878l.882 5.14a.53.53 0 0 1-.771.56l-4.618-2.428a2.122 2.122 0 0 0-1.973 0L6.396 21.01a.53.53 0 0 1-.77-.56l.881-5.139a2.122 2.122 0 0 0-.611-1.879L2.16 9.795a.53.53 0 0 1 .294-.906l5.165-.755a2.122 2.122 0 0 0 1.597-1.16z"/>',
    "users": '<path d="M16 21v-2a4 4 0 0 0-4-4H6a4 4 0 0 0-4 4v2"/><circle cx="9" cy="7" r="4"/><path d="M22 21v-2a4 4 0 0 0-3-3.87"/><path d="M16 3.13a4 4 0 0 1 0 7.75"/>',
    "clock": '<circle cx="12" cy="12" r="10"/><path d="M12 6v6l4 2"/>',
    "building": '<rect x="4" y="2" width="16" height="20" rx="2"/><path d="M9 22v-4h6v4M8 6h.01M16 6h.01M12 6h.01M12 10h.01M12 14h.01M16 10h.01M16 14h.01M8 10h.01M8 14h.01"/>',
    "pin": '<path d="M20 10c0 4.993-5.539 10.193-7.399 11.799a1 1 0 0 1-1.202 0C9.539 20.193 4 14.993 4 10a8 8 0 0 1 16 0"/><circle cx="12" cy="10" r="3"/>',
    "calendar": '<rect x="3" y="4" width="18" height="18" rx="2"/><path d="M16 2v4M8 2v4M3 10h18"/>',
    "mail": '<rect x="2" y="4" width="20" height="16" rx="2"/><path d="m22 7-8.97 5.7a1.94 1.94 0 0 1-2.06 0L2 7"/>',
    "bookmark": '<path d="m19 21-7-4-7 4V5a2 2 0 0 1 2-2h10a2 2 0 0 1 2 2v16z"/>',
    "fork": '<circle cx="12" cy="18" r="3"/><circle cx="6" cy="6" r="3"/><circle cx="18" cy="6" r="3"/><path d="M18 9v2c0 .6-.4 1-1 1H7c-.6 0-1-.4-1-1V9M12 12v3"/>',
    "chevron": '<path d="m6 9 6 6 6-6"/>',
}
GITHUB_MARK = ('<path fill="{c}" d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49'
               '-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 '
               '2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 '
               '2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 '
               '2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0'
               '-4.42-3.58-8-8-8z"/>')


def esc(s):
    return html.escape(str(s or ""), quote=True)


def icon(name, x, y, size, color=MUTED, sw=2, fill="none"):
    s = size / 24
    return (f'<g transform="translate({x:.1f},{y:.1f}) scale({s:.4f})" fill="{fill}" stroke="{color}" '
            f'stroke-width="{sw}" stroke-linecap="round" stroke-linejoin="round">{ICONS[name]}</g>')


def gh_mark(x, y, size, color=TEXT):
    return f'<g transform="translate({x:.1f},{y:.1f}) scale({size/16:.4f})">{GITHUB_MARK.format(c=color)}</g>'


def text(x, y, s, size=12, color=TEXT, weight=400, anchor="start", extra=""):
    return (f'<text x="{x:.1f}" y="{y:.1f}" font-size="{size}" fill="{color}" font-weight="{weight}" '
            f'text-anchor="{anchor}" {extra}>{esc(s)}</text>')


def card(x, y, w, h, r=8):
    return f'<rect x="{x}" y="{y}" width="{w}" height="{h}" rx="{r}" fill="{CARD}" stroke="{BORDER}"/>'


def fmt(n):
    if n < 1000:
        return str(n)
    v = f"{n / 1000:.1f}".rstrip("0").rstrip(".")
    return f"{v}k"


def tw(s, size):
    """Largeur approximative d'un texte (police sans-serif)."""
    return len(s) * size * 0.56


def wrap(s, max_chars, max_lines=2):
    words, lines, cur = (s or "").split(), [], ""
    for w in words:
        if len(cur) + len(w) + (1 if cur else 0) <= max_chars:
            cur = f"{cur} {w}".strip()
        else:
            lines.append(cur)
            cur = w
    if cur:
        lines.append(cur)
    if len(lines) > max_lines:
        lines = lines[:max_lines]
        lines[-1] = lines[-1][: max_chars - 1].rstrip() + "…"
    return lines


def lighten(color, k=0.35):
    c = color.lstrip("#")
    r, g, b = (int(c[i:i + 2], 16) for i in (0, 2, 4))
    r, g, b = (round(v + (255 - v) * k) for v in (r, g, b))
    return f"#{r:02x}{g:02x}{b:02x}"


def hex_alpha(color, a):
    c = color.lstrip("#")
    r, g, b = int(c[0:2], 16), int(c[2:4], 16), int(c[4:6], 16)
    return f"rgba({r},{g},{b},{a})"


# ---------- Données ----------
def fetch(login, token):
    req = urllib.request.Request(
        "https://api.github.com/graphql",
        data=json.dumps({"query": QUERY, "variables": {"login": login}}).encode(),
        headers={"Authorization": f"bearer {token}", "Content-Type": "application/json",
                 "User-Agent": "profile-card"},
    )
    with urllib.request.urlopen(req, timeout=30) as r:
        data = json.load(r)
    if data.get("errors"):
        sys.exit(f"Erreur GitHub : {data['errors']}")
    return data["data"]["user"]


def avatar_data_uri(url):
    try:
        with urllib.request.urlopen(urllib.request.Request(url, headers={"User-Agent": "profile-card"}), timeout=30) as r:
            mime = r.headers.get_content_type() or "image/png"
            return f"data:{mime};base64,{base64.b64encode(r.read()).decode()}"
    except Exception:
        return None


# ---------- Rendu ----------
def render(u, avatar_uri):
    W = 1080
    login = u["login"]
    name = u.get("name") or login
    repos = u["repositories"]["nodes"]
    stars = sum(r["stargazerCount"] for r in repos)
    created = dt.datetime.fromisoformat(u["createdAt"].replace("Z", "+00:00"))
    years = (dt.datetime.now(dt.timezone.utc) - created).days // 365
    cal = u["contributionsCollection"]["contributionCalendar"]
    projects = repos[:6]

    rows = max(1, math.ceil(len(projects) / 2))
    H = 667 + rows * 123 + 40
    o = [f'<svg xmlns="http://www.w3.org/2000/svg" width="{W}" height="{H}" viewBox="0 0 {W} {H}" font-family="{FONT}">',
         f'<rect width="{W}" height="{H}" fill="{BG}"/>']

    # ===== Colonne gauche =====
    cx, cy, r = 163, 147, 120
    o.append(f'<defs><clipPath id="av"><circle cx="{cx}" cy="{cy}" r="{r}"/></clipPath></defs>')
    if avatar_uri:
        o.append(f'<image href="{avatar_uri}" x="{cx-r}" y="{cy-r}" width="{2*r}" height="{2*r}" '
                 f'clip-path="url(#av)" preserveAspectRatio="xMidYMid slice"/>')
    else:
        o.append(f'<circle cx="{cx}" cy="{cy}" r="{r}" fill="#1f1f1f"/>')
        o.append(text(cx, cy + 22, name[:1].upper(), 64, MUTED, 600, "middle"))
    o.append(f'<circle cx="{cx}" cy="{cy}" r="{r}" fill="none" stroke="#2e2e2e" stroke-width="3"/>')

    o.append(text(43, 301, name, 21, TEXT, 600))
    o.append(text(43, 329, f"@{login}", 17, MUTED))

    o.append(f'<rect x="43" y="347" width="242" height="32" rx="6" fill="#1a1a1a" stroke="#2e2e2e"/>')
    label = "Suivre sur GitHub"
    lw = tw(label, 12.5)
    bx = 164 - (lw + 20) / 2
    o.append(gh_mark(bx, 355.5, 15))
    o.append(text(bx + 21, 368, label, 12.5, TEXT, 500))

    # tags
    tx, ty = 43, 393
    for t in TAGS:
        w = tw(t, 10) + 18
        if tx + w > 285:
            tx, ty = 43, ty + 23
        o.append(f'<rect x="{tx}" y="{ty}" width="{w:.1f}" height="17" rx="8.5" fill="none" stroke="#3a3a3a"/>')
        o.append(text(tx + w / 2, ty + 12, t, 10, "#d4d4d4", 400, "middle"))
        tx += w + 6

    ry = ty + 50
    info = []
    if u.get("company"):
        info.append(("building", u["company"]))
    if u.get("location"):
        info.append(("pin", u["location"]))
    info.append(("calendar", f"Inscrit en {MONTHS_LONG[created.month-1]} {created.year}"))
    if EMAIL:
        info.append(("mail", EMAIL))
    for ic, val in info:
        o.append(icon(ic, 43, ry - 11, 14, MUTED))
        o.append(text(66, ry, val, 12.5, "#c4c4c4"))
        ry += 23
    ry += 6
    o.append(icon("users", 43, ry - 11, 14, TEXT))
    f, g = u["followers"]["totalCount"], u["following"]["totalCount"]
    ftxt = f"{f}"
    o.append(f'<text x="66" y="{ry}" font-size="12.5" fill="{MUTED}"><tspan fill="{TEXT}" font-weight="600">{f}</tspan> '
             f'abonnés  ·  <tspan fill="{TEXT}" font-weight="600">{g}</tspan> abonnements</text>')

    # ===== Colonne droite =====
    X, RW = 312, 725
    # Welcome
    o.append(card(X, 27, RW, 81))
    o.append(text(X + 20, 64, WELCOME_TITLE.format(name=name), 17, TEXT, 600))
    o.append(text(X + 20, 86, WELCOME_SUB, 12.5, MUTED))
    o.append(text(X + RW - 20, 72, WELCOME_RIGHT, 10.5, GREEN, 500, "end"))

    # 4 stats
    stats = [("folder", fmt(u["allRepos"]["totalCount"]), "TOTAL REPOS"),
             ("star", fmt(stars), "TOTAL STARS"),
             ("users", fmt(f), "ABONNÉS"),
             ("clock", str(years) if years else "<1", "ANNÉES ACTIF")]
    sw_ = 171
    gap = (RW - 4 * sw_) / 3
    for i, (ic, val, lab) in enumerate(stats):
        sx = X + i * (sw_ + gap)
        o.append(card(sx, 130, sw_, 91))
        mid = sx + sw_ / 2
        o.append(icon(ic, mid - 8, 143, 16, MUTED, 1.8))
        o.append(text(mid, 190, val, 25, TEXT, 700, "middle"))
        o.append(text(mid, 206, lab, 9.5, MUTED, 500, "middle", 'letter-spacing="0.8"'))

    # Contributions
    o.append(card(X, 243, RW, 180))
    o.append(text(X + 14, 270, "Contributions", 14, TEXT, 600))
    o.append(f'<text x="{X+RW-14}" y="270" font-size="12" fill="{MUTED}" text-anchor="end">Total : '
             f'<tspan fill="{TEXT}" font-weight="700">{fmt(cal["totalContributions"])}</tspan></text>')
    weeks = cal["weeks"]
    gx0, gy0, step, cell = 358, 293, 12.55, 10.4
    last_label_x, prev_m = -100, None
    for wi, wk in enumerate(weeks):
        x = gx0 + wi * step
        days = wk["contributionDays"]
        m = int(days[0]["date"][5:7])
        if m != prev_m and x - last_label_x > 24:
            o.append(text(x, 287, MONTHS[m - 1], 8.5, DIM))
            last_label_x = x
        prev_m = m
        for d in days:
            wd = dt.date.fromisoformat(d["date"]).isoweekday() % 7  # dimanche = 0
            y = gy0 + wd * step
            col = LEVELS.get(d.get("contributionLevel"), LEVELS["NONE"])
            o.append(f'<rect x="{x:.1f}" y="{y:.1f}" width="{cell}" height="{cell}" rx="2" fill="{col}"/>')
    for wd, lab in ((1, "Lun"), (3, "Mer"), (5, "Ven")):
        o.append(text(X + 14, gy0 + wd * step + 8.5, lab, 8.5, DIM))
    o.append(text(X + 14, 408, "Activité des 52 dernières semaines.", 10, DIM))

    # Core technologies
    o.append(card(X, 444, 352, 161))
    o.append(text(X + 14, 468, "TECHNOLOGIES", 10, MUTED, 500, "start", 'letter-spacing="0.8"'))
    techs, seen = [], set()
    for rp in repos:
        pl = rp.get("primaryLanguage")
        if pl and pl["name"] not in seen:
            seen.add(pl["name"])
            techs.append((pl["name"], pl.get("color") or "#8b949e"))
    for nme, colr in EXTRA_TECH:
        if nme not in seen:
            seen.add(nme)
            techs.append((nme, colr))
    cx_, cy_ = X + 14, 484
    for nme, colr in techs[:12]:
        w = tw(nme, 10.5) + 26
        if cx_ + w > X + 352 - 14:
            cx_, cy_ = X + 14, cy_ + 28
        if cy_ > 570:
            break
        o.append(f'<rect x="{cx_:.1f}" y="{cy_}" width="{w:.1f}" height="20" rx="10" '
                 f'fill="{hex_alpha(colr, 0.12)}" stroke="{hex_alpha(colr, 0.45)}"/>')
        o.append(f'<circle cx="{cx_+10:.1f}" cy="{cy_+10}" r="3" fill="{colr}"/>')
        o.append(text(cx_ + 18, cy_ + 14, nme, 10.5, lighten(colr) if colr.lower() not in ("#555555", "#000000") else "#c4c4c4"))
        cx_ += w + 8

    # Donut langages
    DX = X + 373
    o.append(card(DX, 444, 352, 161))
    counts = {}
    colors = {}
    for rp in repos[:20]:
        pl = rp.get("primaryLanguage")
        if pl:
            counts[pl["name"]] = counts.get(pl["name"], 0) + 1
            colors[pl["name"]] = pl.get("color") or "#8b949e"
    total = sum(counts.values())
    top = sorted(counts.items(), key=lambda kv: -kv[1])[:5]
    dcx, dcy, dr, dsw = DX + 60, 512, 38, 15
    circ = 2 * math.pi * dr
    o.append(f'<circle cx="{dcx}" cy="{dcy}" r="{dr}" fill="none" stroke="#1f1f1f" stroke-width="{dsw}"/>')
    acc = 0.0
    for nme, c in top:
        seg = circ * c / total
        gapl = 1.5 if len(top) > 1 else 0
        o.append(f'<circle cx="{dcx}" cy="{dcy}" r="{dr}" fill="none" stroke="{colors[nme]}" stroke-width="{dsw}" '
                 f'stroke-dasharray="{max(seg-gapl,0.1):.2f} {circ:.2f}" stroke-dashoffset="{-acc:.2f}" '
                 f'transform="rotate(-90 {dcx} {dcy})"/>')
        acc += seg
    o.append(text(dcx, dcy + 4, f"{len(top)} langage{'s' if len(top) > 1 else ''}", 9.5, MUTED, 400, "middle"))
    ly = 470
    if not top:
        o.append(text(DX + 132, 515, "Aucun langage détecté pour l'instant", 11, DIM))
    for nme, c in top:
        o.append(f'<circle cx="{DX+138}" cy="{ly-4}" r="4.5" fill="{colors[nme]}"/>')
        o.append(text(DX + 152, ly, nme, 12, TEXT))
        o.append(text(DX + 337, ly, f"{round(100 * c / total)}%", 12, MUTED, 400, "end"))
        ly += 23
    if repos:
        o.append(text(DX + 14, 589, f"Basé sur les {min(len(repos), 20)} meilleurs projets", 10, DIM))

    # Notable projects
    o.append(icon("bookmark", X, 630, 16, TEXT))
    o.append(text(X + 24, 644, "Projets notables", 15, TEXT, 600))
    o.append(f'<rect x="{X+RW-92}" y="625" width="92" height="28" rx="6" fill="{CARD}" stroke="{BORDER}"/>')
    o.append(text(X + RW - 80, 643, "Top étoiles", 11, "#d4d4d4"))
    o.append(icon("chevron", X + RW - 20, 634, 10, MUTED))

    PW, PH = 356, 110
    if not projects:
        o.append(card(X, 667, PW, PH))
        o.append(icon("folder", X + 14, 683, 13, MUTED))
        o.append(text(X + 34, 694, "Bientôt", 13.5, GREEN, 500))
        o.append(text(X + 14, 722, "Premiers projets publics en préparation", 12, "#c4c4c4"))
        o.append(text(X + 14, 740, "(homelab, réseau, sécurité).", 12, "#c4c4c4"))
    for i, rp in enumerate(projects):
        px = X + (i % 2) * (PW + 13)
        py = 667 + (i // 2) * 123
        o.append(card(px, py, PW, PH))
        o.append(icon("folder", px + 14, py + 16, 13, MUTED))
        nm = rp["name"] if len(rp["name"]) <= 32 else rp["name"][:31] + "…"
        o.append(text(px + 34, py + 27, nm, 13.5, GREEN, 500))
        pl = rp.get("primaryLanguage")
        if pl:
            lt = pl["name"][:10]
            lwid = tw(lt, 9) + 14
            lc = pl.get("color") or "#8b949e"
            o.append(f'<rect x="{px+PW-14-lwid:.1f}" y="{py+14}" width="{lwid:.1f}" height="16" rx="8" '
                     f'fill="{hex_alpha(lc, 0.12)}"/>')
            o.append(text(px + PW - 14 - lwid / 2, py + 25.5, lt, 9, lc if lc.lower() != "#555555" else MUTED, 500, "middle"))
        for li, line in enumerate(wrap(rp.get("description") or "Pas de description", 50)):
            o.append(text(px + 14, py + 55 + li * 17, line, 12, "#c4c4c4"))
        o.append(icon("star", px + 14, py + 82, 13, MUTED, 2, MUTED))
        o.append(text(px + 32, py + 93, fmt(rp["stargazerCount"]), 11, MUTED))
        fx = px + 32 + tw(fmt(rp["stargazerCount"]), 11) + 16
        o.append(icon("fork", fx, py + 82, 13, MUTED))
        o.append(text(fx + 18, py + 93, fmt(rp["forkCount"]), 11, MUTED))

    # Footer
    fy = H - 18
    lbl = f"github.com/{login}"
    o.append(gh_mark(W - 30 - tw(lbl, 11) - 20, fy - 11, 13, MUTED))
    o.append(text(W - 30, fy, lbl, 11, MUTED, 400, "end"))
    o.append("</svg>")
    return "\n".join(o)


def demo_user():
    """Données fictives pour tester le rendu sans connexion."""
    import random
    random.seed(4)
    start = dt.date.today() - dt.timedelta(days=dt.date.today().isoweekday() % 7 + 52 * 7)
    weeks, total = [], 0
    lv = ["NONE", "FIRST_QUARTILE", "SECOND_QUARTILE", "THIRD_QUARTILE", "FOURTH_QUARTILE"]
    for w in range(53):
        days = []
        for d in range(7):
            day = start + dt.timedelta(days=w * 7 + d)
            if day > dt.date.today():
                break
            l = random.choices(range(5), [3, 4, 3, 1, 0.6])[0]
            total += l * 2
            days.append({"date": day.isoformat(), "contributionCount": l * 2, "contributionLevel": lv[l]})
        if days:
            weeks.append({"contributionDays": days})
    L = lambda n, c: {"name": n, "color": c}
    return {
        "name": "Linus Torvalds", "login": "torvalds", "avatarUrl": "", "location": "Portland, OR",
        "company": "Linux Foundation", "createdAt": "2011-09-03T15:26:22Z",
        "followers": {"totalCount": 270868}, "following": {"totalCount": 0}, "allRepos": {"totalCount": 26},
        "repositories": {"nodes": [
            {"name": "linux", "description": "Linux kernel source tree", "stargazerCount": 213000, "forkCount": 59900, "primaryLanguage": L("C", "#555555")},
            {"name": "subsurface", "description": "This is the official upstream of the Subsurface divelog program", "stargazerCount": 3000, "forkCount": 562, "primaryLanguage": L("C++", "#f34b7d")},
            {"name": "uemacs", "description": "Random version of microemacs with my private modificatons", "stargazerCount": 1800, "forkCount": 288, "primaryLanguage": L("C", "#555555")},
            {"name": "GuitarPedal", "description": "Linus learns analog circuits", "stargazerCount": 1500, "forkCount": 47, "primaryLanguage": L("C", "#555555")},
            {"name": "libdc", "description": "Subsurface version of libdivecomputer, upstream at github.com/libdivecomputer/libdivecomputer", "stargazerCount": 113, "forkCount": 104, "primaryLanguage": L("C", "#555555")},
            {"name": "test-tlb", "description": "Stupid memory latency and TLB tester", "stargazerCount": 800, "forkCount": 90, "primaryLanguage": L("JavaScript", "#f1e05a")},
        ]},
        "contributionsCollection": {"contributionCalendar": {"totalContributions": total, "weeks": weeks}},
    }


def main():
    if "--demo" in sys.argv:
        u, av = demo_user(), None
    elif "--empty" in sys.argv:
        u = demo_user()
        u.update({"name": "Raman Hassan", "login": "ramanhassan-it", "location": "Genève, Suisse", "company": None,
                  "createdAt": "2026-11-01T10:00:00Z", "followers": {"totalCount": 0}, "allRepos": {"totalCount": 1},
                  "repositories": {"nodes": []}})
        for w in u["contributionsCollection"]["contributionCalendar"]["weeks"]:
            for d in w["contributionDays"]:
                d["contributionLevel"], d["contributionCount"] = "NONE", 0
        u["contributionsCollection"]["contributionCalendar"]["totalContributions"] = 0
        av = None
    else:
        token = os.environ.get("GH_TOKEN")
        login = os.environ.get("GH_USER")
        if not token or not login:
            sys.exit("GH_TOKEN et GH_USER sont requis.")
        u = fetch(login, token)
        av = avatar_data_uri(u["avatarUrl"])
    with open(OUTPUT, "w", encoding="utf-8") as fh:
        fh.write(render(u, av))
    print(f"{OUTPUT} généré.")


if __name__ == "__main__":
    main()
    
