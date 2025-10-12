```shell

#!/bin/bash

# Fetch latest branch info and prune deleted remote branches
git fetch --prune

# Define cutoff date (14 days ago)
CUTOFF=$(date -v-14d +%s 2>/dev/null || date -d '14 days ago' +%s)

echo "🧹 Deleting local branches inactive for more than 2 weeks..."
git for-each-ref --sort=committerdate refs/heads/ --format='%(refname:short) %(committerdate:unix)' | while read branch date; do
  if [ "$branch" != "main" ] && [ "$branch" != "master" ] && [ "$branch" != "$(git branch --show-current)" ]; then
    if [ "$date" -lt "$CUTOFF" ]; then
      echo "Deleting local branch: $branch"
      git branch -D "$branch"
    fi
  fi
done

echo ""
echo "🌍 Deleting remote branches inactive for more than 2 weeks (already merged)..."

# List remote branches with last commit date, delete if old + merged
git for-each-ref --sort=committerdate refs/remotes/origin/ --format='%(refname:short) %(committerdate:unix)' | while read ref date; do
  branch=${ref#origin/}
  if [ "$branch" != "main" ] && [ "$branch" != "master" ]; then
    if [ "$date" -lt "$CUTOFF" ]; then
      # Check if remote branch has already been merged into origin/main
      if git branch -r --merged origin/main | grep -q "origin/$branch$"; then
        echo "Deleting remote branch: origin/$branch"
        git push origin --delete "$branch"
      fi
    fi
  fi
done
```
