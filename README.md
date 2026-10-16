# chatbaton — Let a person take over the bot's live chat

**Paid tool: $149 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/chatbaton**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

chatbaton spots a visitor who needs a person without calling a model, lets someone on your team take over the live bot chat and hand it back, and keeps the promises it makes honest. It comes with the browser widget, the "talk to a person" card and a reference server.

## Get it

Buy chatbaton at https://dominares.org/tools/chatbaton: one payment of $149, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python package and the chatbaton command, the browser widget, a reference server with an operator page, and the full test suite.

Requirements: Python 3.10 or newer, no runtime dependencies. Node is optional, for the widget's own tests.

This page documents version 1.0.1. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

### Try the whole flow

```bash
chatbaton demo --port 8730
```

It prints two addresses. Open the first (the visitor chat) and type `I want to talk to a real person`: the bot replies, and the "talk to a person" card appears. Open the second (the operator page, which carries a random token in the address) in another tab, choose the chat, press **Take over** and type a reply. The visitor sees "You are now connected with ..." and your words as a labelled person; their next message is held, not answered by the bot. **Hand back** and the visitor sees "The assistant is back". State is kept in a folder (`chatbaton-state/`, or `--state-dir`).

### Use it in your own chat

```python
from chatbaton import Engine, Lexicon, Voice, owner_names_from

alerts = []

engine = Engine(
    "chatbaton-state/triggers.json",                   # the operator's rules; None means the shipped defaults
    lexicon=Lexicon(org_names=("Acme",)),              # what visitors call you
    owner_names=owner_names_from("Robert Smith"),      # so "can I talk to Robert" counts
    voice=Voice(org_name="Acme", email="billing@example.com", followup="within 24 hours"),
    on_alert=alerts.append,                            # yours: send an email, a chat message, a page
)

decision = engine.decide("I want to talk to a real person", session_id="sess_abc123")
assert decision.kind == "live_person"
assert decision.force_reply and "Acme" in decision.reply_text   # send this instead of calling the model
assert decision.handoff                                        # also send the card event (see below)
assert alerts[0].kind == "live_person"                         # on_alert was called

nobody = engine.decide("nobody ever answers me", session_id="sess_abc123")
assert nobody.kind == "never_respond" and nobody.handoff_reason == "follow_up"   # the reply promises a follow-up
assert engine.decide("nobody ever answers me", has_contact=True).handoff is False  # you already know where to reach them
```

Your chat route does four things, in this order, for every visitor message:

1. **Is a person holding this chat?** `takeover.live_takeover_active(live_path, session_id)`. If yes, record the message for the operator, answer with the event `{"type": "held", "message": takeover.HELD_MESSAGE}` and do not call the model.
2. **Decide.** `engine.decide(text, session_id=..., messages=history, knowledge_text=..., filed=handoff.filed_promise(promises_path, session_id), when="today, before 6 PM", has_contact=...)`. Pass `has_contact=True` when you already hold an email or phone number for this visitor. If `decision.force_reply`, send `decision.reply_text` and skip the model; if not, call your model as usual (`decision.send_alert` has already called `on_alert`).
3. **Card.** If `decision.handoff` and the conversation has no live promise, also send `handoff.handoff_event(decision.handoff_reason, when=..., phone=...)`. The widget draws the card from it. `handoff` is also set, with the reason `follow_up`, when the reply promises that someone will get back to the visitor (`never_respond`) and you do not know how to reach them: the reply then ends by asking them to leave their details in the card, so the promise is never made to someone nobody can reach. Alerts carry a "No contact yet" row in `Alert.details` in that case.
4. **Answer the card.** Your `POST /api/handoff` calls `handoff.validate(...)` (a `HandoffError` message is written for the visitor), stores the details wherever you keep contacts, then `handoff.record_promise(...)`, and replies `{"ok": true, "confirmation": handoff.confirmation(clean, org=..., when=...)}`. Record the promise only after something durable landed.

And two calls for the visitor's page, which the widget makes for you: `POST /api/presence` answers `{"ok": true, "intervention": takeover.live_takeover_active(...)}`, and `GET /api/live-chat/<session>?since=<n>` answers `takeover.poll_payload(live_path, session_id, since)`. `chatbaton/demo.py` is all of this in one file; read it first.

Your operator screen calls `takeover.set_takeover(path, sid, action="start", operator_label="Sam", operator_account="sam")`, `action="heartbeat"` every few seconds while the person is there, `takeover.append_operator_turn(path, sid, text=..., mode="takeover")` for each thing they say (`mode="say_as_bot"` for a line in the bot's voice), and `action="stop"` to hand back.

The widget, in your page:

```html
<script src="/widget.js"></script>   <!-- served from chatbaton/static/widget.js -->
<script>
  var cb = ChatBaton.attach({ box: document.getElementById('chat'), sessionId: 'sess_abc123' });
  // for every event your chat stream sends:  if (cb.handleEvent(evt)) return;
</script>
```

Session ids must start with `sess_`; the widget does nothing for any other id, so a page that has not made one yet stays quiet.

### The kinds

| kind | fires when | default mode | alerts |
|---|---|---|---|
| `live_person` | asks for a person, an agent, the owner or manager, "call me back", "get me someone" | canned reply and the card | yes |
| `team_escalation` | wants a manager, a formal complaint, a review-site report, a legal threat aimed at you | canned reply and the card | yes |
| `hostile` | abuse aimed at you | canned reply | yes |
| `never_respond` | "nobody ever answers me", "still waiting for a reply" | canned reply, and the card unless you already hold a way to reach them (see step 3) | yes |
| `payment_issue` | a repeat charge, "I want a refund", "cancel my subscription", "money back", "turn off auto-renew" (not a question about price or whether a plan auto-renews) | canned reply if you gave an inbox, else the model answers | yes |
| `requirement` | asks what is required (ID, a deposit, an appointment) and the text you pass as `knowledge_text` never says | canned reply and the card | yes |
| `trust_crisis` | across the last few visitor turns: a contradiction, "I will be reporting this", "how can I trust you" | alert only | yes, once per session per three minutes |
| `buying_intent` | pricing, a quote, "sign me up", a demo, "I want to buy" (not a question about an order you already placed) | alert only | yes, once per session per three minutes |

A message that matches nothing is `none`: the model answers.

Messages that are deliberately not requests (each one is run through `Engine.decide` by the tests and must come back `none`): "speak to a manager at my bank", "I will sue my landlord", "human resources", "who is my representative?", "the owner role", "I'll talk to the manager tomorrow", "you should talk to your manager", "do not contact me", "where is my order?".

Billing questions before a sale are not payment problems: "does the plan auto-renew?" is `none` and "how much is a monthly subscription?" is `buying_intent`, not `payment_issue`.

### Rules the operator edits

`triggers.json` (see `examples/triggers.example.json`):

```json
{"builtins": {"hostile": {"mode": "alert_only"},
              "live_person": {"reply": "A person will pick this up.", "extra_phrases": ["ring the bell"]}},
 "custom": [{"label": "Parcel tracking", "phrases": ["where is my parcel"],
             "mode": "canned_reply", "reply": "Your tracking link was emailed when it shipped.", "alert": false}]}
```

Per built-in kind: `enabled`, `mode` (`canned_reply`, `alert_only`, `pass_through`), `reply`, `alert`, `extra_phrases`, `exclude_phrases`. `pass_through` lets the model answer and tells nobody. The built-in detector's own matches alert whatever `alert` says; `alert` governs phrases you add. To silence a kind, set `enabled` to false or the mode to `pass_through`. Phrases match whole words, ignoring case.

The file is read on every message, so a save is live on the next one. `triggers.save_triggers` copies the previous file into `backups/triggers/` first (the newest 40 are kept), writes atomically under a lock, and raises `ValueError` for a bad mode without writing. `triggers.reset_trigger` undoes one row or all of them. From the command line:

```bash
chatbaton triggers show --state-dir ./state
chatbaton triggers set hostile --mode alert_only --add-phrase "you clown" --state-dir ./state
chatbaton triggers reset hostile --state-dir ./state
```

### The words that depend on your business

`Lexicon` holds them, and the defaults are neutral:

| field | what it is for |
|---|---|
| `org_names` | what visitors call you: "Acme is a scam" and "a complaint about Acme" count only with it |
| `external_parties` | outside parties a complaint is usually aimed at ("my bank", "my landlord"), so a question about them is not an escalation |
| `topic_nouns` | nouns that turn "the owner" or "your manager" into a thing: "the manager dashboard" |
| `not_a_person_compounds` | words after "human" that make it a topic: "human resources" |

### Options

```
chatbaton classify TEXT... [--rules FILE] [--org NAME]... [--owner "Full Name"] [--email ADDR] [--knowledge FILE] [--has-contact]
chatbaton triggers show|set|reset [KIND] [--state-dir DIR] [--mode M] [--reply T] [--enable|--disable] [--add-phrase P] [--veto-phrase P]
chatbaton takeover start|heartbeat|stop|say|state SESSION [--state-dir DIR] [--label L] [--account A] [--text T] [--as-bot] [--since N]
chatbaton demo [--host 127.0.0.1] [--port 8730] [--state-dir DIR] [--ops-token T] [--org NAME] [--email ADDR]
```

`classify` with no arguments reads one message per line from a pipe. `--has-contact` says you already hold a way to reach the visitor (no card for a follow-up promise). `--knowledge FILE` is what your own search returned for the question: with it, a requirement question the file never answers becomes `requirement`.

### Takeover, in detail

- A takeover lasts `TAKEOVER_TTL_SECONDS` (120) after its last heartbeat. After that `live_takeover_active` is false and the bot answers again. Nothing has to clean up.
- A heartbeat that arrives after the lease lapsed does not revive it; the person must start again.
- A heartbeat that names a different `operator_account` than the one holding the lease does nothing, so a colleague with the chat open cannot keep someone else's lease alive.
- The announcement ("You are now connected with {name}.") is composed once, when the person takes over, from a template you can change; the label is capped at 80 characters, a message at 4,000.
- A stored entry of the wrong shape (a damaged file) reads as "no takeover"; it never turns the visitor's poll into an error.
- `idle_clock_from` helps with "is the visitor still here": while a person holds the chat the visitor may wait quietly, and the person's turns are not visitor messages, so a plain "no message for three minutes" rule would drop a visitor who is only waiting.

### Limits

- **English only.** The patterns are written for English phrasing. Other languages need their own.
- **A keyword and grammar classifier, not a model.** It will miss some real requests and fire on a few odd ones. The design leans toward not firing on a mention ("my representative") because a false handoff is a visible mistake; add `extra_phrases` for what you see missed, `exclude_phrases` for what it should not catch.
- **Odd firings it does not guard against.** "my lawyer said the contract is fine" and "our attorney is reviewing the contract" count as `team_escalation`; "I hate this weather" and "waste of time to fill in forms?" count as `hostile`; "I won't reply to spam" and "the app won't respond" count as `never_respond`. Add the phrases you see to `exclude_phrases` for that kind.
- **A message at a time.** Detection looks at the latest message. Only `trust_crisis` reads earlier turns, and only the ones you pass in `messages`.
- **Files, on one machine.** State is JSON files under a lock (`flock` on macOS and Linux; a Windows fallback is written but has not been tested). That is safe for several workers on one host. A network file system that does not honour locks, or several hosts, need your own store behind the same functions; the lock helpers fail open (they proceed unlocked) where no lock can be taken.
- **The widget polls.** A visitor sees the person's words within about three seconds, not instantly. The poll starts when the presence ping (every 30 seconds) or a held message learns a person has the chat.
- **Alerts are yours.** `on_alert` receives an `Alert`; chatbaton sends no email or message. Its exceptions are caught and logged so a mail outage cannot break a visitor's chat. Silent alerts (`trust_crisis`, `buying_intent`) are limited to one per session per three minutes; without a `session_id` there is nothing to count against and they alert every time.
- **The reference server is a reference.** One shared operator token, no TLS (put it behind your own proxy to leave localhost), a per-conversation limit of three cards an hour and ten cards an hour per address, bodies capped at 64 KB, a stand-in bot. It binds to 127.0.0.1 unless told otherwise. Do not run it as your product.
- **What is stored, and for how long.** The takeover store keeps operator messages (the newest 500 per chat, 2,000 chats), the promise store keeps a promise sentence with no address (the newest 2,000 chats), and the reference server also keeps transcripts and the cards visitors sent, with names and contact details, in the state folder. chatbaton never deletes those files for you; deleting a visitor's data is your job, and the promise store holds none of their contact details by design.
- **No checking of who the visitor is.** The session id is the visitor's only credential, as in any anonymous web chat. Do not put anything private in a takeover turn that the session's holder should not see.

---

Docs mirror of the private repository Dominares-Tools/chatbaton; updated with each release.
