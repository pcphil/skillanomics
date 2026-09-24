# Web Lenses

Detect by web framework markers: `package.json` with React, Vue, Svelte, Next, or similar; HTML templates; route or page directories. Cite `file:line` for every hit.

## Feedback and Loading States

- Async actions (fetch, submit) with no spinner, skeleton, or disabled button.
- Success paths with no confirmation (toast, inline message).
- Buttons that can be double-submitted.

## Error Recovery

- Catch blocks that show nothing, or a generic "Something went wrong".
- Failed requests with no retry option.
- Form errors shown far from the field, or after clearing the user's input.

## Empty States

- Lists or tables that render blank when there is no data.
- Search with no results and no suggestion.

## Shortcuts and Speed

- Frequent actions with no keyboard shortcut.
- Forms without sensible autofocus, tab order, or Enter-to-submit.
- Missing bulk actions on repeated tasks.

## Undo and Safety

- Destructive actions with no confirm or undo.
- Unsaved edits lost on navigation.

## Remembered State

- Filters, sort order, tab, or theme reset on reload.
- Long forms with no draft saving.
- Defaults that ignore the user's last choice.

## Accessibility

- Images without alt text, inputs without labels.
- Interactive elements that are not keyboard reachable.
- Focus lost after modal close or route change.
- Color-only status indicators.

## Onboarding

- First-run screens with no guidance.
- Jargon without tooltips or help text.
