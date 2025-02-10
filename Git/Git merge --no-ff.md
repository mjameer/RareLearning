## What is `--no-ff` in `git merge --no-ff <source-branch>`?

### Background:
When merging a branch in Git, the default behavior is a **fast-forward merge** if there are no changes on the target branch since the branch was created. However, you can use the `--no-ff` flag to force Git to create a **merge commit** even if the merge could be performed with a fast-forward.

### What Happens:
- **Without `--no-ff`**: Git will simply move the pointer of the target branch forward to include the changes from the source branch.
- **With `--no-ff`**: Git will create a **merge commit**, preserving the target branch's history and making it clear when the source branch was integrated into the target branch.

### Example:

![image](https://github.com/user-attachments/assets/807b8b7a-89a4-430f-b860-c2c4bb8b6639)
