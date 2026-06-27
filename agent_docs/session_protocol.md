# TAORM — Session Protocol

This is the operating manual for starting and ending every session.
It is not project content — it is the habit that makes the memory system work.

---

## START OF SESSION

When you open a new OpenCode session, do these steps in order:

### Step 1: Read this file
You are reading it now. Good.

### Step 2: Read opencode.md
This gives you the project overview and links to all design documents.

### Step 3: Read current_task.md
This tells you what we were doing last time and what to do next.

### Step 4: Read the last 3 entries of changelog.md
This tells you what happened recently and why, so you don't re-explain
or re-litigate old decisions.

### Step 5: Read milestone_log.md (optional)
For big-picture context. Useful if you need to understand where we are
in the overall project timeline.

### Step 6: Verify the environment
If working on Arch, quickly verify:
```bash
sensors                    # thermal working?
cat /sys/fs/cgroup/cgroup.controllers  # cgroups v2?
```

### Step 7: Ask the user
If anything is unclear after reading the above files, ask.
Don't assume. Don't hallucinate. Say "I don't know" if you're not sure.

---

## DURING SESSION

- Work on the task in current_task.md
- If you make a design decision, add it to decisions.md
- If you break something, note it in changelog.md
- Update milestone_log.md if a phase status changes
- Run tests and record results in test_log.md

---

## END OF SESSION

Before the session ends, do these steps:

### Step 1: Update current_task.md
Overwrite it with the NEXT thing to do. Be specific — the next session
should be able to pick up without asking questions.

### Step 2: Add an entry to changelog.md
Use the template. Newest entry at the top. Include:
- What we did
- What we decided
- What broke
- What we deferred
- What to do next

### Step 3: Update milestone_log.md (if status changed)
If a phase moved from Not started to In progress, or In progress to Done,
update it here.

### Step 4: Update codebase_map.md (if files moved)
If you added, removed, or renamed files, update the map.

### Step 5: Update decisions.md (if any decision was made)
Add a new ADR for any significant design choice.

### Step 6: Update test_log.md (if tests were run)
Record what was tested, the numbers, and what happened.

---

## RULES

1. **Never assume.** If you don't know something, say so.
2. **Never hallucinate.** If you can't verify it, don't state it as fact.
3. **Search the web** if you need information about Arch packages, syscalls, etc.
4. **Keep it short.** Files should be concise. Detailed docs live in the design docs.
5. **One decision = one ADR.** Don't bury decisions inside changelog entries.
6. **current_task.md is overwritten every session.** It is NOT history.
7. **changelog.md is append-only.** Newest at the top.
