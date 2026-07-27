name: Update Top Repositories

on:
  schedule:
    - cron: '0 0 * * *'
  workflow_dispatch:
    inputs:
      repo_count:
        description: 'Number of repositories to fetch'
        required: false
        default: 10
        type: number

permissions:
  contents: write

jobs:
  update-readme:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Fetch Top Repositories & Update README
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          REPO_COUNT: ${{ inputs.repo_count || 10 }}
        run: |
          cat > /tmp/update_top_repos.py << 'PYEOF'
          import os, sys, json, re, time
          import urllib.request, urllib.parse
          from datetime import datetime

          token      = os.environ.get("GITHUB_TOKEN", "")
          max_repos  = min(int(os.environ.get("REPO_COUNT", 10)), 100)
          min_stars  = 25000

          print(f"📊 Fetching top {max_repos} repos with >{min_stars:,} stars...")

          params = {
              "q": f"stars:>{min_stars}",
              "sort": "stars",
              "order": "desc",
              "per_page": max_repos,
          }
          url = "https://api.github.com/search/repositories?" + urllib.parse.urlencode(params)

          req = urllib.request.Request(url)
          if token:
              req.add_header("Authorization", f"token {token}")
          req.add_header("User-Agent", "github-actions-top-repos")
          req.add_header("Accept", "application/vnd.github.v3+json")

          items = []
          for attempt in range(3):
              try:
                  with urllib.request.urlopen(req, timeout=30) as resp:
                      data = json.loads(resp.read().decode())
                      items = data.get("items", [])
                      total = data.get("total_count", 0)
                      print(f"✅ Found {total:,} matching repositories")
                      break
              except urllib.error.HTTPError as e:
                  if e.code == 403 and attempt < 2:
                      print(f"⚠️ Rate-limited – retry {attempt+1}/3")
                      time.sleep(5)
                  else:
                      print(f"❌ HTTP {e.code}: {e.reason}")
                      sys.exit(1)
              except Exception as e:
                  print(f"❌ {e}")
                  if attempt == 2:
                      sys.exit(1)
                  time.sleep(3)

          if not items:
              print("❌ No repositories returned")
              sys.exit(1)

          # ---------- clean Markdown table ----------
          header = (
              "| # | Repository | Description | Primary Tech | Stars |\n"
              "|:-:|:-----------|:------------|:------------:|------:|"
          )

          rows = []
          for idx, repo in enumerate(items, 1):
              name  = repo["full_name"]
              link  = repo["html_url"]
              desc  = (repo.get("description") or "No description").replace("\n", " ").replace("|", "\\|")
              if len(desc) > 90:
                  desc = desc[:87] + "..."
              lang  = f"`{repo.get('language') or 'Multi'}`"
              stars = f"⭐ {repo['stargazers_count']:,}"
              badge = " 🏆" if idx <= 3 else (" 💎" if repo["stargazers_count"] > 100_000 else "")

              rows.append(
                  f"| {idx} | [{name}]({link}){badge} | {desc} | {lang} | {stars} |"
              )

          table = header + "\n" + "\n".join(rows)
          timestamp = datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S UTC")
          section = f"{table}\n\n*📅 Last updated: {timestamp}*\n"

          # ---------- inject into README ----------
          with open("README.md", "r", encoding="utf-8") as f:
              readme = f.read()

          start, end = "<!--START_SECTION:top-repos-->", "<!--END_SECTION:top-repos-->"
          if start not in readme or end not in readme:
              print("❌ Markers not found in README.md")
              sys.exit(1)

          pattern = re.compile(re.escape(start) + r".*?" + re.escape(end), re.DOTALL)
          new_readme = pattern.sub(f"{start}\n{section}{end}", readme)

          with open("README.md", "w", encoding="utf-8") as f:
              f.write(new_readme)

          print("✅ README.md updated with clean table")
          print("\n📊 Top 5:")
          for i, r in enumerate(items[:5], 1):
              print(f"  {i}. {r['full_name']} – ⭐ {r['stargazers_count']:,}")
          PYEOF

          python3 /tmp/update_top_repos.py

      - name: Commit & push
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          if git diff --quiet README.md; then
            echo "ℹ️ No changes"
          else
            git add README.md
            git commit -m "docs: update top repositories [skip ci]"
            git push
            echo "✅ Pushed clean table"
          fi
