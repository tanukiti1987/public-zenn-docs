---
title: "チームの知で作る AI コードレビュー 🤖"
emoji: "✨"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["Claude", "生成ai"]
published: false
publication_name: "sikmi_tech"
---

[グロービス Advent Calendar 2025](https://qiita.com/advent-calendar/2025/globis)の2日目の記事として寄稿します。

[しくみ製作所](https://sikmi.com/)で働いている [tanukiti1987](https://zenn.dev/tanukiti1987) です。
普段はグロービス様のプロダクト開発（[GLOPLA LMS](https://gce.globis.co.jp/service/training-type/glopla-lms/)）のお手伝いをさせていただいています。

グロービス様のアドベントカレンダーにしくみ製作所の zenn をリンクするにあたり、快く受け入れていただきありがとうございます 🙇‍♂

今回は、普段のプロジェクトの中で AIコードレビューを導入しましたので、そのお話をさせていただきます。

----

## AI によるコードレビューについて

AI によるコードレビューについては、SaaS である [CodeRabbit](https://www.coderabbit.ai/) や [greptile](https://www.greptile.com/) であったり、 ClaudeCode, Codex で用意されている `/review` などがあります。

PullRequest や既存のコードに対して、AIが様々な視点でコードレビューをしてくれるものです。

ものにより様々ですが、タイポなどの些末な指摘から、セキュリティやパフォーマンスの問題についてもフィードバックをしてくれることがあり、人間がコードレビューをしていても気が付かなかったポイントにもコメントが入ることから、有用に感じている方も多いのではないでしょうか。

チームの人にコードレビューをして貰う前に、一度は AI によるコードレビューを挟むことは、人間のレビュー負荷を下げるためにも、有用なアクションになると思っています。

SaaS の方は業務で導入するまでには幾つかハードルがあるかと思いますが、 Claude Code や Codex の `/review` コマンドは、CLI ツールを使える方ならすぐに試すことができると思います。

## AI コードレビューの課題感

先程書いたように、人間のレビューでは見逃したポイントを指摘してくれることがありますが、一方であまり重要ではない指摘、プロジェクトの指針にはそぐわない指摘など、レビューの観点としては誤った指摘をしてくることもあります。

また、説明が冗長で読むのが大変な指摘などもあり、イマイチに感じるシーンも少なくありません。

## プロジェクトにフィットしたレビューを目指して

私が参画している [GLOPLA LMS](https://gce.globis.co.jp/service/training-type/glopla-lms/) のプロジェクトでは、先程挙げたようなデメリットを解消すべく、カスタムレビューコマンドを作り、活用しています。

### カスタムレビューコマンドの特徴

- チームの人間が PR にコメントした内容を元にしたレビューをしてくれる
  - さらに、過去に本番環境やステージング環境で発生した不具合から学んでレビューをしてくれる
- 簡潔な指摘事項と、詳細な指摘を PR 上で見やすく出力する

レビュー結果が見やすいかどうかも重要ではありますが、このコマンドの最大のメリットは、一般的なコードの良し悪しを評価するAIコードレビューではなく、これまでチームが蓄積した知見を元にレビューをしてくれることになります。

とりわけ、本番やステージング環境に適用した意図せぬ不具合へのパッチPRは、一度はチームのレビューを通過してしまったコードですので、 weak point とも捉えることができます。
そのポイントを抑えたレビューをAIが自動でしてくれることにとても価値を感じています。

### レビュー結果の例

これは、私たちのプロジェクト向けのPRにつけたカスタムレビューコマンド出力の一例です

![](/images/ai-review-with-claude-code/sample.jpg)

(diff 大きすぎる部分、スミマセン。見逃してください)

- 「詳細を表示」をクリックすることで、指摘の内容をより詳細に見ることができます
  - 冗長なコメントをトグルに閉じ込めることで可読性を上げています
- AI も PRを承認したり、承認しなかったりをしてきます

## カスタムコマンドの作り方

コマンドを作るに当たっては主に5つのポイントがあります。

- 開発ブランチ向けの PR から人間のつけたコメントを収集する
- ステージング/本番環境向けの PR を収集する
- 収集したコメント等をレビュー観点で抽象化する
- 抽象化したレビュー観点で実際にレビューをする
- レビューコメントを定期的に収集する

### ディレクトリ/ファイル構造

カスタムコマンド及び、レビューする際に必要な観点を作成/更新するスキルの2段構えにしています。

```
.claude/commands/backend-review.md
.claude/skills/update-backend-reviewer/
  ├── artifacts/
  │   ├── backend_reviewer.md
  │   ├── patch_release_review_points.md
  │   └── review_output_format.md
  ├── collect_patch_prs.sh
  ├── collect_pr_review_comments.sh
  ├── generate_backend_reviewer_guide.sh
  ├── generate_patch_review_guide.sh
  ├── README_PR_Comment_Collector.md
  └── SKILL.md
```


### 開発ブランチ向けの PR から人間のつけたコメントを収集する

私たちのPJでは過去300件のPRを対象にコメントを収集しています。
コメントの収集 → 観点の抽象化までを一息に AI に実行させるのはトークンを大量に消費しすぎますし、途中で問題があったときにリカバリが大変になってしまうので、可能な限りスクリプトを用意していきます。

このコメント収集においても、 `gh` コマンドを使い、人間のつけたコメントを収集し、 text ファイルとして吐き出すスクリプト `collect_pr_review_comments.sh` を私たちのPJでは用意しています。

:::details collect_pr_review_comments.sh
```sh
#!/bin/bash

# PR Review Comments Collector for xxx repository
# Collects human-written review comments from closed backend-related PRs
# Usage: ./tmp/collect_pr_review_comments.sh [number_of_prs] [output_file]

set -e

# Default values
DEFAULT_PR_COUNT=300
DEFAULT_OUTPUT_FILE="./.claude/skills/update-backend-reviewer/artifacts/backend_pr_comments.txt"
REPO="zzz/xxx"
TEMP_DIR="./tmp"

# Parse arguments
PR_COUNT=${1:-$DEFAULT_PR_COUNT}
OUTPUT_FILE=${2:-$DEFAULT_OUTPUT_FILE}

# Color codes for output
RED='\033[0;31m'
GREEN='\033[0;32m'
BLUE='\033[0;34m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${BLUE}🔍 Collecting review comments from ${PR_COUNT} closed PRs in ${REPO}...${NC}"

# Check prerequisites
echo -e "${BLUE}📋 Checking prerequisites...${NC}"

if ! command -v gh &> /dev/null; then
    echo -e "${RED}❌ GitHub CLI (gh) is not installed. Please install it first.${NC}"
    echo -e "${YELLOW}   Install: brew install gh${NC}"
    exit 1
fi

if ! command -v jq &> /dev/null; then
    echo -e "${RED}❌ jq is not installed. Please install it first.${NC}"
    echo -e "${YELLOW}   Install: brew install jq${NC}"
    exit 1
fi

# Check GitHub authentication
if ! gh auth status &> /dev/null; then
    echo -e "${RED}❌ GitHub CLI is not authenticated. Please run 'gh auth login' first.${NC}"
    exit 1
fi

echo -e "${GREEN}✅ Prerequisites check passed${NC}"

# Create temp and artifacts directories if they don't exist
mkdir -p "$TEMP_DIR"
mkdir -p "$(dirname "$OUTPUT_FILE")"

# Step 1: Get closed PRs
echo -e "${BLUE}📥 Fetching latest ${PR_COUNT} closed pull requests...${NC}"
gh api repos/$REPO/pulls \
  --method GET \
  --field state=closed \
  --field per_page=$PR_COUNT \
  --field sort=updated \
  --field direction=desc \
  --jq '.[] | "\(.number)|\(.title)|\(.state)|\(.head.ref)|\(.html_url)"' > "$TEMP_DIR/pr_list_temp.txt"

echo -e "${GREEN}✅ Retrieved $(wc -l < "$TEMP_DIR/pr_list_temp.txt") closed PRs${NC}"

# Step 2: Identify backend-related PRs
echo -e "${BLUE}🔧 Identifying backend-related PRs...${NC}"
backend_prs=()
pr_info_map=""

while IFS='|' read -r pr_number pr_title pr_state pr_branch pr_url; do
  echo -e "  Checking PR #$pr_number for backend files..."

  # Check if PR has backend-related files
  backend_files=$(gh api repos/$REPO/pulls/$pr_number/files \
    --jq '.[] | select(.filename | test("\\.(rb|rake|yml|yaml)$|^Gemfile$|^Rakefile$|^config/|^db/|^app/|^spec/|^lib/")) | .filename' \
    2>/dev/null || echo "")

  if [ -n "$backend_files" ]; then
    backend_prs+=($pr_number)
    # Store PR info for later use
    echo "$pr_number|$pr_title|$pr_state|$pr_url|$backend_files" >> "$TEMP_DIR/backend_pr_info.txt"
    echo -e "    ${GREEN}✅ Backend PR: #$pr_number - $pr_title${NC}"
  else
    echo -e "    ${YELLOW}❌ Frontend/Other PR: #$pr_number${NC}"
  fi
done < "$TEMP_DIR/pr_list_temp.txt"

echo -e "${GREEN}🎯 Found ${#backend_prs[@]} backend-related PRs: ${backend_prs[*]}${NC}"

if [ ${#backend_prs[@]} -eq 0 ]; then
  echo -e "${RED}❌ No backend PRs found in the latest $PR_COUNT closed PRs${NC}"
  exit 1
fi

# Step 3: Collect comments from backend PRs
echo -e "${BLUE}💬 Collecting review comments from backend PRs...${NC}"

# Initialize output file
cat > "$OUTPUT_FILE" << EOF
# バックエンド関連Pull Requestレビューコメント集

## 収集設定
- **対象リポジトリ**: $REPO
- **分析対象**: 過去${PR_COUNT}件の終了済みPull Request
- **バックエンドPR数**: ${#backend_prs[@]}件
- **収集日時**: $(date '+%Y年%m月%d日 %H:%M:%S')
- **除外対象**: Botコメント（github-actions, reg-suit, dependabot等）およびAI生成コメント（レビューサマリー、バックエンドコードレビュー）

## 収集されたコメント

EOF

total_comments=0
total_human_comments=0

for pr_number in "${backend_prs[@]}"; do
  echo -e "  Processing PR #$pr_number..."

  # Get PR info from stored data
  pr_info=$(grep "^$pr_number|" "$TEMP_DIR/backend_pr_info.txt" || echo "")
  if [ -n "$pr_info" ]; then
    IFS='|' read -r _ pr_title pr_state pr_url backend_files_list <<< "$pr_info"
  else
    # Fallback: get info from API
    pr_details=$(gh api repos/$REPO/pulls/$pr_number)
    pr_title=$(echo "$pr_details" | jq -r '.title')
    pr_state=$(echo "$pr_details" | jq -r '.state')
    pr_url=$(echo "$pr_details" | jq -r '.html_url')
    backend_files_list="N/A"
  fi

  # Get review comments (code-level comments)
  echo "    Getting review comments..."
  review_comments=$(gh api repos/$REPO/pulls/$pr_number/comments 2>/dev/null || echo "[]")

  # Get issue comments (general discussion comments)
  echo "    Getting general comments..."
  issue_comments=$(gh api repos/$REPO/issues/$pr_number/comments 2>/dev/null || echo "[]")

  # Process review comments (exclude bots and AI-generated comments)
  human_review_comments=$(echo "$review_comments" | jq -r '
    .[] |
    select(.user.login | test("\\[bot\\]$") | not) |
    select(.body | length > 0) |
    select(.body | test("レビューサマリー|バックエンドコードレビュー") | not) |
    "**@" + .user.login + "** (" + (.created_at | split("T")[0]) + "):\n" + .body + "\n"
  ' 2>/dev/null || echo "")

  # Process issue comments (exclude bots and AI-generated comments)
  human_issue_comments=$(echo "$issue_comments" | jq -r '
    .[] |
    select(.user.login | test("\\[bot\\]$") | not) |
    select(.body | length > 0) |
    select(.body | test("レビューサマリー|バックエンドコードレビュー") | not) |
    "**@" + .user.login + "** (" + (.created_at | split("T")[0]) + "):\n" + .body + "\n"
  ' 2>/dev/null || echo "")

  # Count comments (excluding bots and AI-generated comments)
  review_count=$(echo "$review_comments" | jq '. | length' 2>/dev/null || echo "0")
  issue_count=$(echo "$issue_comments" | jq '. | length' 2>/dev/null || echo "0")

  human_review_count=$(echo "$review_comments" | jq '[.[] | select(.user.login | test("\\[bot\\]$") | not) | select(.body | length > 0) | select(.body | test("レビューサマリー|バックエンドコードレビュー") | not)] | length' 2>/dev/null || echo "0")
  human_issue_count=$(echo "$issue_comments" | jq '[.[] | select(.user.login | test("\\[bot\\]$") | not) | select(.body | length > 0) | select(.body | test("レビューサマリー|バックエンドコードレビュー") | not)] | length' 2>/dev/null || echo "0")

  pr_total_comments=$((review_count + issue_count))
  pr_human_comments=$((human_review_count + human_issue_count))

  total_comments=$((total_comments + pr_total_comments))
  total_human_comments=$((total_human_comments + pr_human_comments))

  # Skip PRs with no human comments
  if [ $pr_human_comments -eq 0 ]; then
    echo -e "    ${YELLOW}⏭️  Skipped PR #$pr_number (no human comments)${NC}"
    continue
  fi

  # Write PR header only if there are human comments
  echo "" >> "$OUTPUT_FILE"
  echo "### PR #$pr_number: $pr_title" >> "$OUTPUT_FILE"
  echo "" >> "$OUTPUT_FILE"
  echo "- **状態**: $pr_state" >> "$OUTPUT_FILE"
  echo "- **URL**: $pr_url" >> "$OUTPUT_FILE"
  echo "- **変更ファイル**: $(echo "$backend_files_list" | tr '\n' ', ' | sed 's/, $//')" >> "$OUTPUT_FILE"
  echo "" >> "$OUTPUT_FILE"
  echo "**コメント統計**: 人間によるコメント ${pr_human_comments}件" >> "$OUTPUT_FILE"
  echo "" >> "$OUTPUT_FILE"

  if [ -n "$human_review_comments" ]; then
    echo "#### レビューコメント" >> "$OUTPUT_FILE"
    echo "" >> "$OUTPUT_FILE"
    echo "$human_review_comments" >> "$OUTPUT_FILE"
  fi

  if [ -n "$human_issue_comments" ]; then
    echo "#### 一般コメント" >> "$OUTPUT_FILE"
    echo "" >> "$OUTPUT_FILE"
    echo "$human_issue_comments" >> "$OUTPUT_FILE"
  fi

  echo "---" >> "$OUTPUT_FILE"

  echo -e "    ${GREEN}✅ Collected $pr_human_comments human comments from PR #$pr_number${NC}"
done

# Step 4: Add summary statistics
cat >> "$OUTPUT_FILE" << EOF

## 収集統計

- **分析対象PR総数**: $PR_COUNT件
- **バックエンドPR数**: ${#backend_prs[@]}件
- **総コメント数**: ${total_comments}件
- **人間によるコメント数**: ${total_human_comments}件
- **Bot コメント数**: $((total_comments - total_human_comments))件
- **平均人間コメント数/バックエンドPR**: $((total_human_comments / ${#backend_prs[@]}))件

## 対象Pull Request一覧

EOF

for pr_number in "${backend_prs[@]}"; do
  pr_info=$(grep "^$pr_number|" "$TEMP_DIR/backend_pr_info.txt")
  IFS='|' read -r _ pr_title pr_state pr_url _ <<< "$pr_info"
  echo "- **PR #$pr_number**: $pr_title - [リンク]($pr_url)" >> "$OUTPUT_FILE"
done

cat >> "$OUTPUT_FILE" << EOF

---

*このレポートは collect_pr_review_comments.sh により自動生成されました*
*生成日時: $(date '+%Y年%m月%d日 %H:%M:%S')*

EOF

# Clean up temporary files
rm -f "$TEMP_DIR/pr_list_temp.txt" "$TEMP_DIR/backend_pr_info.txt"

echo -e "${GREEN}✨ Collection completed successfully!${NC}"
echo -e "${BLUE}📊 Final Statistics:${NC}"
echo -e "   - Total PRs analyzed: $PR_COUNT"
echo -e "   - Backend PRs found: ${#backend_prs[@]}"
echo -e "   - Total comments: $total_comments"
echo -e "   - Human comments collected: $total_human_comments"
echo -e "   - Bot comments filtered out: $((total_comments - total_human_comments))"
echo -e "${GREEN}📄 Results saved to: $OUTPUT_FILE${NC}"
echo ""
echo -e "${YELLOW}🎯 Next steps:${NC}"
echo -e "   1. Review the collected comments in $OUTPUT_FILE"
echo -e "   2. Use Claude Code to analyze the review patterns:"
echo -e "      claude 'この人間によるレビューコメントを分析して、8つの観点（コード品質、セキュリティ、パフォーマンス、テスト、設計、運用保守、データベース、その他）に分類し、各観点の特徴的な指摘パターンを抽出してください。'"
echo ""
```
:::

説明を端折っていましたが、今回のレビューコマンドはリポジトリの中のバックエンド部分をレビューするコマンドとしています。
バックエンド（Rails)とフロントエンド(React)ではレビュー観点も異なりますし、そのナレッジの活かし方も変わると判断し、レビューコメントの抽出はバックエンド関連のPRのみを取り込むようにしたり、CIでつけられるボットからのコメント、AIコードレビューでつけられるコメントを取り除くような工夫を加えています。

例として、シェルスクリプトの例を貼りましたが、このスクリプトも Claude に生成してもらったものです。
もし自プロジェクトで同じことをやる場合には、自プロジェクトにあったコメント収集方法を検討し、スクリプトを作ることをおすすめします。

### ステージング/本番環境向けの PR を収集する

先程のスクリプトと似たようなスクリプトになりますが、こちらはレビューコメントだけでなく、作成された PR 自体がチームのコードレビューをすり抜けてしまった不具合に対する修正になります。

タイトルやPRの説明も含め、レビューの観点として収集します。

私たちのPJでは、このようなPRには `[patch]` という接頭辞のついたPRを作ることになっており、比較的収集が容易でした。
使っているスクリプトも示しておきます。

:::details collect_patch_prs.sh
```sh
#!/bin/bash

# Patch PR Collector for xxx repository
# Collects [patch] prefixed PRs since the last analyzed PR in patch_release_review_points.md
# Usage: ./collect_patch_prs.sh [number_of_prs]

set -e

# Default values
DEFAULT_PR_COUNT=500
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" &> /dev/null && pwd)"
REVIEW_POINTS_FILE="$SCRIPT_DIR/artifacts/patch_release_review_points.md"
OUTPUT_FILE="$SCRIPT_DIR/artifacts/backend_patch_prs.txt"
REPO="zzz/xxx"
TEMP_DIR="./tmp"

# Parse arguments
PR_COUNT=${1:-$DEFAULT_PR_COUNT}

# Color codes for output
RED='\033[0;31m'
GREEN='\033[0;32m'
BLUE='\033[0;34m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${BLUE}🔍 Collecting [patch] PRs from ${REPO}...${NC}"

# Check prerequisites
echo -e "${BLUE}📋 Checking prerequisites...${NC}"

if ! command -v gh &> /dev/null; then
    echo -e "${RED}❌ GitHub CLI (gh) is not installed. Please install it first.${NC}"
    echo -e "${YELLOW}   Install: brew install gh${NC}"
    exit 1
fi

if ! command -v jq &> /dev/null; then
    echo -e "${RED}❌ jq is not installed. Please install it first.${NC}"
    echo -e "${YELLOW}   Install: brew install jq${NC}"
    exit 1
fi

# Check GitHub authentication
if ! gh auth status &> /dev/null; then
    echo -e "${RED}❌ GitHub CLI is not authenticated. Please run 'gh auth login' first.${NC}"
    exit 1
fi

echo -e "${GREEN}✅ Prerequisites check passed${NC}"

# Create temp and artifacts directories if they don't exist
mkdir -p "$TEMP_DIR"
mkdir -p "$(dirname "$OUTPUT_FILE")"

# Step 1: Find the highest PR number in patch_release_review_points.md
echo -e "${BLUE}📖 Reading last analyzed PR number from $REVIEW_POINTS_FILE...${NC}"

if [[ ! -f "$REVIEW_POINTS_FILE" ]]; then
    echo -e "${YELLOW}⚠️  Review points file not found. Will collect all [patch] PRs.${NC}"
    LAST_PR_NUMBER=0
else
    # Extract all PR numbers and find the maximum
    LAST_PR_NUMBER=$(grep -oE 'PR #[0-9]+' "$REVIEW_POINTS_FILE" | grep -oE '[0-9]+' | sort -n | tail -1 || echo "0")
    if [[ -z "$LAST_PR_NUMBER" ]]; then
        LAST_PR_NUMBER=0
    fi
fi

echo -e "${GREEN}✅ Last analyzed PR number: #$LAST_PR_NUMBER${NC}"

# Step 2: Get closed PRs with [patch] prefix
echo -e "${BLUE}📥 Fetching closed [patch] PRs newer than #$LAST_PR_NUMBER...${NC}"

# Use search API to find [patch] PRs
gh api -X GET search/issues \
  -f q="repo:$REPO is:pr is:closed in:title [patch]" \
  -f sort=created \
  -f order=desc \
  -f per_page=$PR_COUNT \
  --jq '.items[] | "\(.number)|\(.title)|\(.html_url)"' > "$TEMP_DIR/patch_pr_list_temp.txt"

# Filter PRs newer than LAST_PR_NUMBER
patch_prs=()
while IFS='|' read -r pr_number pr_title pr_url; do
  if [[ $pr_number -gt $LAST_PR_NUMBER ]]; then
    # Verify it starts with [patch] (case insensitive)
    if echo "$pr_title" | grep -iqE '^\[patch\]'; then
      patch_prs+=($pr_number)
      echo -e "  ${GREEN}✅ Found: #$pr_number - $pr_title${NC}"
    fi
  fi
done < "$TEMP_DIR/patch_pr_list_temp.txt"

echo -e "${GREEN}🎯 Found ${#patch_prs[@]} new [patch] PRs${NC}"

if [ ${#patch_prs[@]} -eq 0 ]; then
    echo -e "${YELLOW}⚠️  No new [patch] PRs found since PR #$LAST_PR_NUMBER${NC}"
    echo "# パッチリリースPR情報" > "$OUTPUT_FILE"
    echo "" >> "$OUTPUT_FILE"
    echo "新しいパッチPRは見つかりませんでした。" >> "$OUTPUT_FILE"
    echo "最後に分析されたPR: #$LAST_PR_NUMBER" >> "$OUTPUT_FILE"
    echo "収集日時: $(date '+%Y年%m月%d日 %H:%M:%S')" >> "$OUTPUT_FILE"
    rm -f "$TEMP_DIR/patch_pr_list_temp.txt"
    exit 0
fi

# Step 3: Collect detailed information from each patch PR
echo -e "${BLUE}💬 Collecting details from [patch] PRs...${NC}"

# Initialize output file
cat > "$OUTPUT_FILE" << EOF
# パッチリリースPR情報

## 収集設定
- **対象リポジトリ**: $REPO
- **分析対象**: PR #$LAST_PR_NUMBER 以降の [patch] PR
- **発見したPR数**: ${#patch_prs[@]}件
- **収集日時**: $(date '+%Y年%m月%d日 %H:%M:%S')

---

## 収集されたパッチPR

EOF

for pr_number in "${patch_prs[@]}"; do
  echo -e "  Processing PR #$pr_number..."

  # Get PR details
  pr_details=$(gh api repos/$REPO/pulls/$pr_number 2>/dev/null || echo "{}")

  pr_title=$(echo "$pr_details" | jq -r '.title // "N/A"')
  pr_body=$(echo "$pr_details" | jq -r '.body // ""')
  pr_url=$(echo "$pr_details" | jq -r '.html_url // "N/A"')
  pr_created_at=$(echo "$pr_details" | jq -r '.created_at // "N/A"' | cut -d'T' -f1)
  pr_merged_at=$(echo "$pr_details" | jq -r '.merged_at // "N/A"' | cut -d'T' -f1)

  # Get review comments
  review_comments=$(gh api repos/$REPO/pulls/$pr_number/comments 2>/dev/null || echo "[]")
  human_review_comments=$(echo "$review_comments" | jq -r '
    .[] |
    select(.user.login | test("\\[bot\\]$") | not) |
    select(.body | length > 0) |
    select(.body | test("レビューサマリー|バックエンドコードレビュー") | not) |
    "  - **@" + .user.login + "**: " + (.body | gsub("\n"; " ") | .[0:200]) + (if (.body | length) > 200 then "..." else "" end)
  ' 2>/dev/null || echo "")

  # Get issue comments
  issue_comments=$(gh api repos/$REPO/issues/$pr_number/comments 2>/dev/null || echo "[]")
  human_issue_comments=$(echo "$issue_comments" | jq -r '
    .[] |
    select(.user.login | test("\\[bot\\]$") | not) |
    select(.body | length > 0) |
    select(.body | test("レビューサマリー|バックエンドコードレビュー") | not) |
    "  - **@" + .user.login + "**: " + (.body | gsub("\n"; " ") | .[0:200]) + (if (.body | length) > 200 then "..." else "" end)
  ' 2>/dev/null || echo "")

  # Get changed files
  changed_files=$(gh api repos/$REPO/pulls/$pr_number/files --jq '.[].filename' 2>/dev/null | head -10 | tr '\n' ', ' | sed 's/, $//')

  # Write to output file
  cat >> "$OUTPUT_FILE" << EOF

### PR #$pr_number: $pr_title

- **URL**: $pr_url
- **作成日**: $pr_created_at
- **マージ日**: $pr_merged_at
- **変更ファイル**: $changed_files

**説明**:
$pr_body

EOF

  if [ -n "$human_review_comments" ] || [ -n "$human_issue_comments" ]; then
    echo "**レビューコメント**:" >> "$OUTPUT_FILE"
    if [ -n "$human_review_comments" ]; then
      echo "$human_review_comments" >> "$OUTPUT_FILE"
    fi
    if [ -n "$human_issue_comments" ]; then
      echo "$human_issue_comments" >> "$OUTPUT_FILE"
    fi
    echo "" >> "$OUTPUT_FILE"
  fi

  echo "---" >> "$OUTPUT_FILE"

  echo -e "    ${GREEN}✅ Collected PR #$pr_number${NC}"
done

# Add summary
cat >> "$OUTPUT_FILE" << EOF

## 収集統計

- **新規パッチPR数**: ${#patch_prs[@]}件
- **前回の最終PR番号**: #$LAST_PR_NUMBER
- **今回の最新PR番号**: #${patch_prs[0]}

---

*このレポートは collect_patch_prs.sh により自動生成されました*
*生成日時: $(date '+%Y年%m月%d日 %H:%M:%S')*

EOF

# Clean up temporary files
rm -f "$TEMP_DIR/patch_pr_list_temp.txt"

echo -e "${GREEN}✨ Collection completed successfully!${NC}"
echo -e "${BLUE}📊 Final Statistics:${NC}"
echo -e "   - New [patch] PRs found: ${#patch_prs[@]}"
echo -e "   - Last analyzed PR: #$LAST_PR_NUMBER"
echo -e "${GREEN}📄 Results saved to: $OUTPUT_FILE${NC}"
echo ""
echo -e "${YELLOW}🎯 Next steps:${NC}"
echo -e "   1. Review the collected patch PRs in $OUTPUT_FILE"
echo -e "   2. Run generate_patch_review_guide.sh to update patch_release_review_points.md"
echo ""
```
:::

### 収集したコメント等をレビュー観点で抽象化する

これまでに集めたコメントやPR情報を元に、レビュー観点を作っていきます。

2つのテキスト情報と、レビューの骨組みを Claude に渡し、自分たち専用の観点を構築します。

シェルスクリプトでは `claude -p --dangerously-skip-permissions` コマンドを使って、claude にレビュー観点のマークダウンを作ってもらうというシンプルな作りにしています。

:::details generate_backend_reviewer_guide.sh
```sh
#!/bin/bash

# Backend Reviewer Guide Generation Script
# このスクリプトは backend_pr_comments.txt を分析して backend_reviewer.md を生成します

set -e

# スクリプトディレクトリの取得
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" &> /dev/null && pwd)"
PROJECT_ROOT="$(cd "$SCRIPT_DIR/../../.." &> /dev/null && pwd)"

# 入力ファイルと出力ファイルのパス
INPUT_FILE="$SCRIPT_DIR/artifacts/backend_pr_comments.txt"
OUTPUT_FILE="$SCRIPT_DIR/artifacts/backend_reviewer.md"

echo "📊 Backend Reviewer Guide Generation"
echo "=================================="
echo "Input file: $INPUT_FILE"
echo "Output file: $OUTPUT_FILE"
echo ""

# 入力ファイルの存在確認
if [[ ! -f "$INPUT_FILE" ]]; then
    echo "❌ Error: Input file not found: $INPUT_FILE"
    echo "Please run collect_pr_review_comments.sh first to generate PR comments."
    exit 1
fi

# Artifactsディレクトリの作成
mkdir -p "$SCRIPT_DIR/artifacts"

echo "🤖 Generating backend reviewer guide using Claude..."

# プロジェクトルートに移動してClaudeコマンドを実行
cd "$PROJECT_ROOT"

claude -p --dangerously-skip-permissions '.claude/skills/update-backend-reviewer/artifacts/backend_pr_comments.txt に示す PullRequest のコメントを分析し、8つの観点（コード品質、セキュリティ、パフォーマンス、テスト、設計、運用保守、データベース、その他）に分類し、各観点の特徴的な指摘パターンを抽出してください。 具体的なコード例は最低限に抑え、レビューする観点を列挙することを重視してください。そして、バックエンドのレビュー観点を示す reviewer の心得である @.claude/skills/update-backend-reviewer/artifacts/backend_reviewer.md を作成してください。これを元に、後続で作られる新しい PullRequest のレビューをあなたにしてもらいます。 think hard'

# 出力ファイルの確認
if [[ -f "$OUTPUT_FILE" ]]; then
    echo "✅ Backend reviewer guide successfully generated!"
    echo "📄 File: $OUTPUT_FILE"
    echo ""
    echo "📋 Guide contents summary:"
    echo "$(head -20 "$OUTPUT_FILE" | grep -E '^#|^##' || true)"
    echo ""
    echo "🔍 Use this guide for reviewing future backend Pull Requests."
else
    echo "❌ Error: Failed to generate backend reviewer guide"
    exit 1
fi

echo ""
echo "✨ Done! You can now use the generated guide for PR reviews."
```
:::

:::details generate_patch_review_guide.sh
```sh
#!/bin/bash

# Patch Review Guide Generation Script
# このスクリプトは backend_patch_prs.txt を分析して patch_release_review_points.md を更新します
# パッチリリースはステージングリリース後に発覚した不具合のため、より重要な観点として扱います

set -e

# スクリプトディレクトリの取得
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" &> /dev/null && pwd)"
PROJECT_ROOT="$(cd "$SCRIPT_DIR/../../.." &> /dev/null && pwd)"

# 入力ファイルと出力ファイルのパス
INPUT_FILE="$SCRIPT_DIR/artifacts/backend_patch_prs.txt"
OUTPUT_FILE="$SCRIPT_DIR/artifacts/patch_release_review_points.md"

echo "📊 Patch Review Guide Generation"
echo "=================================="
echo "Input file: $INPUT_FILE"
echo "Output file: $OUTPUT_FILE"
echo ""

# 入力ファイルの存在確認
if [[ ! -f "$INPUT_FILE" ]]; then
    echo "❌ Error: Input file not found: $INPUT_FILE"
    echo "Please run collect_patch_prs.sh first to collect patch PRs."
    exit 1
fi

# 新しいパッチPRがあるか確認
if grep -q "新しいパッチPRは見つかりませんでした" "$INPUT_FILE"; then
    echo "ℹ️  No new patch PRs to analyze."
    echo "The patch review guide is already up to date."
    exit 0
fi

# Artifactsディレクトリの作成
mkdir -p "$SCRIPT_DIR/artifacts"

echo "🤖 Updating patch review guide using Claude..."
echo ""
echo "⚠️  パッチリリースは本番環境への緊急修正です。"
echo "   レビュー観点として最も重要度が高い項目として分析します。"
echo ""

# プロジェクトルートに移動してClaudeコマンドを実行
cd "$PROJECT_ROOT"

claude -p --dangerously-skip-permissions "
あなたはバックエンドコードレビューの専門家です。

## 背景
パッチリリース（[patch]タグ付きPR）は、ステージングリリース後に発覚した本番環境の不具合修正です。
これらは通常のレビューで見逃された問題であり、今後のレビューで最も注意すべき観点となります。

## タスク
1. @.claude/skills/update-backend-reviewer/artifacts/backend_patch_prs.txt に含まれる新しいパッチPRを分析してください
2. 各パッチPRから以下を抽出してください:
   - 不具合の根本原因（なぜ見逃されたか）
   - 影響範囲
   - 防止のためのレビュー観点
3. @.claude/skills/update-backend-reviewer/artifacts/patch_release_review_points.md を更新してください:
   - 既存の問題パターンに該当する場合は、具体例として追加
   - 新しいパターンの場合は、新しいセクションを追加
   - PR番号と簡潔な説明を含める

## 出力形式
patch_release_review_points.md は以下の構造を維持してください:
- はじめに（目的と重要性の説明）
- 主要な問題パターンと対策（カテゴリごとに整理）
- レビュー実施手順
- まとめ

## 重要な注意点
- 具体的なコード例は最小限に抑え、レビュー観点のチェックリストを重視してください
- 新しいPR番号は適切なカテゴリに追加してください
- 文書全体の一貫性を保ってください

think hard
"

# 出力ファイルの確認
if [[ -f "$OUTPUT_FILE" ]]; then
    echo ""
    echo "✅ Patch review guide successfully updated!"
    echo "📄 File: $OUTPUT_FILE"
    echo ""
    echo "📋 Updated sections:"
    echo "$(grep -E '^### [0-9]+\.' "$OUTPUT_FILE" || true)"
    echo ""
    echo "🔍 Use this guide to prevent future patch releases."
else
    echo "❌ Error: Failed to update patch review guide"
    exit 1
fi

echo ""
echo "✨ Done! The patch review guide has been updated with the latest findings."
```
:::

なお、 `generate_backend_reviewer_guide.sh` は常にレビュー指針をまっさらな状態から作り直し。 `generate_patch_review_guide.sh` は、件数自体そこまで多くないので、元々のレビュー指針に継ぎ足しでレビュー観点を追加しています。

### 抽象化したレビュー観点で実際にレビューをする

ここまでで用意できたレビュー観点を元に、PRにレビューしてもらうためのカスタムコマンドを配置します。

:::details backend-review.md
```markdown
# バックエンドレビューコマンド

## 説明

xxx プロジェクトのバックエンド Pull Request レビューを実施するカスタムコマンドです。過去のPRコメント分析に基づく体系的なレビューを行います。

## 使用方法

バックエンドのPRファイルやコード変更に対して、統一された観点からレビューを実施します。

## 指示

あなたはxxx プロジェクトのバックエンドコードレビュアーです。`.claude/skills/update-backend-reviewer/artifacts/backend_reviewer.md`に記載されたガイドラインに従って、提供されたコード変更をレビューしてください。

また、レビュー観点として `.claude/skills/update-backend-reviewer/artifacts/patch_release_review_points.md` も重要です。
過去にチームが行った不具合修正の観点も踏まえ、レビューを行ってください。

### レビュー手順

1. まず`.claude/skills/update-backend-reviewer/artifacts/backend_reviewer.md`を読み込んでレビューガイドラインを確認する
2. `.claude/skills/update-backend-reviewer/artifacts/patch_release_review_points.md` を確認し、追加のレビュー観点も確認する
3. `gh`コマンドを使用してPull Requestの詳細情報（差分、説明、関連する議論）にアクセスし、コンテキストを把握する
4. PR の差分およびコードベースを探索しながら、ガイドラインに記載された8つの観点でコードを**批判的かつ包括的に**レビューする。
   - コード品質 (Code Quality)
   - セキュリティ (Security)
   - パフォーマンス (Performance)
   - テスト (Testing)
   - 設計 (Design)
   - 運用保守 (Operations/Maintenance)
   - データベース (Database)
   - その他 (Others)
5. 各観点のチェックポイントと心得に基づいて**妥協のない厳格なレビュー**を実施
6. レビューバッジを適切に使用してコメントの意図を明確化
7. **潜在的な問題や改善点を積極的に特定し、コードの品質向上に重点を置く**

### 技術スタック情報

**xxx プロジェクト:**
- **バックエンド**: Ruby on Rails 8.0, Ruby 3.4.7
- **データベース**: MySQL 8.0.28 with Trilogy driver
- **API**: GraphQL
- **バックグラウンドジョブ**: Sidekiq + Redis

### レビュー出力形式

**重要**: レビュー出力は `.claude/skills/update-backend-reviewer/artifacts/review_output_format.md` に定義された統一フォーマットに**必ず従ってください**。

#### 必須事項

- `.claude/skills/update-backend-reviewer/artifacts/review_output_format.md` を読み込み、フォーマット仕様を確認する
- **shields.io バッジシステムを使用する**
  - `review_output_format.md` を参照
- **すべての指摘の見出しの末尾に視点タグバッジを必ず付ける**（shields.io バッジを使用、色は silver で統一）
  - 例: `### 1. [問題タイトル] ![Security](https://img.shields.io/badge/Security-silver?style=flat)`
  - 視点タグ: Code_Quality, Security, Performance, Testing, Design, Operations, Database, Others
- 問題コードや修正案は `<details>` タグでトグル表示にする
- セクション構成の順序を守る（サマリー → 優先度別 → ポジティブフィードバック）
- ファイル・行番号は `` `ファイル名:行番号` `` の形式で記載する
- すべての PR で詳細な標準フォーマットを使用する（問題点、理由、修正案を含む）

建設的で実用性を重視したフィードバックを心がけ、チームの学習と改善を支援してください。
```
:::

私たちのPJでは `建設的で実用性を重視したフィードバックを心がけ、チームの学習と改善を支援してください。` という一文を追加しています。
どこに問題がある。という断定的な言い方をせず、この辺りの動作確認はできているのか？という確認点もレビュー結果に踏まえてくれることがあります。この確認も侮ることができず、実際に深くコードを読んだり、動作確認をすると、修正すべきポイントがあった。となることもあります。

また、上記コマンドで使用していますが、レビューフォーマットも別途用意しています。
レビューフォーマットを用意することで、大量のテキストと冗長な表現を防ぐことができ、レビュー結果が常に変動するような気持ち悪さを防ぐことができます。

:::details review_output_format.md
```markdown
# Pull Request レビューフォーマット仕様

Pull Request のコードレビューコメントを統一されたフォーマットで出力するための仕様です。

## バッジシステム

### 優先度バッジ（shields.io）

| バッジ | 意味 | 使用場面 |
|--------|------|----------|
| ![MUST](https://img.shields.io/badge/MUST-red) | 必須対応 | セキュリティ問題、バグ、本番影響のある問題 |
| ![SHOULD](https://img.shields.io/badge/SHOULD-yellow) | 推奨対応 | パフォーマンス改善、保守性向上、ベストプラクティス |
| ![IMO](https://img.shields.io/badge/IMO-blue) | 個人的意見 | 設計の代替案、好みの問題 |
| ![nits](https://img.shields.io/badge/nits-lightgrey) | 細かい指摘 | コードスタイル、タイポ、小さなリファクタリング |
| ![ask](https://img.shields.io/badge/ask-orange) | 質問 | 実装意図の確認、不明点の質問 |
| ![FYI](https://img.shields.io/badge/FYI-informational) | 参考情報 | 関連情報、Tips、ドキュメントへのリンク |
| ![next](https://img.shields.io/badge/next-purple) | 次回対応推奨 | 今回は対応不要だが将来的に改善したい事項 |
| ![memo](https://img.shields.io/badge/memo-lightblue) | メモ | レビュアーのメモ、検討事項 |
| ![GOOD](https://img.shields.io/badge/GOOD-brightgreen) | 良い実装 | 評価すべき実装、学びになるコード |

### 視点タグ（shields.io）

各指摘には必ず視点タグバッジを付けます（見出しの末尾に配置）：

| バッジ | 視点 | 説明 |
|--------|------|------|
| ![Code Quality](https://img.shields.io/badge/Code_Quality-silver?style=flat) | コード品質 | コードの可読性、保守性、構造など |
| ![Security](https://img.shields.io/badge/Security-silver?style=flat) | セキュリティ | 認証、認可、脆弱性、データ保護など |
| ![Performance](https://img.shields.io/badge/Performance-silver?style=flat) | パフォーマンス | クエリ最適化、キャッシュ、レスポンス速度など |
| ![Testing](https://img.shields.io/badge/Testing-silver?style=flat) | テスト | テストカバレッジ、テスト品質など |
| ![Design](https://img.shields.io/badge/Design-silver?style=flat) | 設計 | アーキテクチャ、設計パターン、責務分離など |
| ![Operations](https://img.shields.io/badge/Operations-silver?style=flat) | 運用・保守 | ログ、監視、デプロイ、保守性など |
| ![Database](https://img.shields.io/badge/Database-silver?style=flat) | データベース | マイグレーション、インデックス、データ整合性など |
| ![Others](https://img.shields.io/badge/Others-silver?style=flat) | その他 | 上記に当てはまらないもの |

## セクション構成

以下の順序でレビューコメントを構成します：

### 1. 📊 レビューサマリー

```markdown
## 📊 レビューサマリー

- **総合評価**: [承認 / 条件付き承認 / 要修正]
- **diff サイズ**: [行数] 行
- **Critical Issues**: [件数] 件
- **Important Issues**: [件数] 件
- **Suggestions**: [件数] 件
```

### 2. 優先度別セクション

```markdown
## 🔴 Critical Issues

### 1. [問題タイトル] ![視点タグバッジ](https://img.shields.io/badge/視点-silver?style=flat)

![MUST](https://img.shields.io/badge/MUST-red) [問題の簡潔な説明] - `ファイル名:行番号`

<details>
<summary>📝 詳細を表示</summary>

**問題点:**
[問題の詳細]

**現在のコード:**
\`\`\`ruby
# 問題コード
\`\`\`

**修正案:**
\`\`\`ruby
# 修正後コード
\`\`\`

**理由:**
[修正が必要な理由]

</details>

---

## 🟡 Important Issues

### 1. [問題タイトル] ![視点タグバッジ](https://img.shields.io/badge/視点-silver?style=flat)

![SHOULD](https://img.shields.io/badge/SHOULD-yellow) [問題の説明] - `ファイル名:行番号`

<details>...</details>

---

## 💡 Suggestions

### 1. [提案タイトル] ![視点タグバッジ](https://img.shields.io/badge/視点-silver?style=flat)

![IMO](https://img.shields.io/badge/IMO-blue) [提案の説明] - `ファイル名:行番号`

<details>...</details>
```

### 3. ポジティブフィードバック

```markdown
## ✨ Positive Points

- ![GOOD](https://img.shields.io/badge/GOOD-brightgreen) [良い実装の説明]
- ![GOOD](https://img.shields.io/badge/GOOD-brightgreen) [評価すべき点]
```

## ファイル・行番号参照形式

- 単一行: `` `app/models/user.rb:42` ``
- 複数行: `` `app/models/user.rb:42-45` ``

## 出力例（最小版）

```markdown
## 📊 レビューサマリー

- **総合評価**: 条件付き承認
- **diff サイズ**: 320 行
- **Critical Issues**: 1 件
- **Important Issues**: 1 件
- **Suggestions**: 1 件

---

## 🔴 Critical Issues

### 1. SQL インジェクションの脆弱性 ![Security](https://img.shields.io/badge/Security-silver?style=flat)

![MUST](https://img.shields.io/badge/MUST-red) ユーザー入力を直接 SQL に埋め込んでいる - `app/models/search.rb:23`

<details>
<summary>📝 詳細を表示</summary>

**問題点:**
ユーザー入力をサニタイズせずに SQL クエリに埋め込んでいます。

**現在のコード:**
\`\`\`ruby
def search(query)
  execute("SELECT * FROM users WHERE name LIKE '%#{query}%'")
end
\`\`\`

**修正案:**
\`\`\`ruby
def search(query)
  where("name LIKE ?", "%#{sanitize_sql_like(query)}%")
end
\`\`\`

**理由:**
SQL インジェクション攻撃を防ぐため、プレースホルダーを使用します。

</details>

---

## 🟡 Important Issues

### 1. N+1 クエリ問題 ![Performance](https://img.shields.io/badge/Performance-silver?style=flat)

![SHOULD](https://img.shields.io/badge/SHOULD-yellow) N+1 クエリが発生 - `app/controllers/users_controller.rb:45`

<details>
<summary>📝 詳細を表示</summary>

**問題点:**
ユーザーごとに投稿を個別取得しています。

**現在のコード:**
\`\`\`ruby
@users = User.all
\`\`\`

**修正案:**
\`\`\`ruby
@users = User.includes(:posts)
\`\`\`

**理由:**
クエリ数を大幅に削減できます。

</details>

---

## 💡 Suggestions

### 1. サービスオブジェクトへの切り出し ![Design](https://img.shields.io/badge/Design-silver?style=flat)

![IMO](https://img.shields.io/badge/IMO-blue) コントローラーのロジックをサービスに切り出す - `app/controllers/orders_controller.rb:34-67`

<details>
<summary>📝 詳細を表示</summary>

**提案理由:**
コントローラーの責務を超えています。

**修正案:**
\`\`\`ruby
# app/services/order_processor.rb
class OrderProcessor
  def process
    # ロジック
  end
end
\`\`\`

**メリット:**
テストが書きやすく、再利用性が高まります。

</details>

---

## ✨ Positive Points

- ![GOOD](https://img.shields.io/badge/GOOD-brightgreen) エラーメッセージが分かりやすい
- ![GOOD](https://img.shields.io/badge/GOOD-brightgreen) テストが説明的
```

## 注意事項

- すべての指摘の見出し（### 1. など）の末尾に shields.io の視点タグバッジを配置する（例: `### 1. [問題タイトル] ![Security](https://img.shields.io/badge/Security-silver?style=flat)`）
- 視点タグバッジは全て silver 色で統一する
- 優先度バッジは見出しの次の行に配置する（例: `![MUST](https://img.shields.io/badge/MUST-red)`）
- `<details>` タグで問題コード・修正案をトグル表示にする
- コードブロックに言語指定（```ruby など）を必ず行う
- セクション構成の順序を守る
```
:::

### レビューコメントを定期的に収集する

ここまでの流れで、取り急ぎ自分たち専用の AIレビューは完成していますが、レビュー観点は定期的に更新されたり、追加されて然るべきです。

最後に、 Claude Code のスキルを使って、レビュー観点が更新されるようにします。

スキルは、カスタムコマンドと違い、 Claude が必要と判断したときのみ、ドキュメントを深く読み、アクションを行うものです。レビュー観点の更新はそう頻繁には行わないので、常にコンテキストに載っていなくてもよいよね。ということで、スキル化しました。

:::details SKILL.md
```markdown
---
name: update-backend-reviewer
description: "バックエンドPRレビュー指針を更新します。PRコメントを収集し、分析してbackend_reviewer.mdを再生成します。パッチリリース指針も同時に更新します。"
---

# バックエンドレビュー指針更新スキル

このスキルは、最新のPRレビューコメントを分析してバックエンドレビュー指針を更新します。
**パッチリリース（本番環境への緊急修正）の指針も同時に更新します。**

## 2種類のレビュー指針

| 指針 | 目的 | 重要度 |
|------|------|--------|
| `backend_reviewer.md` | 通常のコードレビュー観点 | 標準 |
| `patch_release_review_points.md` | パッチリリース防止のための重点観点 | **最重要** |

> **パッチリリースとは？**
> ステージングリリース後に発覚した不具合の緊急修正PRです。
> 通常のレビューで見逃された問題であり、今後のレビューで最も注意すべき観点となります。

---

## 実行手順

### Part A: 通常レビュー指針の更新

#### Step 1: PRコメントの収集

```bash
./.claude/skills/update-backend-reviewer/collect_pr_review_comments.sh
```

- 直近300件のクローズ済みバックエンドPRからレビューコメントを収集
- Bot（github-actions, dependabot等）のコメントは除外
- 出力: `artifacts/backend_pr_comments.txt`

#### Step 2: レビュー指針の生成

```bash
./.claude/skills/update-backend-reviewer/generate_backend_reviewer_guide.sh
```

- 収集したコメントを8観点（コード品質、セキュリティ、パフォーマンス、テスト、設計、運用保守、データベース、その他）で分析
- 出力: `artifacts/backend_reviewer.md`

---

### Part B: パッチリリース指針の更新（重要）

#### Step 3: パッチPRの収集

```bash
./.claude/skills/update-backend-reviewer/collect_patch_prs.sh
```

- `[patch]` タグ付きPRを収集（前回の分析以降の新規分）
- タイトル、説明、レビューコメントを取得
- 出力: `artifacts/backend_patch_prs.txt`

#### Step 4: パッチ指針の更新

```bash
./.claude/skills/update-backend-reviewer/generate_patch_review_guide.sh
```

- 新規パッチPRの根本原因を分析
- 既存の問題パターンに追加、または新規パターンを作成
- 出力: `artifacts/patch_release_review_points.md`

---

## 前提条件

- GitHub CLI (gh) がインストール済みで認証済みであること
- jq がインストール済みであること

確認コマンド:
```bash
gh auth status
jq --version
```

## ディレクトリ構成

```
.claude/skills/update-backend-reviewer/
├── SKILL.md                              # このファイル
├── collect_pr_review_comments.sh         # 通常PRコメント収集
├── generate_backend_reviewer_guide.sh    # 通常レビュー指針生成
├── collect_patch_prs.sh                  # パッチPR収集
├── generate_patch_review_guide.sh        # パッチ指針更新
└── artifacts/
    ├── backend_pr_comments.txt           # 収集したPRコメント
    ├── backend_reviewer.md               # 通常レビュー指針
    ├── backend_patch_prs.txt             # 収集したパッチPR情報
    └── patch_release_review_points.md    # パッチリリース防止指針（最重要）
```

## 使用シーン

- 定期的にレビュー指針を最新化したい場合
- 新しいレビューパターンが増えてきた場合
- チームのレビュー観点を振り返りたい場合
- **パッチリリースが発生した後、その教訓を指針に反映したい場合**
```
:::

:::details README_PR_Comment_Collector.md
```markdown
# PR Review Analysis Tools

xxx リポジトリのバックエンドPRレビューコメントを収集・分析し、レビューガイドラインを自動生成するツールセットです。

## 📂 このディレクトリについて

| ファイル | 概要 |
|----------|------|
| `collect_pr_review_comments.sh` | PRレビューコメント収集スクリプト |
| `generate_backend_reviewer_guide.sh` | レビューガイド生成スクリプト |
| `artifacts/backend_reviewer.md` | 生成されたレビューガイドライン |
| `artifacts/backend_pr_comments.txt` | 収集されたコメントデータ |

## 🎯 目的

- **レビュー観点の体系化**: 過去のレビューコメントからパターンを抽出
- **新メンバー支援**: 効果的なレビュー観点を明文化
- **レビュー品質向上**: チーム内のベストプラクティス共有

## 🚀 使用方法

### 前提条件

```bash
# GitHub CLI & jq のインストール
brew install gh jq
gh auth login
```

### backend_reviewer.md 更新手順

**1. PRコメント収集**
```bash
# 実行権限付与
chmod +x ./scripts/review_analysis/collect_pr_review_comments.sh

# 最新50件のPRからコメント収集
./scripts/review_analysis/collect_pr_review_comments.sh

# または件数を指定
./scripts/review_analysis/collect_pr_review_comments.sh 100
```

**2. レビューガイド生成**
```bash
# 実行権限付与
chmod +x ./scripts/review_analysis/generate_backend_reviewer_guide.sh

# ガイドライン自動生成
./scripts/review_analysis/generate_backend_reviewer_guide.sh
```

**3. 完了確認**
```bash
# 生成されたガイドを確認
cat ./scripts/review_analysis/artifacts/backend_reviewer.md
```

### パラメータ

| スクリプト | パラメータ | デフォルト | 説明 |
|------------|------------|------------|------|
| `collect_pr_review_comments.sh` | `number_of_prs` | 50 | 分析するPR件数 |
| `collect_pr_review_comments.sh` | `output_file` | `artifacts/backend_pr_comments.txt` | 出力先 |

## 📋 収集対象

### バックエンドPRの判定条件
- 拡張子: `.rb`, `.rake`, `.yml`, `.yaml`
- ファイル: `Gemfile`, `Rakefile`
- ディレクトリ: `config/`, `db/`, `app/`, `spec/`, `lib/`

### Bot除外
`github-actions[bot]`, `dependabot[bot]` 等のBotコメントは自動除外

## 📄 出力フォーマット

### backend_pr_comments.txt
```markdown
# バックエンド関連Pull Requestレビューコメント集

## 収集設定
- 分析対象: 過去50件の終了済みPR
- バックエンドPR数: 15件
- 収集日時: 2025年08月19日

## 収集されたコメント

### PR #26004: feat: スキルリスト処理の失敗ステータス細分化
**@reviewer** (2025-08-07):
N+1問題の可能性があります。includesの使用を検討してください。
```

### backend_reviewer.md
8つの観点（コード品質、セキュリティ、パフォーマンス、テスト、設計、運用保守、データベース、その他）でレビューポイントを整理したガイドライン

## 🔍 分析観点

1. **コード品質**: 命名規則、可読性、構造
2. **セキュリティ**: 認証・認可、SQL インジェクション対策
3. **パフォーマンス**: N+1問題、クエリ最適化
4. **テスト**: カバレッジ、エッジケース
5. **設計・アーキテクチャ**: SOLID原則、責務分離
6. **運用・保守性**: ログ、エラーハンドリング
7. **データベース**: マイグレーション、インデックス
8. **その他**: ドキュメント、プロセス改善

## 🛠️ トラブルシューティング

### GitHub CLI 認証エラー
```bash
gh auth login
gh auth status  # 確認
```

### API制限エラー
- 1時間待機またはPR件数を減らして実行
- Personal Access Token 使用で制限緩和

### jq コマンドエラー
```bash
brew install jq  # macOS
sudo apt-get install jq  # Ubuntu
```

## 📈 定期実行

### 週次分析（推奨）
```bash
# 毎週最新100件を分析してガイド更新
./scripts/review_analysis/collect_pr_review_comments.sh 100
./scripts/review_analysis/generate_backend_reviewer_guide.sh
```

### cron設定例
```bash
# 毎週月曜日 9時に実行
0 9 * * 1 cd /path/to/xxx && ./scripts/review_analysis/collect_pr_review_comments.sh 100 && ./scripts/review_analysis/generate_backend_reviewer_guide.sh
```

## ⚠️ 注意事項

- **API制限**: 認証済みで5,000 requests/hour
- **権限**: リポジトリ読み取り権限が必要
- **実行時間**: 100件以上の分析は時間がかかる場合がある
- **機密情報**: コメント内容に機密情報が含まれる可能性に注意

---

*xxx プロジェクトのレビュー品質向上を目的としたツール*
*最終更新: 2025年08月19日*
```
:::

スキルは、必要に応じてエージェントが使用しますが、自力で呼び出すこともできます。

```
skill を発揮して バックエンドコードレビューの指針を更新して
```

と話しかけることで、無事レビュー指針が更新されるようになりました。

## AI コードレビューの活用例

私が関わるグロービス様の PJにおいても、しくみ製作所内での PJにおいても、AIレビューの活用は進んでいます。
また、エッジなケースでは、人間のコードレビューはせず、AIが承認したり、一通りの指摘事項を修正したら、マージOKとするPJもあります。

理想的には人間のコードレビューを実施すべきと考えていますが、プロダクトのフェーズによっては、上記のような使い方もありなのだと思います。

## 最後に

レビューに出されているコードの具体的なパフォーマンスやロジックの欠陥を指摘するのは AIレビューが得意な一方で、 UIなどビジュアル面での指摘や、システム・アーキテクチャ的な側面での指摘は苦手な傾向があると思います。

今回、人間のレビューコメントを元にレビュー指針を作り、AIにレビューをしてもらう取り組みをご紹介しましたが、それでもなお、前述のような特徴は出てくると思います。

レビュー結果においても、過信はせず、良い側面は取り入れつつ、疑問に思う点は少し深く自力でレビューをしていくと、AIレビューにより、生産性の向上を見込めると思います。

もしご興味いただけましたら、年末に向けて、自分のチーム向けの AI コードレビュー作ってみてください！

それでは、よいお年を！
