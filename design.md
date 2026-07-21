# Figma Design Specification

The following describes the intended look‑and‑feel of the **Chatting** app. The actual Figma file can be created by a designer using these guidelines.

## General Style
- **Color Palette**: Primary blue `#1E88E5`, secondary teal `#008080`, accent orange `#FF6D00`. Backgrounds use light gray `#FAFAFA` and dark mode uses `#121212`.
- **Typography**: Inter – 16 px body, 20 px headings. Rounded corners (4 px radius).
- **Components**:
  - Buttons: Primary with solid fill, secondary with outline.
  - Inputs: Underlined style in light mode, full border in dark mode.
  - Message bubbles: Sent messages align right, received left; bubble tail on the side of alignment.

## Screens
1. **Login / Sign‑up** – Email + password fields, Google/Apple OAuth buttons.
2. **Chat List** – Sidebar with avatar, name, last message preview, unread badge. Search bar at top.
3. **Chat View** – Scrollable list of messages, sticky input bar bottom. File attachment icon triggers file picker.
4. **Settings** – Toggle dark mode, manage account, log out.

## Interaction Flow
- On first launch, the app shows a welcome modal explaining offline capabilities.
- When sending a message while offline, it queues in IndexedDB and shows a “sending…” status until synced.
- Push notifications are shown when a new message arrives while the tab is not focused.

---
> **Note**: A Figma file can be created by importing this spec or using these color codes and components.
