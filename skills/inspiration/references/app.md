# App Lenses (mobile and desktop)

Detect by markers such as `pubspec.yaml`, Android or iOS project folders, `Package.swift`, Electron or Tauri config. Cite `file:line` for every hit.

## Feedback and Loading States

- Network or disk operations with no progress indicator.
- Actions with no haptic, visual, or message confirmation.

## Offline and Connectivity

- No handling for lost connection (blank screen, crash, endless spinner).
- No cached data shown while refreshing.
- Writes lost when offline instead of queued.

## Error Recovery

- Generic error dialogs with no next step.
- Failed operations with no retry.

## Empty States

- Lists or screens that render blank with no data.
- First-launch screens with no guidance.

## Remembered State

- Scroll position, selected tab, filters, or draft input lost on restart or rotation.
- Settings that reset after update.

## Undo and Safety

- Destructive actions with no confirm or undo (swipe-to-delete without snackbar undo).

## Shortcuts and Speed

- Repeated flows with no quick action, share target, widget, or menu shortcut (desktop: keyboard shortcuts, tray).

## Accessibility

- Missing semantic labels, tiny tap targets, fixed font sizes ignoring system settings.
- Color-only status indicators; no dark mode support where the platform expects it.

## Onboarding and Permissions

- Permission prompts with no explanation of why.
- No tour or hints for non-obvious gestures.
