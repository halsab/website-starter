# Website Starter

This repository contains a foundation specification, not application source code. Use it from a new or empty repository to create a neutral baseline; the only required input is `PROJECT_NAME`, from which the coding agent derives all package and database names.

## Workflow

1. Create or open an empty repository.
2. Copy the initialization prompt below, replace `<project-name>`, and run it in Codex.
3. Review the generated foundation, then begin product development.

## Initialization prompt

```text
Initialize this repository using the foundation specification from:
https://github.com/halsab/website-starter

PROJECT_NAME: <project-name>

Read SPEC.md from the starter repository and implement the complete foundation in this repository. Derive all package and database identifiers from PROJECT_NAME according to SPEC.md. Follow SPEC.md exactly, keep the result product-neutral and repository content in English, use pragmatic TDD, run the complete verification defined by SPEC.md, create the initial commit only after verification passes, and do not push.
```

The complete baseline and safety requirements are defined in [SPEC.md](SPEC.md).
