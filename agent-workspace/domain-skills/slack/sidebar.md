# Slack: Read unread state from the sidebar

URL: `https://app.slack.com/client/<TEAM_ID>/<CHANNEL_ID>`

Verified live on 2026-09-20, except where marked **unverified** below.

## Rule: never open a channel to check it

Opening a channel clears its unread marker. A passive watcher must read state from the **sidebar** only and never click a row.

## Do this first

```python
rows = js("""(() => {
  const P = 'data-qa-channel-sidebar-';
  return [...document.querySelectorAll('[' + P + 'channel-id]')].map(el => {
    const name = el.querySelector('[data-qa^="channel_sidebar_name_"]');
    return {
      channel_id: el.getAttribute(P + 'channel-id'),
      type: el.getAttribute(P + 'channel-type'),
      name: name ? name.textContent.trim() : null,
      muted: el.getAttribute(P + 'channel-is-muted'),
      selected: el.getAttribute(P + 'channel-is-selected'),
      starred: el.getAttribute(P + 'is-starred'),
      is_you: el.getAttribute(P + 'is-you'),
      section_id: el.getAttribute(P + 'channel-section-id'),
      shared_type: el.getAttribute(P + 'shared-type'),
    };
  });
})()""")
```

## Stable selectors

Every sidebar row has `data-qa-channel-sidebar-channel-id`. These sibling attributes live on the same element:

| Attribute | Meaning |
| --- | --- |
| `data-qa-channel-sidebar-channel-type` | `channel`, `private`, `im` or `mpim` |
| `data-qa-channel-sidebar-channel-is-muted` | muted flag |
| `data-qa-channel-sidebar-channel-is-selected` | currently open row |
| `data-qa-channel-sidebar-is-starred` | starred flag |
| `data-qa-channel-sidebar-is-you` | the user's own DM row |
| `data-qa-channel-sidebar-channel-section-id` | sidebar section the row belongs to |
| `data-qa-channel-sidebar-shared-type` | present only on Slack Connect / externally shared channels |

- Name: descendant `[data-qa^="channel_sidebar_name_"]`.
- Selected row class: `p-channel_sidebar__channel--selected`.

`data-qa-channel-sidebar-shared-type` is a free internal-vs-external signal: if the attribute is present, the channel includes people outside the workspace. Check for presence, not for a specific value.

## Unread and mention markers (unverified)

No unread channel existed when this was written, so the following are **expected but NOT verified**. Confirm them against a live unread before relying on them, then update this file:

- Unread rows get class `p-channel_sidebar__channel--unread`.
- Mention badge: `[data-qa="mention_badge"]` or `.c-mention_badge`.

## URL shape

```text
https://app.slack.com/client/<TEAM_ID>/<CHANNEL_ID>
```

## Logged out

Either of these means there is no session. Treat both as `logged_out`:

- `app.slack.com/workspace-signin`
- `slack.com/signin`

## Multiple workspaces

Use one tab per workspace, identified by the `<TEAM_ID>` in its URL.

## Traps

- Clicking a row to inspect it clears the unread marker you were trying to observe.
- The unread class and mention badge selectors above are unverified; do not treat their absence as proof of "nothing unread" until confirmed.
