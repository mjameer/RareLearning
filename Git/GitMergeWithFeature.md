# Merging Changes from Master into a Feature Branch

This guide explains how to merge the latest changes from the master branch into your feature branch while ensuring your changes are preserved.


## Steps to Merge Master into Feature Branch

```bash
git checkout <your-feature-branch>
git fetch origin
git merge origin/master -f
# Resolve conflicts if any
git add <resolved-file>
git commit
git push origin <your-feature-branch>
```
