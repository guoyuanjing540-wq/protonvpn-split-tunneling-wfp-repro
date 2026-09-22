# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This is a documentation-only repository. There is no source code, build system, or test suite — the repository consists of a single `README.md` documenting a user-reported bug reproduction: a Proton VPN split-tunneling issue where an excluded process (`com.vortex.helper.exe`, a Mihomo/Clash-based proxy helper) loses outbound connectivity (`WSAEACCES` / socket error `10013`) after Proton VPN connects, along with WFP (Windows Filtering Platform) evidence and a recovery sequence.

The root cause is explicitly unconfirmed. Do not state or imply a definitive root cause when editing this content — the README is careful to frame observations as unconfirmed, and edits should preserve that framing.

## Working in this repository

There are no build, lint, or test commands — changes are edits to `README.md` (and any future Markdown/log evidence files). When editing:

- Preserve the bilingual (English/Chinese) sections; keep both languages in sync if updating either.
- Keep the distinction between confirmed observations (exact error codes, WFP event IDs, filter names) and speculative interpretation.
- WFP `FilterRTID` values are noted as unstable across sessions — don't treat specific runtime IDs as stable identifiers.
