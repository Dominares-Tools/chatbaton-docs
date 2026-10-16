# Changelog

## 1.0.1 (2026-10-04)

Fixes from a review of 1.0.0.

- Your own plan or advice about a manager ("I'll talk to the manager tomorrow", "you should talk to
  your manager") is no longer a request for a person; the manager and supervisor asks now get the
  same plan guard as the owner ask.
- `never_respond` allows an adverb: "nobody ever answers me", "no one even replies".
- A reply that promises a follow-up (`never_respond`) when nobody knows how to reach the visitor now
  also shows the card (`Decision.handoff_reason == "follow_up"`) and ends by asking for their
  details. New `has_contact` argument on `Engine.decide` and `--has-contact` on `classify`;
  `Decision.handoff_reason` and `Alert.details` ("No contact yet") are new. The reference server
  passes it from the cards it has stored. Use `decision.handoff_reason` as the card's reason.
- `payment_issue` no longer fires on pre-sale billing questions ("how much is a monthly
  subscription?", "does the plan auto-renew?", "is autopay available?"); auto-renew and autopay
  count only when the visitor wants them stopped or says they charged. "I want a refund" and
  "refund please" now fire.
- `buying_intent` no longer fires on bare "order", "purchase" or "upgrade" ("where is my order?").
- README: the examples in the kinds table and the list of messages that are not requests are now
  run through `Engine.decide` by the tests; the Limits section lists known odd firings.
- Tests: the brand scan reads the git file list, so an untracked virtual environment cannot fail
  it; direct tests for the reply builders, `load_triggers_full`, `Decision`, `make_handler`, `serve`.

## 1.0.0 (2026-10-04)

First release.

- `chatbaton.detect`: no-model detection of a visitor message that needs a person (`live_person`,
  `team_escalation`, `hostile`, `never_respond`, `payment_issue`), with the owner ask, the bare
  "agent please" ask, identity questions left to the model, and a `Lexicon` for the words that depend
  on your business.
- `chatbaton.triggers`: operator-editable rules in one JSON file (reply mode `canned_reply`,
  `alert_only` or `pass_through` per kind, your own reply text, extra and veto phrases, custom
  triggers), safe defaults for a missing or corrupt file, a backup before every write.
- `chatbaton.engine`: `Engine.decide(text)` ties detection, rules, replies and an `on_alert` hook
  together; also finds an unanswered requirement question, a trust crisis across turns and buying
  intent, with a per-session cooldown on the two silent ones.
- `chatbaton.takeover`: the live takeover store: a lease that expires without a heartbeat, hand back,
  one line in the bot's voice, an announcement you can edit.
- `chatbaton.handoff`: the "talk to a person" card: validation, the confirmation, and a promise store
  that repeats a promise only while it is still true.
- `chatbaton.signals`, `chatbaton.trust`, `chatbaton.requirements`: the pure sensors behind it.
- `chatbaton.demo` and `chatbaton/static/widget.js`: a reference server (standard library only) and
  the browser widget for the visitor side, plus an operator page.
- The `chatbaton` command: `classify`, `triggers`, `takeover`, `demo`.
