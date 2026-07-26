# TEAMCITY PULL REQUEST PIPELINE

## Goal

When a developer creates a Pull Request to main:

1. Trigger TeamCity
2. Find the changed YAML file
3. Allow ONLY one YAML file
4. YAML must be either:
   - `*-pre.yaml`
   - `*-post.yaml`
5. Store the file path
6. Run validation on that file

---

## STEP 1 - CREATE PROJECT

**Projects**
→ Create Project

**Repository**

```
https://github.com/laxmandudhate/healthcheck.git
```

---

## STEP 2 - BUILD CONFIGURATION

**Name**

```
PR-Build
```

---

## STEP 3 - VERSION CONTROL SETTINGS

**Repository**

```
https://github.com/laxmandudhate/healthcheck.git
```

**Default Branch**

```
refs/heads/main
```

**Branch Specification**

```
+:refs/heads/*
```

**Do NOT add**

```
refs/pull/*
```

---

## STEP 4 - PULL REQUEST BUILD FEATURE

**Build Features**
→ Add Build Feature
→ Pull Requests

**Provider**

```
GitHub
```

**Authentication**

```
GitHub Token
```

**Target Branch Filter**

```
+:main
```

**Source Branch Filter**

```
Leave Empty
```

---

## STEP 5 - VCS TRIGGER

**Triggers**
→ Add Trigger
→ VCS Trigger

**Branch Filter**

```
+:pull/*
-:*
```

---

## STEP 6 - BUILD STEP 1

**Runner**

Command Line

**Purpose**

Debug Git Checkout

**Script**

```bash
#!/bin/bash

echo "Branch:"
git branch -a

echo
echo "Commit:"
git rev-parse HEAD

echo
echo "History:"
git log --oneline --graph --decorate --all -10
```

---

## STEP 7 - BUILD STEP 2

**Purpose**

Find changed YAML file

**Rules**

- Exactly ONE file is allowed.

**Allowed**

- `*-pre.yaml`
- `*-post.yaml`

**Script**

```bash
#!/bin/bash
set -e

echo "===== PR BUILD ====="

echo "Branch: %teamcity.build.branch%"

git fetch --unshallow || true
git fetch origin main

FILES=$(git diff --name-only origin/main..HEAD | grep -E '(-pre|-post)\.yaml$' || true)

COUNT=$(echo "$FILES" | sed '/^$/d' | wc -l)

if [ "$COUNT" -eq 0 ]; then
    echo "No pre/post YAML file found."
    exit 1
fi

if [ "$COUNT" -gt 1 ]; then
    echo "Only one pre/post YAML file is allowed."
    echo "$FILES"
    exit 1
fi

FILE="$FILES"

echo "Changed YAML file"

echo "$FILE"

echo "##teamcity[setParameter name='env.YAML_FILE' value='$FILE']"
```

---

## STEP 8 - BUILD STEP 3

**Purpose**

Run validation

**Script**

```bash
#!/bin/bash

echo "File to validate"

echo "%env.YAML_FILE%"
```

**Examples**

```bash
python validate.py "%env.YAML_FILE%"
```

OR

```bash
yamllint "%env.YAML_FILE%"
```

OR

```bash
./validate.sh "%env.YAML_FILE%"
```

---

## PIPELINE FLOW

```
Developer
    ↓
Create Feature Branch
    ↓
Modify
    changes/CHNG1234567890/ITSK1234586068/laxman-mode-pre.yaml
    OR
    changes/CHNG1234567890/ITSK1234586068/laxman-mode-post.yaml
    ↓
Commit
    ↓
Push
    ↓
Create Pull Request
    ↓
TeamCity Trigger
    ↓
Checkout Repository
    ↓
Step 1: Print Git Information
    ↓
Step 2: Find Changed YAML
    ↓
Verify: Exactly ONE file exists
    ↓
Save path into env.YAML_FILE
    ↓
Step 3: Validate YAML
    ↓
Success OR Fail Build
```

---

## RESULTS

### Case 1: PASS ✓

**Changed**

```
changes/.../laxman-mode-pre.yaml
```

**Result** → PASS

---

### Case 2: PASS ✓

**Changed**

```
changes/.../laxman-mode-post.yaml
```

**Result** → PASS

---

### Case 3: FAIL ✗

**Changed**

```
pre.yaml
post.yaml
```

**Result** → FAIL

**Reason** → Only one YAML file allowed

---

### Case 4: FAIL ✗

**Changed** → No YAML file

**Result** → FAIL

**Reason** → No pre/post YAML file found

---

## TEAMCITY PARAMETER

**Parameter Name**

```
env.YAML_FILE
```

**Example Value**

```
changes/CHNG1234567890/ITSK1234586068/laxman-mode-pre.yaml
```

**Use in next steps**

```
%env.YAML_FILE%
```
