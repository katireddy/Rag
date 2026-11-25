# Git – Short Notes for DevOps Interviews

## 1. What is Git?

Git is a **Distributed Version Control System (DVCS)** used to track code changes, maintain versions, collaborate with developers, and manage source code.

## 2. Important Git Commands

| Command | Purpose |
|---|---|
| `git init` | Initialize a Git repository |
| `git clone <url>` | Download an existing repository |
| `git status` | Check modified and untracked files |
| `git add .` | Stage all changes |
| `git add <filename>` | Stage a specific file |
| `git commit -m "message"` | Save changes locally |
| `git push` | Upload commits to a remote repository |
| `git pull` | Fetch and integrate remote changes |
| `git fetch` | Download remote changes without merging |
| `git merge <branch>` | Merge another branch into the current branch |
| `git rebase <branch>` | Reapply commits on top of another branch |
| `git reset` | Move the current branch/HEAD to another commit |
| `git revert <commit>` | Undo a commit by creating a new commit |
| `git cherry-pick <commit>` | Apply a specific commit to the current branch |
| `git log` | View commit history |
| `git diff` | Compare changes |
| `git branch` | List branches |
| `git switch <branch>` | Switch branches |
| `git stash` | Temporarily save uncommitted changes |

## 3. Basic Git Workflow

```bash
git pull
# Make code changes
git status
git add .
git commit -m "Add feature"
git push
```

The usual flow is:

1. **Working directory** – Modify your code.
2. **Staging area** – Use `git add` to select changes for the next commit.
3. **Local repository** – Use `git commit` to save a snapshot locally.
4. **Remote repository** – Use `git push` to upload commits to GitHub or another remote.

## 4. Important Differences

| Command | Meaning |
|---|---|
| `git reset` | Moves the current branch/HEAD to another commit. Depending on the option, it can also change the staging area and working files. |
| `git revert` | Creates a new commit that reverses the changes introduced by an earlier commit. |
| `git pull` | Fetches remote changes and integrates them into the current branch. |
| `git fetch` | Downloads remote updates but does not integrate them into the current branch. |
| `git cherry-pick` | Applies the changes from a selected commit onto the current branch. |
| `git merge` | Integrates another branch into the current branch. |
| `git rebase` | Replays commits onto a new base, creating a more linear history. |

**Quick memory tips:**

- Reset = Move the branch pointer.
- Revert = Undo using a new commit.
- Fetch = Download updates.
- Pull = Fetch + integrate.
- Cherry-pick = Apply one selected commit.

## 5. Git in VS Code

Most everyday Git operations can be performed from the **Source Control** panel in VS Code.

Typical workflow:

1. Open the project in VS Code.
2. Open **Source Control**.
3. Review modified and untracked files.
4. Stage selected files using the `+` control, or stage all changes.
5. Enter a meaningful commit message.
6. Select **Commit**.
7. Pull, push, or sync changes using the Source Control menu or status bar.
8. Resolve merge conflicts in the editor when they occur.

You can stage individual files instead of staging every change. Always review the diff before committing.

## 6. Best Practices

- Pull the latest changes before starting work and regularly during collaboration.
- Review changes before staging and committing.
- Use meaningful commit messages.
- Create separate branches for features and bug fixes.
- Keep commits focused and small.
- Resolve conflicts carefully; do not blindly overwrite another developer's changes.
- Be cautious with `git reset --hard` because it can discard local work.
- Avoid force-pushing shared branches unless the team workflow explicitly allows it.

> Pulling frequently can reduce the chance and size of merge conflicts, but it does not guarantee that conflicts will not occur.

## 7. Interview Explanation

"I use Git for version control and collaboration. My regular workflow is to pull the latest changes, modify the code, stage the required files, commit the changes locally, and push them to the remote repository.

In day-to-day development, I use the VS Code Source Control interface for common Git operations. I can stage individual files, commit changes, pull and push updates, manage branches, and resolve merge conflicts through the editor.

I also understand operations such as reset, revert, rebase, merge, and cherry-pick. Pulling the latest changes regularly helps reduce integration issues and merge conflicts."
