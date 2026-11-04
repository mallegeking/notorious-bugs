# notorious-bugs

Write-ups of IT problems that took real digging to solve. Each one documents what broke, how it was tracked down, what didn't work, and what finally fixed it.

## Why this repo exists

I'm Priviledge AKA MrPrivii AKA Sticks, a Fachinformatiker für Systemintegration working as an IT system administrator at a university research institute in Germany.

Most of the problems I've learned the most from had no clean answer online. The fix was spread across a forum thread, a vendor blog and an event log, and sometimes it was written in a language I don't speak. One of the bugs here was solved with an article from Microsoft's Japanese support team.

When I finally solve one of these, I want the full story written down in one place. That's for future me, and for the next admin who searches for the same error code at 4 pm on a Friday.

## What each write-up contains

Every entry follows the same structure, so you can jump to the part you need:

- **Summary** of the problem and the fix in a few sentences
- **Symptoms**, including the exact error messages
- **Red herrings** that looked like the cause but weren't
- **Diagnosis**, with the commands I ran and what they showed
- **Root cause**, as far as it could be determined
- **What didn't work**, and why
- **Fix**, step by step
- **Verification**, so you know it actually worked
- **References** to the sources that helped

The write-ups describe what actually happened, dead ends included. If a cause is a guess or only a correlation, the write-up says so.

## Index

| Bug | Platform | Status |
|---|---|---|
| [Windows Security won't open: zeroed Security Health binaries and `CopyError`](windows/security-health-copyerror/README.md) | Windows 11 | Fixed |

## Repo structure

```
notorious-bugs/
├── README.md
├── LICENSE
└── windows/
    └── security-health-copyerror/
        └── README.md
```

Write-ups are grouped by platform. Each bug gets its own folder with a `README.md`, plus screenshots or scripts if they help.

## Before you follow a fix

- **Test first.** These fixes worked in my environment. Yours may differ. Try them on a test machine or take a backup before touching production.
- **Check the current docs.** Commands, update numbers and tools change. The references in each write-up are a starting point, not the final word.
- **Anonymized on purpose.** Hostnames, usernames, domains and anything else that identifies an organization have been removed or replaced.

## Questions or a better fix?

If you've hit the same bug and solved it differently, or found a mistake, open an issue. Corrections are welcome.

## License

The content of this repository is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/). You can share and adapt it for any purpose, as long as you give credit and link back here.
