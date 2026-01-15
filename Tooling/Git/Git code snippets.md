```powershell
# REPOSITORY INITIALIZATION
git init

# STAGING & COMMITTING
git add <file>              # Stage specific file
git add .                   # Stage all changes
git commit -m "message"     # Create commit

# STATUS & HISTORY
git status                  # Working tree state
git log                     # Full commit history
git log --oneline --graph   # Compact visual history

# BRANCHING
git branch <name>           # Create branch
git checkout <name>         # Switch branch
git checkout -b <name>      # Create and switch
git merge <name>            # Merge branch into current

# GITHUB INTEGRATION - Initial Setup
git remote add origin https://github.com/username/repo.git
git branch -M main
git push -u origin main

# GITHUB INTEGRATION - Ongoing
git pull origin main        # Fetch + merge remote changes
git push origin main        # Upload local commits

# CONFIGURATION
git config --global user.name "Your Name"
git config --global user.email "email@example.com"
git config --global core.autocrlf true  # CRLF/LF handling for Windows

# CLONE EXISTING REPO
git clone https://github.com/username/repo.git
```

```powershell
# TYPICAL WORKFLOW EXAMPLE
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/username/repo.git
git push -u origin main

# SUBSEQUENT CHANGES
git add modified_file.txt
git commit -m "Update feature"
git push origin main
```