# Repository Guidelines

## Project Structure & Module Organization

This repository contains practical documentation for coding agents, not an application. `agent/README.md` is the cross-tool catalog; `agent/claude/`, `agent/codex/`, and `agent/pi/` hold tool-specific introductions and focused guides. Keep new material with its related tool and add or update links in the relevant README. Local configuration and scratch paths such as `.claude/`, `.agent/`, `.agents/`, `.codegraph/`, and `ai_tmp/` are ignored; do not add generated files or private settings.

## Build, Test, and Development Commands

There is no build system or automated test suite. Edit Markdown directly, then use `git diff --check` to catch whitespace errors and `git diff` to review the change. Check relative links and commands manually, including whether examples still match the referenced tool's current behavior.

## Coding Style & Naming Conventions

Use concise Markdown with a single top-level title and nested section headings. Existing guides are primarily in Chinese, so preserve the language of the document you edit. Use descriptive filenames and the existing Pi pattern, such as `配置-模型接入.md`, for focused topics. Label fenced code blocks (`powershell`, `bash`, `json`) and make placeholders obvious. Prefer PowerShell for Windows examples and keep commands copyable.

## Testing Guidelines

No test framework or coverage requirement is configured. For documentation changes, review rendered Markdown, follow changed relative links, and confirm command examples against current documentation or tool output. Call out any command that changes local configuration or system state.

## Commit & Pull Request Guidelines

Recent commits use short imperative subjects such as `Add ...`, `Rewrite ...`, or a topic prefix such as `WSL guide:`. Keep each commit focused. A pull request should summarize the documentation changed and its purpose, list how claims and links were checked, and link a related issue when applicable. Include screenshots only when a rendering or layout change needs visual review.

## Security & Configuration

Never commit real API keys, tokens, personal credentials, or unredacted logs. Use clearly marked placeholders in examples, and keep machine-specific configuration and generated scratch data out of tracked files.
