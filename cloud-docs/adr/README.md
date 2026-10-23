# Architecture decision records

An ADR records one significant decision: the context, the options, the
choice and its consequences. ADRs are immutable once accepted. To change
a decision, write a new ADR that supersedes the old one and update the
old one's status line only.

## Process

1. Copy `0000-template.md` to `NNNN-short-title.md` using the next number.
2. Open a pull request with status **Proposed**.
3. Add a row to the index below in the same commit.
4. Merge with status **Accepted** once decided. An ADR may also be merged as **Proposed** to record an
   open decision; its status line is updated in a later pull request when it is accepted or rejected.

## Index

| ADR | Title | Status | Date |
| --- | --- | --- | --- |
| [0001](0001-use-github-for-source-control-and-ci.md) | Use GitHub for source control, reviews and CI | Accepted | 2026-10-23 |
