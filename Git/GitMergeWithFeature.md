# Merging Changes from Master into a Feature Branch

This guide explains how to merge the latest changes from the master branch into your feature branch while ensuring your changes are preserved.

![image](https://github.com/user-attachments/assets/9dc4b8d8-a978-4c24-b41d-27d38cd9ef07)


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

<img width="841" alt="image" src="https://github.com/user-attachments/assets/c46153b1-c025-4fd9-b80a-2ae357e28ea3" />


### Reference 

- https://www.atlassian.com/git/tutorials/merging-vs-rebasing
