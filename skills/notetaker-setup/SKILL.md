---
name: notetaker-setup
description: |
  Sets up monday AI Notetaker and changes which meetings it joins automatically.
  Checks whether Notetaker is ready, gets a calendar connected, and turns on
  auto-join. Use when user asks to "set up Notetaker", "turn on Notetaker",
  "have Notetaker join my meetings", "connect my calendar to Notetaker",
  "what are my Notetaker settings", "is Notetaker set up", "change auto-join",
  "stop Notetaker joining my meetings", or "only join meetings I organize".
  Don't use for recording or skipping a single meeting, or for reading meeting summaries and transcripts.
license: MIT
metadata:
  author: monday.com
  version: "0.6.0"
---

# Notetaker setup

Flow: **Read settings → Act on `status` → Confirm by reading again**.

Every step uses the monday connector. Reads come from `get_user_context` or `all_monday_api`, and the only write is the `update_notetaker_auto_join_settings` mutation through `all_monday_api`.

## Step 1: Read the settings

1. Call `get_user_context`. If its result has a `notetaker` block, use it.
2. Otherwise run this through `all_monday_api`, with nothing else in the same request (other fields are permission-checked and fail for users without access):

```graphql
query {
  notetaker {
    settings {
      status
      has_access
      reason
      calendar_connected
      setup_url
      auto_join { self_hosted invited locked { self_hosted invited } }
    }
  }
}
```

3. If `settings` is null, or the field doesn't exist, Notetaker setup isn't available for this account through the connector yet. Tell the user to set it up in Notetaker in monday, and stop. Don't guess at other queries.

## Step 2: Act on `status`

`status` names the one step to deal with. Handle only that one, then go to Step 3.

### NO_ACCESS
The user can't use Notetaker. Explain why from `reason`, and say who can fix it. Don't retry, and don't change any settings.

| `reason` | Tell the user |
|---|---|
| NOT_ALLOWED_PER_SCOPE | They don't have the Notetaker permission. An account admin can grant it. |
| BLOCKED | An account admin blocked Notetaker for the account or for them. |
| AI_DISABLED | AI features are off for the account. An account admin can turn them on. |

`reason` NO_CREDITS never comes with NO_ACCESS: setup continues as usual. When `reason` is NO_CREDITS and auto-join is on or just turned on, add one sentence that the AI credits are used up, so Notetaker is on hold until they renew next period or an admin adds more.

### NOT_CONNECTED
No calendar is connected.

- **If `calendar_connected` is null,** the check failed. Ask the user whether they've already connected a calendar, instead of telling them to connect one.
- **Otherwise,** don't ask which calendar they use. Call the monday `show_connect_integration` tool twice in the same reply: once with `integrationName` `googleCalendar` and once with `outlookCalendar`. If it isn't loaded yet, search for it and load it first; it is part of the monday connector. Each call shows a connect button in the chat. Say in one sentence to connect the calendar they use. When a widget tells you the connection is done, go to Step 3. Don't give the user a link instead.
- **Only if the monday connector has no `show_connect_integration` tool at all,** give `setup_url` as a markdown link labelled "Set up Notetaker". If that's null too, tell them to open Notetaker in monday.

### AUTO_JOIN_OFF
A calendar is connected, but Notetaker doesn't join any meetings automatically. Ask which meetings it should join, offering exactly these three options with these labels:

| Option | `self_hosted` | `invited` |
|---|---|---|
| All meetings (Recommended) | ALL | ALL |
| Team meetings (internal) | INTERNAL | INTERNAL |
| Meetings I organize | ALL | OFF |

Then run the mutation (see "Changing auto-join"). If the user declines, accept that and don't offer again in this conversation.

### AUTO_JOIN_LOCKED_OFF
An account admin turned auto-join off and locked it. Tell the user that only an admin can change this. Don't call the mutation.

### ACTIVE
Notetaker is set up. Describe the current setup in plain words, using the option label when the values match one of the options above. Only change something if the user asks.

## Changing auto-join

Use this for AUTO_JOIN_OFF, or when the user asks to change auto-join while ACTIVE.

1. If `auto_join.locked.self_hosted` or `auto_join.locked.invited` is true, tell the user an admin controls that setting, and change only the unlocked one.
2. Run this through `all_monday_api`, passing both arguments for the chosen option:

```graphql
mutation {
  update_notetaker_auto_join_settings(self_hosted: ALL, invited: ALL) {
    self_hosted
    invited
    locked { self_hosted invited }
  }
}
```

To stop Notetaker joining meetings, pass `self_hosted: OFF, invited: OFF`.

3. Errors come back by `extensions.code`. Don't retry any of them.

| Code | Tell the user |
|---|---|
| AUTO_JOIN_FIELD_LOCKED | An admin locked the setting named in `extensions.field`. |
| NOTETAKER_SETTINGS_DISABLED | This isn't available for their account yet. Set it up in Notetaker in monday. |

## Step 3: Confirm

Read the settings again with the query from Step 1. Tell the user the result in one or two sentences, for example: "Notetaker is set up and will join all your meetings." If `status` has moved to another step, go back to Step 2 for it. Do this at most once per step, so the conversation doesn't loop.

## Rules

- Never show raw link URLs as plain text; always use the labelled markdown link.
- Act only on what `status` says. Don't infer the setup from other fields.
- Use the option labels exactly as written. Don't show the user raw enum values like INTERNAL.
- Never retry a failed mutation, and never work around an admin lock.
- Once Notetaker is ACTIVE, any Notetaker meeting tools the connector has (for example listing upcoming meetings, recording one, or skipping one) can take over. They aren't part of this skill.
