# How parallel AI agents should talk to each other (and the bug that proved it)

*Ahmad Ammar · 2026-08-01*

If you run more than one coding agent at a time, you hit a problem nobody has a settled answer for: **how do two agent sessions message each other?** Not the model talking to a tool — two independent sessions, running in parallel, that need to hand off a decision or a result.

The obvious channel is a human relaying copy-paste. I spent a day watching that fail in two specific ways, then replaced it, then found a bug in the replacement that is the best argument for the whole approach. Here's the pattern and the evidence.

## Why the human relay fails

It fails for two reasons that have nothing to do with typos.

**1. It loses provenance.** When a message arrives as pasted text, the receiver cannot prove who wrote it. That matters more than it sounds. Two messages reached a session this way in one day: one **asserted a state that had never happened** ("you enabled X" — to a session that had done no such thing), and one reported **CI green on a commit that was already two commits stale**. Both were caught. They were caught *only because the receiver independently checked* — the message itself carried zero evidence.

**2. It looks exactly like a prompt injection.** "Read this and do it" is the shape of an attack. A well-behaved agent should be suspicious of instructions with no verifiable origin — which means the safe agent and the useful relay are in direct conflict.

## The pattern: a committed file + a pointer + a provenance check

The fix is to stop sending *content* and start sending a *reference to a committed artifact*:

1. The sender writes the message to a file in a path it owns and **commits it**.
2. The sender passes one plain-text line: `Read <path> and follow it.`
3. Before acting, the receiver checks the origin:

```bash
git log -1 --format='%an %ci' -- path/to/message.md
```

Now the message has an author, a timestamp, and an immutable history. "Read this and do it" stops being injection-shaped because the receiver can verify the sender before it acts — and can refuse anything whose provenance it doesn't like. Refusal becomes a *feature*, not a failure.

Add a naming convention so a session can find its own mailbox — `message-to-<recipient>-<topic>-<date>.md` — and a tiny script that lists the open ones at session start. Round-trip on this channel that day: a message sent, read, acted on, and replied to in **nine minutes**, with a verifiable trail at every step.

## The bug that proves the point

Here's the part I didn't plan. I wrote the "list my open messages" tool. It ran, and it reported **`0 open`** — while two real messages sat unread. The tool hid its own inbox.

The cause was one line. Each message carries a status, and the closer looks like this:

```js
// BROKEN: "done" appears in the boilerplate of every OPEN message too
function isClosed(statusLine) {
  return /\bdone\b/i.test(statusLine);
}
```

Every *open* message's status line reads: `STATUS: open · Set to "done" after you consume it.` The word `done` is right there in the instructions — so the check marked open messages closed and filtered them out. The fix reads the **value**, not the line:

```js
// FIXED: read only the token right after STATUS:
function isClosed(statusLine) {
  const m = statusLine.match(/STATUS:\s*\**\s*([a-z-]+)/i);
  return (m ? m[1].toLowerCase() : "") === "done";
}
```

The lesson isn't "parse carefully." It's this: **a green result reads as "safe" when it often means "the check can't see."** Before you trust any all-clear — a gate, a linter, an inbox that says empty — prove the red can appear. One test would have caught it:

```js
// negative control: an OPEN message whose text also contains "done" must still surface
assert(isClosed('STATUS: open · Set to "done" after you consume') === false);
```

My synthetic test fixture used a clean status line, so it never contained the trap. The real message did. That is the whole case for dogfooding on real inputs, and for writing the test that tries to make green turn red *before* you rely on it.

## Two things to take

- **Provenance beats convenience.** A message you can't attribute is a message you can't trust — build the channel so every hop is verifiable, and let the receiver refuse.
- **Never trust a green you haven't tried to turn red.** The most dangerous failure isn't the check that fails loudly; it's the one that passes while blind. My inbox tool passed while blind to its own inbox.
