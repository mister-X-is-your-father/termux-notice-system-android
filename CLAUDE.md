# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

Documentation-only knowledge base for sending Android notifications from Claude Code running inside Termux. Covers Claude Code sandbox bypass, Termux:API APK signing constraints, `termux-notification` usage, and remote notification via reverse SSH tunnels.

## Repository Structure

Single `README.md` in Japanese. No source code, build system, or tests.

## Key Technical Context

- **Target environment**: Termux on Android 15 (SDK 35) with Claude Code (Opus 4.6)
- **Claude Code sandbox bypass**: Use `setsid bash -c '...'` to escape the sandbox that blocks commands like `termux-notification`
- **APK signing constraint**: Termux and Termux:API must share the same signing key (F-Droid with F-Droid, GitHub with GitHub) due to `android:sharedUserId`
- **Remote notification path**: Reverse SSH tunnel (`ssh -R`) from Termux to remote server, then `ssh -p 28022 localhost 'termux-notification ...'` back through the tunnel
- **Android 15 quirk**: Writable DEX files are rejected; `chmod 444` required before `dalvikvm` execution

## Writing Conventions

- Documentation is written in Japanese
- Commit messages are in English, using conventional commit style (`docs:`, `feat:`, etc.)
