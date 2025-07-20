# MkDocs Offline Documentation Setup Guide

This guide walks you through a full offline-first setup of MkDocs with Material theme, including customization with CSS and project structure for technical documentation.

---

## 🚀 Installation Steps (Ubuntu/Debian)

1. **Install Python, pip, and virtualenv**:

```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv git
```

2. **Create project folder and virtual environment**:

```bash
mkdir ~/my-docs
cd ~/my-docs
python3 -m venv .venv
source .venv/bin/activate
```

3. **Install MkDocs and Material theme**:

```bash
pip install mkdocs-material
```

---

## 📁 Recommended Project Structure

```
my-docs/
├── .venv/                 # Virtual environment (not included in version control)
├── docs/                  # Markdown files (site content)
│   ├── index.md
│   ├── intro.md
│   ├── cloud.md
│   └── ...
├── assets/               # Optional custom assets (CSS, images)
│   ├── css/
│   │   └── custom.css
│   └── images/
│       └── bg.jpg
├── mkdocs.yml            # Site configuration file
```

---

## ⚙️ MkDocs + Material Setup

Edit `mkdocs.yml`:

```yaml
site_name: My Offline Docs

theme:
  name: material
  palette:
    scheme: slate
  font:
    text: Roboto Mono
    code: JetBrains Mono

extra_css:
  - assets/css/custom.css

nav:
  - Home: index.md
  - Introduction: intro.md
  - Projects:
      - Cloud: cloud.md
      - AI: ai.md
      - DevOps: devops.md
```

---

## 🎨 Custom Theme with CSS (Optional)

Place this in `docs/assets/css/custom.css`:

```css
/* Background for homepage */
body[data-md-page="index"] .md-main__inner.md-grid {
    background-image: url("../images/bg.jpg");
    background-size: cover;
    background-position: center;
    background-repeat: no-repeat;
    min-height: 100vh;
}

/* Cool tech font + color scheme */
body {
    font-family: 'Fira Code', monospace;
    color: #40E0D0;
}

/* Transparent background boxes */
.md-content, .md-sidebar, .md-header {
    background-color: transparent !important;
}

.md-header {
    background-color: #012a2a !important;
}
```

---

## 🧪 Preview the Site Offline

Run locally with hot reload:

```bash
mkdocs serve
```

Visit: [http://127.0.0.1:8000](http://127.0.0.1:8000)

---

## 🏗 Build Static Site (for offline use)

```bash
mkdocs build
```

This creates a static site in the `site/` directory. You can open `site/index.html` directly in any browser—**no server needed**.

To copy to another machine:

```bash
tar -czvf my-docs.tar.gz site/
```

---

## ☁️ (Optional) Deploy to GitHub Pages

If you want to deploy later:

1. Make sure the project is a git repo:

```bash
git init
git remote add origin <your-github-url>
```

2. Deploy with built-in command:

```bash
mkdocs gh-deploy
```

---

## ✅ Final Notes
- You do **not** need to be online to preview, build, or browse the final output.
- Store and distribute the `site/` folder to share it.
- Use this guide for reproducible setups on air-gapped or legacy systems.

