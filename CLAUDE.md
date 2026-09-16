# CLAUDE.md

repo: https://github.com/mayur0204/Internal-chat-tool.git

last updated 16 Sept, app is demo-ready, all 4 done conditions from Section 4 checked manually against the running app before submitting.

Notes for whoever (or whatever) is reading this repo without the full context. Built this in 2 days off one client note from Renee (ops lead at the startup). The note is the entire spec, no follow up calls, nothing else. It contradicts itself twice and I had to just make a call on both instead of asking her since she said she's unreachable this week. Both calls are documented below, don't second guess them without reading the reasoning first, I already went back and forth on this more than I'd like to admit.

## what this actually is

One shared channel, `#general`. Two roles: member and admin, that's it. No DMs, no reactions, no threads, none of that stuff. The brief specifically said not to add features the note doesn't call for so I didn't, even though half of me wanted to throw in reactions because it's like, 10 lines of code. Didn't do it. Scope creep is scope creep even when it's small.

## stack

Kept it dumb simple on purpose, this is a class project not something going to prod.

- Node + Express, one file basically
- SQLite for storage (chat.db), just users + messages tables
- frontend is plain html/css/js, no react no nothing, one page
- polling every 3 sec instead of websockets. brief said either is fine so went with whatever was less work
- auth is a joke on purpose - you just pick which seeded user you are from a dropdown, no passwords. this is fine for 4 people who trust each other, would NOT do this for anything real

if someone reading this later wants to add a proper db or real login, cool, but that's not what was asked for here so it's out of scope for this submission.

## the two contradictions (this is the part that actually matters for grading)

Renee's note contradicts itself in two spots. Writing the full reasoning here since this is literally 25% of the rubric.

**#1 - deletion.** She says "any user can delete any message" in one paragraph, then two lines later says "only admins can remove inappropriate content... locked down so randoms can't delete stuff to cover for each other." Those can't both be true at the same time. I read the admin-only line as the actual hard requirement because she frames it as a safety thing and explains WHY (so people can't delete evidence of bad behavior) - that's a specific, reasoned requirement. The "any user, any message" line reads more like she's describing team culture/vibe than writing a permission spec. So:

- members can delete their own messages, nothing else
- admins can delete literally anything

Enforced on the server (`DELETE /messages/:id` checks role + author id), not just hidden in the UI, so you can't just hit the API directly and bypass it.

**#2 - editing.** Says messages are permanent, no edits, ever, that's important for accountability. Cool. Then immediately after tells a story about how she typo'd "Tuesday" instead of "Thursday" in #general and just edited the message and it "worked fine" and hopes that's "already covered." It is not covered, on purpose. I'm treating the explicit rule as the actual spec and the anecdote as her assuming her old tool's behavior (probably slack or teams) carries over here. It doesn't. No edit button, no PATCH route, period.

workaround if you typo something: delete it (if it's yours) and repost. yes that's mildly annoying, no I don't think that's a bug, it's literally what "no editing" has to mean if you take it seriously instead of quietly working around it.

honestly could see an argument for edit-with-visible-history as a compromise (a lot of real chat apps do this) but decided against it bc the note is pretty explicit and I didn't want to silently override an explicit written rule based on my own guess about what she'd probably want. that's a decision I'm ready to defend in the viva if asked why not.

## checking this against the Done conditions in section 4

went through all four manually before calling this done:

1. post a message logged in as one user -> open the app as a different member -> message shows up after the next poll. works.
2. logged in as demo_member, deleted my own message fine. tried deleting someone else's message, blocked (button isn't even there but also tested hitting the route directly, server rejects it too).
3. logged in as demo_admin, deleted a message that wasn't mine, worked, no issues.
4. no edit button anywhere in the UI, and there's no PATCH route to hit even if someone tried going around the UI. typo fix = delete + repost as covered above.

permissions table for quick reference:

| action | member | admin |
|---|---|---|
| post | yes | yes |
| delete own msg | yes | yes |
| delete others' msg | no | yes |
| edit (any msg) | no | no |

demo accounts are demo_admin and demo_member, check seed.js if you need to see what gets created.

## file layout

```
server.js       -> express app + all routes
db.js           -> sqlite setup, queries
seed.js         -> resets db, creates demo_admin + a couple members + sample msgs
public/
  index.html
  app.js        -> polling, rendering, delete button logic
  styles.css
docs/
  client-note.md        -> Renee's note, unedited
  contradiction-log.md   -> both contradictions + full reasoning
  requirements.md        -> before/after 1-pager for the writeup
```

## running it

```
git clone https://github.com/mayur0204/Internal-chat-tool.git
cd Internal-chat-tool
npm install
node seed.js
node server.js
```
goes to localhost:3000. run seed.js again anytime to wipe and reset the demo data.

## stuff I didn't build (on purpose, not because I ran out of time... mostly)

- editing, covered above at length
- real auth, no passwords, no hashing, just pick a demo user. don't @ me about this being insecure, it's a 4 person internal tool for a class project
- multiple channels - one channel covers everything the brief actually needs demoed
- pagination on message history, fine for a handful of demo messages, would matter if this were real

## if someone else picks this up later

read the contradiction log before you touch anything related to permissions or message editing, the reasoning matters more than the code here. if Renee ever actually responds to something, the one open question worth asking her is whether "admin" should be something she can assign to people later instead of just being hardcoded in the seed data.

- mayur
