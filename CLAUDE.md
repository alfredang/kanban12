# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file demo/training Kanban app for a fictitious bank's internal IT PMO. The whole app is [index.html](index.html): inline `<style>` and `<script>`, vanilla JS, no framework, no build step, no tests, no package manager. There is no git repo.

**Run it:** double-click `index.html` or run `Start-Process index.html`. It must keep working from `file://`.

## Hard constraints (from the original spec)

- Vanilla HTML/CSS/JS only. No CDN, web fonts, images or other external resources. The only allowed network call is the FormSubmit AJAX endpoint.
- No persistence: no `localStorage`, `sessionStorage`, `indexedDB` or cookies. A refresh resets the board to the seed data, and the UI says so.
- No `alert()` or `confirm()`. Validation errors are inline, and delete uses an inline "Delete? Yes/No" confirm.
- No real UOB logo or trademarks. The header uses a neutral "IT PMO" text wordmark. The `UOB-ITPM-####` ID prefix and the `[UOB IT PMO]` email subject are required by the spec, so keep them, but they live in config constants.
- The user's email address must appear only in `FORMSUBMIT_ENDPOINT`, and nowhere else.
- Avoid `!important`.

## Architecture

- **Single state object:** `state = { tasks, filters, confirmDeleteId, draggingId }`. All UI is derived from it.
- **One render path:** mutations (`addTask`, `moveTask`, `deleteTask`, filter changes) update state and then call `renderBoard()`. It rebuilds the columns, count badges and summary strip, and is the only place that touches card DOM. `renderCard()` returns an HTML string, so every user-supplied value must go through `escapeHtml()`. Toasts use `textContent`.
- **Event delegation:** `bindBoardEvents()` attaches the drag/drop, click and change handlers once on the board, so re-rendering doesn't need to rebind. Drop-target highlight is a class on the column. Keyboard users move cards with the per-card "Move ▸" `<select>`.
- **Add Task:** a `<dialog>` opened with `showModal()`. `validateForm()` produces per-field inline errors. `handleSubmit` is optimistic: the card is added immediately, then `notifyNewTask()` posts to FormSubmit inside try/catch. On failure a warning toast appears and the card stays.
- **Dates:** compared as local `YYYY-MM-DD` strings (`todayISO`, `isOverdue`), never UTC, to avoid off-by-one errors. Seed due dates are computed relative to today, so there is always an overdue task.
- **Config block** at the top of the script: `FORMSUBMIT_ENDPOINT` (placeholder `YOUR_EMAIL@example.com`), `ID_PREFIX`, `SUBJECT_PREFIX`, `TOAST_MS`. A HTML comment above it explains the one-time FormSubmit email activation.
- **Layout:** the board is a 4-column CSS grid that collapses to one column below 768px. Styling uses CSS custom properties on `:root`.
