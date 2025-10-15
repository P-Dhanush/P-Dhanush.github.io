 (cd "$(git rev-parse --show-toplevel)" && git apply --3way <<'EOF' 
diff --git a/README.md b/README.md
index b65e19ca79b1b86d3f5b29ba274667f6cb3e7642..8768bd58d7066f2899f4afebd87a8bcaa900d0fa 100644
--- a/README.md
+++ b/README.md
@@ -1 +1,25 @@
-# P-Dhanush.github.io
\ No newline at end of file
+# Personal Blog on GitHub Pages
+
+This repository powers the blog hosted at **https://p-dhanush.github.io**. It uses the built-in `jekyll-theme-minimal` theme so you can publish Markdown posts with minimal configuration.
+
+## Writing and Publishing Posts
+
+1. Create a Markdown file inside the `_posts` directory named with the pattern `YYYY-MM-DD-title.md`.
+2. Ensure the post's front matter contains a past or present `date` so Jekyll publishes it immediately.
+3. Commit the file to the default branch (usually `main`). GitHub Pages will rebuild the site automatically.
+
+If you prefer working through pull requests, you do **not** need to approve your own PR to merge it. After you review your changes:
+
+- Click **Merge pull request** directly, or
+- Adjust your repository's branch protection settings to disable "Require approvals" if you do not have additional collaborators yet.
+
+## Local Preview (optional)
+
+To preview locally you can install Ruby, Jekyll, and Bundler, then run:
+
+```bash
+bundle install
+bundle exec jekyll serve
+```
+
+The site will be available at `http://localhost:4000`.
 
EOF
)
