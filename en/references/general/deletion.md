# Deletion

Delete what is not used. Version control keeps the history, so there is no need to keep things because they "might be useful later".

### Delete everything unused [Required]

Delete code, logic, configuration, and files once you confirm they have no consumers.

**Why:** Unused content misleads readers and adds cost to every change and investigation.

**Source:** Team

### Delete on discovery [Required]

Handle it once you find it. Do not let it go as "keep it for now". When it is unrelated to the current task, list it in your report and ask first, then use a separate branch and PR once authorized. See [change-workflow.md](../collaboration/change-workflow.md).

**Source:** Team

### Confirm there are no consumers before deleting [Required]

Judge from real evidence, not guesses:

1. Search globally for references, including dynamic references in string form.
2. Check configuration, scripts, scheduled jobs, and CI.
3. When data is involved, confirm whether anything still reads it.
4. When in doubt, confirm with the user.

If nothing reads it, there is no need to keep writing or retaining it. The exception is legally retained, audit, and backup data, which follow their retention periods. See [Privacy](../backend/security/privacy.md).

## Cleanup scope

| Category | Content |
|---|---|
| Code | Uncalled functions, dead branches, superseded implementations |
| Flags | New and old flags whose switchover is complete |
| Data | Fields, tables, and old indexes with no readers (except legally retained, audit, and backup data) |
| Tests | Mocks in product code, unused test doubles, outdated tests and test data |
| Docs | Outdated docs and files with no purpose |
| Engineering | Unnecessary gate checks and scripts |
| Branches | Merged branches |
| Requirements | Requirements not worth doing |

### Delete the old implementation after replacing it [Required]

Once the new implementation ships, delete the flags, old paths, and old tests together.

**Source:** Team

### Delete dropped requirements along with their docs [Required]

When you decide not to do something, delete the requirement from the docs too, so later work does not keep building around it.

**Source:** Team

### Confirm before destructive data operations [Required]

Version control is the fallback for deleted code. Deleted production data cannot be recovered the same way. Confirm with the user before deleting production data or anything unrecoverable.
