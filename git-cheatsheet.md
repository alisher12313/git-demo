# Git Revision Cheat Sheet

---

## 1. Version control concept
Git records **snapshots** of your project over time, so you can see who changed what, go back, and work in parallel on branches. It's **distributed**: every clone has the full history, and GitHub is just another copy that everyone shares.

## 2. Install & configure
```bash
git --version                                  # check Git is installed
git config --global user.name "Your Name"      # label on your commits
git config --global user.email "you@mail.com"  # links commits to your GitHub profile
git config --global user.email                 # show current value
git config --list                              # show all settings
```
- Name and email are **labels for people**, not a login. Git doesn't check them.
- Use an email that's on your GitHub account, so commits link to your profile.

## 3. GitHub & SSH
The SSH key is **how the server knows it's you**, so you don't log in every time.
```bash
ssh-keygen -t rsa -C "you@mail.com"            # create key pair (ed25519 is more modern)
Get-Content ~\.ssh\id_rsa.pub | Set-Clipboard  # copy PUBLIC key (PowerShell)
ssh -T git@github.com                          # test connection
```
- Paste the **public** key (`.pub`) into GitHub → Settings → SSH and GPG keys. **Never share the private key.**
- The key proves identity only. Your access still depends on repo permissions.
- It only works with SSH addresses: `git@github.com:user/repo.git`.
```bash
git clone git@github.com:user/repo.git
git remote -v                                            # see remote address
git remote set-url origin git@github.com:user/repo.git   # switch HTTPS → SSH
```

## 4. Graphical tools
Same Git, but with buttons: `git gui`, `gitk`, and the PyCharm Git panel (commits, diffs, **Resolve Conflicts**). Good for seeing history and fixing conflicts. Everything they do maps to commands.

## 5. The basic workflow
```
Working folder --add--> Staging (index) --commit--> Local repo --push--> GitHub
```
```bash
git status              # what changed / what's staged
git add file.txt        # stage one file   (git add . = everything)
git commit -m "msg"     # save snapshot locally
git push                # upload to GitHub
git fetch               # download new commits, DON'T touch your files
git merge origin/main   # apply them to your branch
git pull                # fetch + merge in one step
```
Looking at changes:
```bash
git diff                # unstaged changes
git diff --staged       # staged changes
git log --oneline       # history, short
git log --oneline --all # include GitHub's branches (origin/main)
git show <hash>         # details of one commit   (q = exit)
```
Other useful ones:
```bash
touch file.txt          # create empty file
echo "text" > file.txt  # create/overwrite file with text (>> appends)
mkdir folder            # create folder
cd ~/Desktop            # go to Desktop (Git Bash; PowerShell: cd ~\Desktop)
```
- `.gitignore` lists files Git should never track (like `.env`).

## 6. Git internals
- `.git` folder = the whole repository. **Never edit it by hand.**
- Every object is named by its **SHA-1 hash** (40 characters, calculated from the contents).
- 3 object types: **commit** (points to a tree plus the previous commit) → **tree** (file names and folders) → **blob** (file contents).
- Unchanged files aren't duplicated. New commits just point to the same blob.
- `HEAD` = where you are now. `refs/heads/main` = the hash of the latest commit on main. **A branch is just a pointer to a commit.**
- Stored in `.git/objects/61/2179...` (the first 2 characters are the folder name).
```bash
git show --pretty=raw <hash>   # commit + its tree
git ls-tree <tree-hash>        # files in a tree
git show <blob-hash>           # file contents
cat .git/HEAD                  # which branch you're on
```

## 7. Undoing changes

**Files (not committed yet):**
```bash
git restore --staged file   # unstage (old: git reset -- file); changes kept
git restore file            # DISCARD changes (old: git checkout -- file) ⚠️
git clean -n                # preview deleting untracked files
git clean -fd               # delete untracked files (-x also deletes ignored ones ⚠️)
```

**Commits, NOT pushed yet** (only changes your computer):
```bash
git commit --amend -m "new msg"     # fix last commit message
git add forgotten.txt
git commit --amend --no-edit        # add forgotten file to last commit
git reset --soft HEAD~1             # undo commit, changes stay staged
git reset HEAD~1                    # (--mixed) undo commit, changes unstaged
git reset --hard HEAD~1             # undo commit AND delete changes ⚠️
```

| Command | Commit | Staged? | Changes in file |
|---|---|---|---|
| `git reset --soft HEAD~1` | undone | ✅ staged | kept |
| `git reset HEAD~1` | undone | ❌ unstaged | kept |
| `git reset --hard HEAD~1` | undone | ❌ | **deleted** ⚠️ |

**Commits ALREADY pushed:**
```bash
git revert <hash>    # new commit that cancels the old one → normal push
git revert HEAD      # revert the last commit
```
- Rule: **not pushed → reset/amend. Pushed → revert.**
- Reset/amend after pushing needs `git push --force-with-lease`. Only do that on your own branch.
- Revert needs the commit in your local branch, so `git pull` first if you're behind.
- `HEAD~1` = one commit back, `HEAD^^` / `HEAD~2` = two commits back.
- **Pushed a secret (`.env`)?** Change the keys first, then:
  ```bash
  git rm --cached .env
  echo ".env" >> .gitignore
  git add .gitignore
  git commit -m "Stop tracking .env"
  git push
  ```
- Vim opened? Press `Esc`, then `:wq` (save) or `:q!` (quit).

## 8. Branching & merge
```bash
git branch                  # list (* = current)
git branch -a               # include GitHub branches
git switch -c feature       # create + switch (old: git checkout -b)
git switch main             # switch (old: git checkout main)
git push -u origin feature  # first push of a new branch
git merge feature           # merge feature INTO current branch
git stash                   # park uncommitted work
git stash pop               # bring it back
```

**Conflicts.** When the same lines changed on both branches, Git marks the file:
```
<<<<<<< HEAD
your version
=======
their version
>>>>>>> feature
```
Edit it to the final version, **delete the markers**, then `git add file` + `git commit`. Other options:
```bash
git merge --abort                 # cancel the merge
git checkout --ours file          # keep your version
git checkout --theirs file        # keep their version
git merge -X ours feature         # auto-pick your side for all conflicts
git mergetool                     # visual tool (or PyCharm: Git → Resolve Conflicts)
git revert -m 1 <merge-hash>      # undo a merge commit
```

**Keeping your branch up to date with main:**
```bash
git switch feature
git fetch
git rebase origin/main      # clean straight history
# or
git pull origin main        # merge commit, simpler
```
- Both give you main's latest code (e.g. a friend's utils). The difference is history and pushing: rebase needs `--force-with-lease` if the branch was already pushed.
- **Golden rule:** rebase only **your own** branch onto main. Never rebase `main` or a branch others push to, because rebase creates new commits with new hashes.
- Starting a new task:
  ```bash
  git switch main
  git pull
  git switch -c new-branch
  ```
