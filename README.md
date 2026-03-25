# Srivatsav Gorti - Portfolio

Personal portfolio and blog of a Staff Software Engineer with 10 years of experience building scalable backend platforms and distributed systems.

## 🚀 Live Demo

[View Live Site](https://srivatsav.github.io/portfolio/)

## 🛠️ Tech Stack

- **Hugo** - Static site generator
- **HTML/CSS/JavaScript** - Custom theme with dark mode
- **GitHub Pages** - Free hosting

## 📝 Adding Blog Posts

1. Create a new Markdown file in `content/blog/`
2. Front matter format:
   ```yaml
   ---
   title: "Post Title"
   date: 2025-03-25
   ---
   ```
3. Write your content in Markdown
4. Commit and push - GitHub Actions will deploy automatically

## 🚀 Local Development

```bash
# Install Hugo
brew install hugo  # macOS
# or
choco install hugo-extended  # Windows

# Serve locally
cd portfolio
hugo server -D
```

Visit `http://localhost:1313`

## 📦 GitHub Pages Setup

1. **Create repository** named `portfolio` (or update baseURL in `hugo.toml`)
2. **Push this code** to GitHub:
   ```bash
   git init
   git add .
   git commit -m "Initial portfolio"
   git branch -M main
   git remote add origin https://github.com/srivatsav/portfolio.git
   git push -u origin main
   ```
3. **Enable GitHub Pages**:
   - Go to repo Settings → Pages
   - Source: Deploy from a branch
   - Branch: `main` / `/(root)`
   - Folder: `(root)` or `/public` (depending on your workflow)
4. **Add GitHub Actions** (optional, for auto-deploy):
   ```bash
   mkdir -p .github/workflows
   ```

## 📁 Project Structure

```
portfolio/
├── content/          # Markdown content
│   ├── _index.md      # Home
│   ├── about.md       # About page
│   ├── experience.md  # Experience timeline
│   ├── tech-stack.md  # Tech skills
│   └── blog/         # Blog posts
├── layouts/          # HTML templates
│   ├── index.html
│   ├── single.html
│   └── partials/
├── static/
│   └── css/
│       └── style.css  # Custom styles
├── hugo.toml        # Site configuration
└── README.md         # This file
```

## 🎨 Customization

- Edit `hugo.toml` for site metadata
- Edit `static/css/style.css` for styling
- Edit layouts in `layouts/` for structure

## 📧 Contact

- Email: vatsav.gs@gmail.com
- LinkedIn: [linkedin.com/in/srivatsav-gorti](https://linkedin.com/in/srivatsav-gorti)
- GitHub: [github.com/srivatsav](https://github.com/srivatsav)

---

Built with Hugo ❤️
