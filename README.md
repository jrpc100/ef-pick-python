# ef-pick-python
import os
import sqlite3
import hashlib
import hmac
import streamlit as st
from pathlib import Path

APP_DIR = Path(__file__).parent
DB_PATH = Path(os.getenv("EF_PICK_DB", str(APP_DIR / "ef_pick.db")))

st.set_page_config(page_title="EF PICK", page_icon="⚽", layout="wide")

# ---------- Language ----------
if "lang" not in st.session_state:
    st.session_state.lang = "ml"

TEXT = {
    "ml": {
        "app_name": "EF PICK",
        "tagline": "eFootball പഠിക്കാം. മികച്ച രീതിയിൽ കളിക്കാം.",
        "home": "ഹോം", "tips": "eFootball ടിപ്സ്", "skills": "Skill Tutorials",
        "coach": "EF Coach AI", "admin": "Admin Panel", "language": "ഭാഷ",
        "search": "ടിപ്സ് തിരയുക", "all": "എല്ലാം", "attack": "അറ്റാക്ക്",
        "defence": "ഡിഫൻസ്", "formation": "ഫോർമേഷൻ", "training": "ട്രെയിനിംഗ്",
        "settings": "സെറ്റിംഗ്സ്", "saved": "സേവ് ചെയ്തവ", "save": "സേവ് ചെയ്യുക",
        "unsave": "സേവ് മാറ്റുക", "mode": "Control Mode", "touch": "Touch Control",
        "classic": "Classic Control Pad", "video": "YouTube Video URL",
        "ask": "EF Coach-നോട് ചോദിക്കൂ", "send": "ചോദിക്കുക", "login": "Admin Login",
        "password": "Admin Password", "logout": "Logout", "dashboard": "Dashboard",
        "manage_tips": "Tips നിയന്ത്രിക്കുക", "manage_skills": "Skills നിയന്ത്രിക്കുക",
        "add_tip": "പുതിയ Tip ചേർക്കുക", "add_skill": "പുതിയ Skill ചേർക്കുക",
        "title": "Title", "category": "Category", "description": "വിവരണം",
        "steps": "Steps (ഓരോ വരിയിലും ഒന്ന്)", "delete": "Delete",
        "edit": "Edit", "save_changes": "Save Changes", "add": "Add",
        "no_results": "ഫലം കണ്ടെത്തിയില്ല.", "welcome": "സ്വാഗതം!",
        "home_desc": "eFootball Mobile-നുള്ള ടിപ്സ്, സ്കിൽസ്, ടാക്ടിക്സ് എന്നിവ ഒരിടത്ത്.",
        "quick": "Quick Start", "tip_count": "ആകെ ടിപ്സ്", "skill_count": "ആകെ Skills",
        "notice": "ഈ EF Coach ഒരു offline rule-based helper ആണ്; യഥാർത്ഥ generative AI അല്ല.",
        "need_login": "Admin Panel ഉപയോഗിക്കാൻ login ചെയ്യണം.",
        "no_password": "EF_ADMIN_PASSWORD സെറ്റ് ചെയ്തിട്ടില്ല. Environment variable അല്ലെങ്കിൽ Streamlit secrets-ൽ ചേർക്കുക.",
        "bad_login": "Password ശരിയല്ല.", "empty": "ഇവിടെ ഒന്നുമില്ല.",
        "control": "Control", "skill_name": "Skill Name", "add_video": "Video URL ചേർക്കുക",
        "video_help": "YouTube URL ഓപ്ഷണൽ ആണ്.", "saved_ok": "സേവ് ചെയ്തു.",
        "tip_saved": "Saved tips", "coach_hint": "ഉദാ: How to defend? / Best formation?",
    },
    "en": {
        "app_name": "EF PICK", "tagline": "Learn eFootball. Play smarter.",
        "home": "Home", "tips": "eFootball Tips", "skills": "Skill Tutorials",
        "coach": "EF Coach AI", "admin": "Admin Panel", "language": "Language",
        "search": "Search tips", "all": "All", "attack": "Attack",
        "defence": "Defence", "formation": "Formation", "training": "Training",
        "settings": "Settings", "saved": "Saved", "save": "Save",
        "unsave": "Unsave", "mode": "Control Mode", "touch": "Touch Control",
        "classic": "Classic Control Pad", "video": "YouTube Video URL",
        "ask": "Ask EF Coach", "send": "Ask", "login": "Admin Login",
        "password": "Admin Password", "logout": "Logout", "dashboard": "Dashboard",
        "manage_tips": "Manage Tips", "manage_skills": "Manage Skills",
        "add_tip": "Add a Tip", "add_skill": "Add a Skill",
        "title": "Title", "category": "Category", "description": "Description",
        "steps": "Steps (one per line)", "delete": "Delete", "edit": "Edit",
        "save_changes": "Save Changes", "add": "Add", "no_results": "No results found.",
        "welcome": "Welcome!", "home_desc": "Tips, skills and tactics for eFootball Mobile in one place.",
        "quick": "Quick Start", "tip_count": "Total Tips", "skill_count": "Total Skills",
        "notice": "This EF Coach is an offline rule-based helper, not a generative AI model.",
        "need_login": "Log in to use the Admin Panel.",
        "no_password": "EF_ADMIN_PASSWORD is not set. Add it as an environment variable or Streamlit secret.",
        "bad_login": "Incorrect password.", "empty": "Nothing here yet.",
        "control": "Control", "skill_name": "Skill Name", "add_video": "Add Video URL",
        "video_help": "YouTube URL is optional.", "saved_ok": "Saved.",
        "tip_saved": "Saved tips", "coach_hint": "e.g. How to defend? / Best formation?",
    }
}
t = TEXT[st.session_state.lang]

# ---------- Database ----------
def db():
    conn = sqlite3.connect(DB_PATH, check_same_thread=False)
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    with db() as con:
        con.execute("""CREATE TABLE IF NOT EXISTS tips(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL, category TEXT NOT NULL,
            description TEXT NOT NULL, created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)""")
        con.execute("""CREATE TABLE IF NOT EXISTS skills(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL, control TEXT NOT NULL,
            steps TEXT NOT NULL, video_url TEXT DEFAULT '',
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)""")
        con.execute("""CREATE TABLE IF NOT EXISTS coach_replies(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            keyword TEXT NOT NULL, reply TEXT NOT NULL)""")
        if con.execute("SELECT COUNT(*) FROM tips").fetchone()[0] == 0:
            seed_tips = [
                ("Build-up: Don't rush", "Attack", "Use short passes to move the opponent's defensive line. Look for space before playing a through pass."),
                ("Use the wings", "Attack", "Switch the ball to the opposite wing when one side is crowded. Avoid forcing passes into traffic."),
                ("Defend with patience", "Defence", "Keep your defensive shape. Use pressure carefully and avoid pulling centre-backs out of position."),
                ("Protect the middle", "Defence", "Block central passing lanes and guide the opponent toward the touchline."),
                ("Choose a balanced formation", "Formation", "Pick a formation that fits your players and preferred style. Keep enough support around the ball."),
                ("Train player roles", "Training", "Learn each player's strengths and use them in suitable positions.")
            ]
            con.executemany("INSERT INTO tips(title,category,description) VALUES(?,?,?)", seed_tips)
        if con.execute("SELECT COUNT(*) FROM skills").fetchone()[0] == 0:
            seed_skills = [
                ("Double Touch", "Touch Control", "Move the player and use the skill command when available. Practise slowly before using it in matches.", ""),
                ("Double Touch + Pass Cancel", "Touch Control", "Perform Double Touch, then use the pass-cancel input at the right moment. Timing varies by control setup.", ""),
                ("Fake Shot", "Touch Control", "Start a shot action and cancel it before the player completes the shot. Practise in training.", ""),
                ("Marseille Turn", "Classic Control Pad", "Use the skill input with a directional movement. The exact gesture depends on your control settings.", ""),
                ("Marseille Turn + Pass Cancel", "Classic Control Pad", "Perform the turn, then use pass cancel with precise timing. Practise in training mode.", ""),
                ("Chop Turn", "Touch Control", "Use a sharp directional change with the skill input to turn away from pressure.", ""),
                ("Elastico / Flip Flap", "Touch Control", "Use the required skill input with a player who has the appropriate skill. Timing and availability can vary.", "")
            ]
            con.executemany("INSERT INTO skills(name,control,steps,video_url) VALUES(?,?,?,?)", seed_skills)

init_db()

def get_tips():
    with db() as con:
        return con.execute("SELECT * FROM tips ORDER BY id DESC").fetchall()

def get_skills():
    with db() as con:
        return con.execute("SELECT * FROM skills ORDER BY id DESC").fetchall()

# ---------- Styling ----------
st.markdown("""
<style>
.block-container {padding-top: 1.3rem; max-width: 1100px;}
.hero {padding: 1.5rem; border-radius: 18px; background: linear-gradient(120deg,#101c3c,#174c69); color: white; margin-bottom: 1rem;}
.hero h1 {font-size: 2.3rem; margin-bottom: .3rem;}
.small-muted {opacity:.75; font-size:.9rem;}
div[data-testid="stMetric"] {border:1px solid rgba(128,128,128,.25); padding:14px; border-radius:14px;}
</style>
""", unsafe_allow_html=True)

# ---------- Sidebar navigation ----------
with st.sidebar:
    st.markdown("## ⚽ EF PICK")
    st.session_state.lang = st.radio(
        t["language"], ["ml", "en"],
        format_func=lambda x: "മലയാളം" if x == "ml" else "English",
        index=0 if st.session_state.lang == "ml" else 1,
        horizontal=True
    )
    t = TEXT[st.session_state.lang]
    page = st.radio("Menu", [t["home"], t["tips"], t["skills"], t["coach"], t["admin"]])
    st.caption("eFootball Mobile • EF PICK")

# ---------- Pages ----------
if page == t["home"]:
    tips = get_tips()
    skills = get_skills()
    st.markdown(f'<div class="hero"><h1>⚽ {t["app_name"]}</h1><div>{t["tagline"]}</div></div>', unsafe_allow_html=True)
    st.subheader(t["welcome"])
    st.write(t["home_desc"])
    c1, c2 = st.columns(2)
    c1.metric(t["tip_count"], len(tips))
    c2.metric(t["skill_count"], len(skills))
    st.subheader(t["quick"])
    a, b = st.columns(2)
    with a:
        st.info("⚡ " + t["tips"])
        st.write("Attack • Defence • Formation • Training")
    with b:
        st.info("🎮 " + t["skills"])
        st.write("Touch Control • Classic Control Pad")
    st.caption(t["notice"])

elif page == t["tips"]:
    st.title("📘 " + t["tips"])
    query = st.text_input(t["search"])
    categories = [t["all"], "Attack", "Defence", "Formation", "Training", "Settings"]
    cat = st.selectbox(t["category"], categories)
    saved_ids = st.session_state.setdefault("saved_tips", [])
    rows = get_tips()
    filtered = [r for r in rows if (cat == t["all"] or r["category"] == cat)
                and (query.lower() in (r["title"] + " " + r["description"] + " " + r["category"]).lower())]
    if not filtered:
        st.info(t["no_results"])
    for r in filtered:
        with st.expander(f"{r['title']}  ·  {r['category']}"):
            st.write(r["description"])
            if r["id"] in saved_ids:
                if st.button(t["unsave"], key=f"unsave{r['id']}"):
                    saved_ids.remove(r["id"]); st.rerun()
            else:
                if st.button(t["save"], key=f"save{r['id']}"):
                    saved_ids.append(r["id"]); st.rerun()
    st.subheader("⭐ " + t["tip_saved"])
    saved = [r for r in rows if r["id"] in saved_ids]
    if saved:
        for r in saved:
            st.write("• " + r["title"])
    else:
        st.caption(t["empty"])

elif page == t["skills"]:
    st.title("🎮 " + t["skills"])
    control_mode = st.radio(t["mode"], [t["all"], "Touch Control", "Classic Control Pad"], horizontal=True)
    for r in get_skills():
        if control_mode != t["all"] and r["control"] != control_mode:
            continue
        with st.expander(f"{r['name']}  ·  {r['control']}"):
            for n, line in enumerate(r["steps"].splitlines(), 1):
                st.write(f"{n}. {line}")
            if r["video_url"].strip():
                st.link_button("▶ " + t["video"], r["video_url"])
            else:
                st.caption(t["video_help"])

elif page == t["coach"]:
    st.title("🤖 " + t["coach"])
    st.info(t["notice"])
    question = st.text_input(t["ask"], placeholder=t["coach_hint"])
    if st.button(t["send"], type="primary") and question.strip():
        q = question.lower()
        if any(x in q for x in ["defend", "defence", "defense", "ഡിഫൻസ്", "പ്രതിരോധ"]):
            answer = "Defend patiently. Keep your defensive shape, block central passing lanes, and avoid pulling centre-backs out of position."
        elif any(x in q for x in ["attack", "അറ്റാക്ക്", "ആക്രമണം", "score", "goal"]):
            answer = "Build up with short passes, use width, create space, and avoid forcing through passes. Time your runs."
        elif any(x in q for x in ["formation", "ഫോർമേഷൻ", "ടാക്ടിക്"]):
            answer = "Choose a balanced formation that fits your players and play style. Keep midfield support and avoid leaving large gaps."
        elif any(x in q for x in ["double touch", "skill", "ഡബിൾ", "സ്കിൽ"]):
            answer = "Practise skills in training first. Double Touch and other skill inputs can vary with control mode and player abilities."
        elif any(x in q for x in ["pass", "പാസ്"]):
            answer = "Use short passes to keep possession. Look at player positioning before attempting a through pass."
        else:
            answer = "Try asking about attacking, defending, formations, passing, or skill moves. Add more coach replies from Admin Panel."
        st.markdown("**EF Coach:**")
        st.write(answer)

elif page == t["admin"]:
    st.title("🔐 " + t["admin"])
    admin_password = os.getenv("EF_ADMIN_PASSWORD", "")
    try:
        if not admin_password:
            admin_password = st.secrets.get("EF_ADMIN_PASSWORD", "")
    except Exception:
        pass

    if not admin_password:
        st.warning(t["no_password"])
        st.markdown("Set `EF_ADMIN_PASSWORD` in your hosting provider's environment variables or `.streamlit/secrets.toml`.")
    elif not st.session_state.get("admin_logged_in", False):
        st.write(t["need_login"])
        entered = st.text_input(t["password"], type="password")
        if st.button(t["login"], type="primary"):
            if hmac.compare_digest(entered, str(admin_password)):
                st.session_state.admin_logged_in = True
                st.rerun()
            else:
                st.error(t["bad_login"])
    else:
        top1, top2 = st.columns([4,1])
        top1.success("Admin authenticated")
        if top2.button(t["logout"]):
            st.session_state.admin_logged_in = False
            st.rerun()

        st.subheader("📊 " + t["dashboard"])
        c1, c2 = st.columns(2)
        c1.metric(t["tip_count"], len(get_tips()))
        c2.metric(t["skill_count"], len(get_skills()))

        tab1, tab2 = st.tabs([t["manage_tips"], t["manage_skills"]])
        with tab1:
            st.markdown("### " + t["add_tip"])
            with st.form("new_tip_form"):
                new_title = st.text_input(t["title"])
                new_cat = st.selectbox(t["category"], ["Attack","Defence","Formation","Training","Settings"])
                new_desc = st.text_area(t["description"])
                submitted = st.form_submit_button(t["add"])
            if submitted:
                if new_title.strip() and new_desc.strip():
                    with db() as con:
                        con.execute("INSERT INTO tips(title,category,description) VALUES(?,?,?)",
                                    (new_title.strip(), new_cat, new_desc.strip()))
                    st.success(t["saved_ok"]); st.rerun()
                else:
                    st.error("Title and description are required.")

            st.markdown("### Existing Tips")
            for r in get_tips():
                with st.expander(f"{r['title']} · {r['category']}"):
                    with st.form(f"edit_tip_{r['id']}"):
                        et = st.text_input(t["title"], value=r["title"])
                        ec = st.selectbox(t["category"], ["Attack","Defence","Formation","Training","Settings"],
                                          index=["Attack","Defence","Formation","Training","Settings"].index(r["category"])
                                          if r["category"] in ["Attack","Defence","Formation","Training","Settings"] else 0)
                        ed = st.text_area(t["description"], value=r["description"])
                        save = st.form_submit_button(t["save_changes"])
                    if save:
                        with db() as con:
                            con.execute("UPDATE tips SET title=?,category=?,description=? WHERE id=?",
                                        (et.strip(), ec, ed.strip(), r["id"]))
                        st.success(t["saved_ok"]); st.rerun()
                    if st.button(t["delete"], key=f"del_tip_{r['id']}"):
                        with db() as con:
                            con.execute("DELETE FROM tips WHERE id=?", (r["id"],))
                        st.rerun()

        with tab2:
            st.markdown("### " + t["add_skill"])
            with st.form("new_skill_form"):
                sn = st.text_input(t["skill_name"])
                sc = st.selectbox(t["control"], ["Touch Control","Classic Control Pad"])
                ss = st.text_area(t["steps"])
                sv = st.text_input(t["add_video"], placeholder="https://www.youtube.com/watch?v=...")
                skill_submit = st.form_submit_button(t["add"])
            if skill_submit:
                if sn.strip() and ss.strip():
                    with db() as con:
                        con.execute("INSERT INTO skills(name,control,steps,video_url) VALUES(?,?,?,?)",
                                    (sn.strip(), sc, ss.strip(), sv.strip()))
                    st.success(t["saved_ok"]); st.rerun()
                else:
                    st.error("Skill name and steps are required.")

            st.markdown("### Existing Skills")
            for r in get_skills():
                with st.expander(f"{r['name']} · {r['control']}"):
                    with st.form(f"edit_skill_{r['id']}"):
                        en = st.text_input(t["skill_name"], value=r["name"])
                        ec = st.selectbox(t["control"], ["Touch Control","Classic Control Pad"],
                                          index=0 if r["control"] == "Touch Control" else 1)
                        es = st.text_area(t["steps"], value=r["steps"])
                        ev = st.text_input(t["add_video"], value=r["video_url"])
                        update_skill = st.form_submit_button(t["save_changes"])
                    if update_skill:
                        with db() as con:
                            con.execute("UPDATE skills SET name=?,control=?,steps=?,video_url=? WHERE id=?",
                                        (en.strip(), ec, es.strip(), ev.strip(), r["id"]))
                        st.success(t["saved_ok"]); st.rerun()
                    if st.button(t["delete"], key=f"del_skill_{r['id']}"):
                        with db() as con:
                            con.execute("DELETE FROM skills WHERE id=?", (r["id"],))
                        st.rerun()

st.markdown("---")
st.caption("EF PICK • Community-made eFootball learning helper. Not affiliated with KONAMI.")
