## Core Concepts

**Repository**: A data structure containing project files and complete version history.  
**Commit**: A snapshot of repository state at a specific point in time, identified by a SHA-1 hash.  
**Branch**: A movable pointer to a commit, enabling parallel development streams.  
**Remote**: A reference to a repository hosted on an external server (e.g., GitHub).

## Essential Commands

### Repository Initialization

```cmd
git init
```

Creates a new Git repository in the current directory, establishing `.git` metadata structure.

### Staging and Committing

```cmd
git add <file>
git add .
git commit -m "commit message"
```

`git add` stages files to the index. `git commit` creates a commit object from staged changes.

### Status Inspection

```cmd
git status
git log
git log --oneline --graph
```

Displays working tree state and commit history.

### Branch Operations

```cmd
git branch <branch-name>
git checkout <branch-name>
git checkout -b <branch-name>
git merge <branch-name>
```

Creates, switches between, and merges branches.

## GitHub Integration

### 1. Link Local Repository to GitHub Remote

```cmd
git remote add origin https://github.com/username/repository.git
```

Establishes `origin` as the default remote reference pointing to the GitHub repository URL.

### 2. Push Initial Commit

```cmd
git branch -M main
git push -u origin main
```

`-M` renames current branch to `main`. `-u` sets upstream tracking relationship.

### 3. Subsequent Operations

```cmd
git pull origin main
git push origin main
```

`git pull` fetches and merges remote changes. `git push` uploads local commits to remote.

### Authentication

GitHub requires token-based authentication. Configure credentials using:

```cmd
git config --global user.name "Your Name"
git config --global user.email "email@example.com"
```

For HTTPS URLs, Windows Credential Manager stores access tokens after first authentication.

## Windows-Specific Considerations

**Path Separators**: Git internally uses forward slashes (`/`) regardless of Windows backslash convention.

**Line Endings**: Configure `core.autocrlf` to handle CRLF/LF conversion:

```cmd
git config --global core.autocrlf true
```

Value `true` converts LF to CRLF on checkout, CRLF to LF on commit.

**Shell Quoting**: Windows Command Prompt requires double quotes for arguments containing spaces. PowerShell follows similar conventions but may interpret certain characters differently. Git Bash (bundled with Git for Windows) provides Unix-like shell behavior and is recommended for cross-platform consistency.




## Optimal Reconnection Protocol

### Step 1: Remote Reference Removal

```powershell
git remote remove origin
```

**Operation**: Deletes the named remote reference `origin` from `.git/config`, severing the association between local repository and previous remote endpoint.

### Step 2: Remote Reference Establishment

```powershell
git remote add origin https://github.com/username/repository.git
```

**Operation**: Creates new remote reference entry mapping symbolic name `origin` to specified GitHub repository URL.

### Step 3: Remote State Inspection

```powershell
git fetch origin
```

**Operation**: Downloads all refs and objects from remote repository without modifying working tree or local branches. Populates remote-tracking references under `.git/refs/remotes/origin/`.

**Output Analysis**: Examine fetch output to determine reconciliation strategy.

## Reconciliation Decision Tree

### Scenario A: Identical History (Zero Divergence)

**Condition**: Local `main` and `origin/main` reference identical commit object SHA-1.

**Verification**:

```powershell
git log main..origin/main  # Empty output
git log origin/main..main  # Empty output
```

**Action**:

```powershell
git branch --set-upstream-to=origin/main main
```

**Effect**: Establishes upstream tracking relationship without data transfer. Configuration entries `branch.main.remote` and `branch.main.merge` are written to `.git/config`.

### Scenario B: Remote Ahead (Local Behind)

**Condition**: Remote contains commits not present in local repository. Local history is strict subset of remote history.

**Verification**:

```powershell
git log main..origin/main  # Shows remote-only commits
git log origin/main..main  # Empty output
```

**Action**:

```powershell
git pull origin main
```

**Effect**: Fast-forward merge. Local `main` ref advances to match `origin/main` without creating merge commit. Working tree updated with remote changes.

### Scenario C: Local Ahead (Remote Behind)

**Condition**: Local repository contains commits not present on remote. Remote history is strict subset of local history.

**Verification**:

```powershell
git log main..origin/main  # Empty output
git log origin/main..main  # Shows local-only commits
```

**Action**:

```powershell
git push -u origin main
```

**Effect**: Uploads local commit objects to remote. Remote `main` ref fast-forwards to match local `main`. `-u` flag establishes upstream tracking.

### Scenario D: Divergent History (Conflict State)

**Condition**: Both repositories contain commits not present in the other. Commit graphs have diverged from common ancestor.

**Verification**:

```powershell
git log main..origin/main  # Shows remote-only commits
git log origin/main..main  # Shows local-only commits
```

**Action - Method 1: Merge Reconciliation**

```powershell
git pull origin main
```

**Effect**: Creates merge commit with two parents, integrating both histories. Preserves complete commit graph topology. May trigger merge conflicts requiring manual resolution.

**Action - Method 2: Forced Synchronization**

```powershell
git push --force origin main
```

**Effect**: Overwrites remote history with local history. Remote commits not present locally are orphaned and subject to garbage collection. **Destructive operation**: Remote commit objects become unreachable unless referenced by other branches or tags.

**Selection Criterion**: Use Method 1 if remote commits contain valuable data requiring preservation. Use Method 2 if local state represents canonical truth and remote history is disposable.

## Windows Command Line Consideration

**Quoting Convention**: PowerShell and Command Prompt handle argument parsing identically for Git commands containing standard alphanumeric characters and common symbols. Complex commit messages containing special characters (`$`, `` ` ``, `&`, `|`) require double-quote encapsulation in Command Prompt or backtick escaping in PowerShell. Git Bash provides POSIX-compliant shell behavior eliminating platform-specific quoting variations.

## Minimal Command Sequence - Common Case

```powershell
git remote remove origin
git remote add origin https://github.com/username/repository.git
git fetch origin
git branch --set-upstream-to=origin/main main
git pull origin main
```

**Applicability**: Suitable when remote history is superset of local history or when merge reconciliation is acceptable conflict resolution strategy.