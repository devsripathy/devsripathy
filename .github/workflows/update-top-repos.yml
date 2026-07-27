name: Update Top Repositories

on:
  schedule:
    # Run every day at 00:00 UTC
    - cron: '0 0 * * *'
  workflow_dispatch:
    # Allows manual triggering from GitHub UI
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
          python3 -c '
          import os
          import sys
          import urllib.request
          import urllib.parse
          import json
          from datetime import datetime
          import re
          
          # Configuration
          token = os.environ.get("GITHUB_TOKEN", "")
          repo_count = int(os.environ.get("REPO_COUNT", 10))
          min_stars = 25000
          max_repos = min(repo_count, 100)
          
          print(f"📊 Fetching top {max_repos} repositories with >{min_stars:,} stars...")
          
          # Build API request
          params = {
              "q": f"stars:>{min_stars}",
              "sort": "stars",
              "order": "desc",
              "per_page": max_repos
          }
          url = "https://api.github.com/search/repositories?" + urllib.parse.urlencode(params)
          
          req = urllib.request.Request(url)
          if token:
              req.add_header("Authorization", f"token {token}")
          req.add_header("User-Agent", "Python-GitHub-Action")
          req.add_header("Accept", "application/vnd.github.v3+json")
          
          # Make API request with retry logic
          items = []
          total_count = 0
          retry_count = 3
          
          for attempt in range(retry_count):
              try:
                  with urllib.request.urlopen(req, timeout=30) as response:
                      data = json.loads(response.read().decode())
                      items = data.get("items", [])
                      total_count = data.get("total_count", 0)
                      
                      if total_count > 0:
                          print(f"✅ Found {total_count:,} repositories with >{min_stars:,} stars")
                      break
              except urllib.error.HTTPError as e:
                  if e.code == 403 and attempt < retry_count - 1:
                      print(f"⚠️ Rate limit hit, retrying in 5 seconds... (Attempt {attempt + 1}/{retry_count})")
                      import time
                      time.sleep(5)
                  else:
                      print(f"❌ HTTP Error: {e.code} - {e.reason}")
                      if hasattr(e, "read"):
                          try:
                              error_detail = e.read().decode()
                              print(f"   Details: {error_detail}")
                          except:
                              pass
                      sys.exit(1)
              except Exception as e:
                  print(f"❌ Error fetching API: {e}")
                  if attempt == retry_count - 1:
                      sys.exit(1)
                  import time
                  time.sleep(3)
          
          if not items:
              print("❌ No repositories found or API returned empty result")
              sys.exit(1)
          
          # Build markdown table
          print(f"📝 Building markdown table with {len(items)} repositories...")
          
          # Add header
          rows = [
              "| # | Repository | Description | Language | Stars |",
              "|---|------------|-------------|----------|-------|"
          ]
          
          for idx, repo in enumerate(items, 1):
              repo_name = repo["full_name"]
              html_url = repo["html_url"]
              
              # Clean description
              desc = repo.get("description") or "No description provided."
              desc = desc.replace("\n", " ").replace("|", "\\|")
              if len(desc) > 100:
                  desc = desc[:97] + "..."
              
              # Format language
              lang = repo.get("language") or "Multi"
              lang = f"`{lang}`"
              
              # Format stars with emoji and commas
              stars = repo["stargazers_count"]
              stars_formatted = f"⭐ {stars:,}"
              
              # Add extra badges for top repos
              badges = ""
              if idx <= 3:
                  badges = " 🏆"
              elif stars > 100000:
                  badges = " 💎"
              
              rows.append(f"| {idx} | [{repo_name}]({html_url}){badges} | {desc} | {lang} | {stars_formatted} |")
          
          table_content = "\n".join(rows) + "\n"
          
          # Add timestamp footer
          timestamp = datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S UTC")
          footer = f"\n\n*📅 Last updated: {timestamp}*\n"
          table_content += footer
          
          # Read current README
          try:
              with open("README.md", "r", encoding="utf-8") as f:
                  readme = f.read()
          except FileNotFoundError:
              print("❌ README.md not found in repository")
              sys.exit(1)
          
          # Define markers
          start_tag = "<!--START_SECTION:top-repos-->"
          end_tag = "<!--END_SECTION:top-repos-->"
          
          # Check if markers exist
          if start_tag not in readme or end_tag not in readme:
              print("❌ Could not find START_SECTION or END_SECTION markers in README.md")
              print("   Please add these comments to your README:")
              print(f"   {start_tag}")
              print("   (content will be auto-generated here)")
              print(f"   {end_tag}")
              sys.exit(1)
          
          # Update README content
          pattern = re.compile(
              re.escape(start_tag) + r".*?" + re.escape(end_tag),
              re.DOTALL
          )
          
          new_section = f"{start_tag}\n{table_content}{end_tag}"
          new_readme = pattern.sub(new_section, readme)
          
          # Write updated README
          with open("README.md", "w", encoding="utf-8") as f:
              f.write(new_readme)
          
          # Check if changes were made
          if readme != new_readme:
              print("✅ Successfully updated README.md!")
              
              # Print summary using items list directly
              print("\n📊 Top 5 repositories:")
              for i in range(min(5, len(items))):
                  repo_item = items[i]
                  print(f"  {i+1}. {repo_item['full_name']} - ⭐ {repo_item['stargazers_count']:,}")
              
              if len(items) > 5:
                  print(f"  ... and {len(items) - 5} more")
          else:
              print("ℹ️ No changes needed - README is up to date")
          '
          
      - name: Commit and push changes
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          
          # Check if README.md has changes
          if git diff --quiet README.md; then
            echo "ℹ️ No changes to commit"
          else
            git add README.md
            git commit -m "docs: update top repositories [skip ci]"
            git push
            echo "✅ Changes committed and pushed"
          fi
          
      - name: Log completion
        run: |
          echo "✅ Workflow completed successfully"
          echo "📅 $(date)"
