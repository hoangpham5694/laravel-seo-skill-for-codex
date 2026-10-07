# Laravel SEO Codex Skill

This package converts the Laravel SEO implementation guide into a Codex skill.

## Contents

```text
laravel-seo-codex-skill/
├── SKILL.md
├── README.md
└── references/
    └── implementation-guide.md
```

`SKILL.md` is the entrypoint. The detailed implementation guide is kept under `references/` so Codex can load it only when deeper examples are needed.

## Install

Copy the entire `laravel-seo-codex-skill` directory into the skills directory used by your Codex environment, keeping the folder structure intact.

After installation, use the skill for requests such as:

- review SEO implementation in this Laravel project;
- add canonical/Open Graph/Twitter metadata;
- refactor `$meta` arrays into `SeoData` + `SeoFactory`;
- add Article JSON-LD;
- verify pagination canonical and staging noindex;
- add SEO feature tests.
