# WhatsApp Feedback Pipeline

A local Windows pipeline that catches up selected WhatsApp groups, identifies likely product feedback, and stages it as **GitHub Project draft issues**. Nothing becomes a repository issue until a person reviews and converts the draft.

## Workflow

1. Windows starts the pipeline at sign-in and every six hours while the laptop is available.
2. The saved WhatsApp Web session synchronizes the selected groups.
3. Messages newer than the local cursor are inserted into SQLite with message-ID deduplication.
4. A lightweight local rules engine classifies bugs, feature requests, design ideas, feedback, and noise.
5. Every 48 hours, candidates are deduplicated and added to a private GitHub Project as draft items.
6. A Windows notification appears and the review project opens.
7. The reviewer converts approved drafts to issues and archives rejected drafts.

If a scheduled time occurs while the laptop is off, Windows Task Scheduler starts the workflow when possible. The next run synchronizes WhatsApp before preparing the overdue digest.

## Requirements

- Windows 10 or Windows 11.
- Node.js 20 or newer.
- Google Chrome or Microsoft Edge installed on Windows.
- GitHub CLI (`gh`) authenticated with `gh auth login`.
- A WhatsApp account that is already a member of the selected groups.
- Permission from the team to process feedback messages and store approved content in GitHub.

No paid API, cloud server, or separate dashboard is required.

## Installation

Extract the project into a permanent folder. Moving it later will invalidate the scheduled-task paths.

Open PowerShell in the project folder and run:

```powershell
Set-ExecutionPolicy -Scope Process Bypass
.\scripts\setup-windows.ps1
```

The setup process will:

1. Install the locked Node dependencies.
2. Verify GitHub CLI authentication.
3. Create a new private GitHub repository.
4. Create a GitHub Project named `Product Feedback Inbox`.
5. Display a WhatsApp QR code.
6. List the available WhatsApp groups.
7. Ask which groups should be monitored and whether each is a `user` or `design` source.
8. Install two Windows scheduled tasks.

The installer uses the Chrome or Edge installation already present on Windows; it does not download another browser runtime. It also applies a pinned, integrity-checked upstream compatibility patch for the current WhatsApp Web `2.3000.x` serialized-ID change. The patch is temporary and can be removed after the fix is included in a stable `whatsapp-web.js` release.

Run the initial catch-up and digest:

```powershell
.\scripts\run-now.ps1
```

## Reviewing candidates

The pipeline creates **draft issues inside the GitHub Project**, not repository issues. A draft contains:

- Suggested title and classification.
- Confidence score.
- Source group.
- Original WhatsApp text.
- Quoted-message context when available.
- Similar messages grouped with the candidate.

For an approved candidate, open its menu, choose **Convert to issue**, and select the private repository created during setup. Archive or delete rejected drafts.

## Manual controls

Run these scripts from PowerShell:

```powershell
# Disable both scheduled tasks and stop a running catch-up.
.\scripts\pause.ps1

# Re-enable scheduling, catch up missed messages, and run immediately.
.\scripts\resume.ps1

# Re-enable scheduling but set the current time as the backlog cutoff.
.\scripts\resume-skip-backlog.ps1

# Catch up immediately and force a review digest even if 48 hours have not passed.
.\scripts\run-now.ps1

# Show task state, message counts, last sync, and last digest.
.\scripts\status.ps1

# Pause the pipeline and unlink its WhatsApp Web session.
.\scripts\disconnect-whatsapp.ps1

# Remove the Windows tasks without deleting configuration or messages.
.\scripts\uninstall-tasks.ps1
```

## Configuration

Local configuration is saved to `config/config.json` and excluded from Git. Useful settings include:

| Setting | Default | Purpose |
|---|---:|---|
| `whatsapp.fetchLimitPerGroup` | 750 | Maximum available messages inspected per group and run. |
| `whatsapp.initialLookbackHours` | 72 | History considered during the first run. |
| `review.digestIntervalHours` | 48 | Time between review batches. |
| `review.archiveIrrelevantAfterDays` | 7 | Local retention for classified noise. |
| `classification.minimumScore` | 3 | Candidate threshold for early-user feedback. |
| `classification.designGroupMinimumScore` | 2 | Lower threshold for the design group. |
| `privacy.includeSenderInDraft` | false | Whether a sender reference may be included. |
| `privacy.redactPhoneNumbers` | true | Removes phone-number patterns before GitHub upload. |

## Local data

Runtime files are kept under `.data/`:

- SQLite message and cursor database.
- Saved WhatsApp Web authentication session.
- Current-process ID while a run is active.

Logs are saved under `logs/`. Both directories are excluded from Git.

The default policy sends only classified candidates to GitHub, hides sender identifiers, and redacts phone-number patterns. Irrelevant messages are deleted locally after seven days.

## Testing

```powershell
npm test
npm run check
```

The automated tests cover feedback classification, duplicate grouping, phone-number redaction, draft formatting, SQLite deduplication, and state transitions.

## Limitations

- `whatsapp-web.js` is unofficial and can require updates when WhatsApp Web changes.
- Version 1.34.7 currently needs the pinned upstream `getChats()` compatibility fix from [wwebjs/whatsapp-web.js#201910](https://github.com/wwebjs/whatsapp-web.js/pull/201910). Setup applies the reviewed commit automatically after each locked dependency installation.
- Its browser automation dependency currently reports upstream archive-extraction audit advisories. The Windows installer mitigates that path by skipping browser downloads and using the installed Chrome or Edge executable.
- History synchronization returns only messages available to the linked web session. Very long offline periods or unusually busy groups can leave gaps.
- The first version intentionally uses local rules instead of a large AI model so that it remains responsive on an 8 GB laptop.
- Images, videos, and voice notes are recorded as message metadata; the first version does not upload their contents to GitHub.
- Renaming a selected WhatsApp group does not break collection because the saved chat ID is used, though the original configured name remains in draft metadata until setup is rerun.

## Troubleshooting

### GitHub Project permission error

Run:

```powershell
gh auth refresh -s project
```

Then retry `npm run setup`.

### WhatsApp session expired

Run `npm run setup` and scan the new QR code. Group and GitHub settings can then be selected again.

### No scheduled run occurred

Run `scripts\status.ps1`. Confirm that both scheduled tasks are enabled and that the project folder has not moved.

### A second pipeline instance is reported

Run `scripts\pause.ps1`, followed by `scripts\resume.ps1`. This clears a stale process and starts a new catch-up.

## Security notes

- GitHub authentication is delegated to GitHub CLI; this project does not write a GitHub token into its configuration.
- WhatsApp authentication data is stored locally and must not be committed or shared.
- Keep the GitHub repository and Project private if they contain early-user feedback.
- Inform group participants that actionable feedback may be preserved in the product tracker.

