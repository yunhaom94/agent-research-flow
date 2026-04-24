```
{project}/Notes/
├── Ideas.md           # Central idea tracking (piloting idea, main ideas, minor ideas, foundations)
├── Notes.md           # General notes, brainstorm, unorganized thoughts
├── Plans.md           # TODOs, paper outline, implementation plan
└── Literatures/
    ├── Literatures.md # Summaries and tracking of reviewed papers
    └── *.md           # Full-text Markdown conversions of individual papers
{project}/Paper/ # Latex source for the paper write-up
{project}/Code/ # Code implementations, experiments, etc. This could be another repo
{project}/Data/ # Experimental data, results, etc. 
```

**These files mix human and agent writing.** The user adds raw notes (bullet
fragments, shorthand, question marks as placeholders) alongside more structured agent summaries.
Don't assume polished sections are more important than rough fragments — the rough bits are often
the user's latest thinking. Preserve the user's raw voice; don't rewrite their shorthand into
formal prose unless asked. **ONLY** write to the Notes files after the user permits.