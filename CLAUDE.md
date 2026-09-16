# CLAUDE.md

Repo: https://github.com/mayur0204/Internal-chat-tool.git

Build complete and demo-ready as of 16 September 2026; all Section 4 done conditions have been checked against the running app.

Context for anyone (human or Claude) picking up this repo cold. This is a 2-day build for Renee's
internal team chat tool. The full spec is one forwarded note (see `docs/client-note.md`), and it
contradicts itself in two places. Don't re-litigate those without reading the contradiction log
first — the reasoning is there, not repeated here.

## What this app actually is

Single shared channel (`#general`), team members post messages, everyone sees them. Two roles:
`member` and `admin`. That's it. No DMs, no reactions, no threading, no channel management UI —
none of that was asked for and we're not scope-creeping a 2-day sprint.

## Stack

- Backend: Node + Express, single process
- DB: SQLite (`chat.db`), one `users` table, one `messages` table
- Frontend: plain HTML/CSS/vanilla JS, no framework — the whole UI is one page
- Updates: polling every 3s (`GET /messages?since=`), not websockets. The brief explicitly said
  either is fine and not to over-invest here, so we didn't.
- Auth: dead simple session cookie, seeded users only (no signup flow needed for a 4-person
  internal tool). Seed script creates 1 admin + a few members.

Nothing here needs to be fancier than this. If a future dev is tempted to add Redis or whatever,
they've misread the assignment.

## The two contradictions and how they're actually wired

Renee's note contradicts itself twice. Full writeup with reasoning is in
`docs/contradiction-log.md` — this section is just "what the code does," for reference while
reading the code.

**Deletion.** Note says "any user can delete any message" AND "only admins can remove
inappropriate content." Can't have both. We went with: members can delete *their own* messages
only; admins can delete *any* message. The safety framing in the note ("locked down so randoms
can't just start deleting stuff to cover for each other") is the more specific and more clearly
load-bearing requirement — the "any user, any message" line reads like a culture statement, not a
permission spec. Enforced server-side in `DELETE /messages/:id` — checks `req.session.role` and
`message.author_id` before allowing it, doesn't trust anything the client sends about permissions.

**Editing.** Note says messages are permanent, no editing, ever — then two paragraphs later
describes editing a message in place and asks for that to "already be covered." It isn't, and we
didn't add it. There is no `PATCH /messages/:id` route, full stop. The explicit rule is the actual
requirement (accountability, stated as "important"); the anecdote is Renee describing a habit from
whatever tool she used before, not a feature request. Workaround for typos in this app: delete +
repost, which only works within whatever your delete permissions already are. Yes, that means a
member who typos something has to delete their own message and retype it. That's the intended
behavior, not a bug — see the log for why we didn't just quietly add edit-with-history instead
(short version: the note is explicit enough that silently overriding it felt like the wrong call
for something we can't ask Renee about this week).

If someone wants to revisit this later, it's a product decision, not a technical one — the code
makes it easy to add a PATCH route back in, we just didn't.

## Verifying the Done conditions

These are the four Section 4 checks I verified in the running app.

1. **Posted message shows for all members**
   - Log in as one user and open `#general`.
   - Post a message.
   - Open the chat as another member account.
   - The new message appears for the other member through the normal polling update.

2. **Delete as a regular member**
   - Log in as `demo_member`.
   - Post a message and use its **Delete** button.
   - The member's own message is deleted successfully.
   - Try to delete a message posted by another user. The action is blocked because a regular member cannot delete someone else's message.
   - The server checks the logged-in role and message author before allowing `DELETE /messages/:id`.

3. **Delete as admin**
   - Log in as `demo_admin`.
   - Open the same `#general` channel.
   - Use the **Delete** button on a message posted by another user.
   - The message is deleted successfully because the admin role is allowed to delete any message.

4. **Attempted edit**
   - There is no Edit button in the UI.
   - There is no `PATCH /messages/:id` route.
   - If someone tries to call that edit route directly, it is not available, so the message cannot be edited.
   - For a typo, the intended workaround is to delete the message and repost it, subject to the existing delete permissions.

## Permissions, concretely

| Action | member | admin |
|---|---|---|
| post message | yes | yes |
| delete own message | yes | yes |
| delete someone else's message | no | yes |
| edit any message | no | no |

Both roles are demoable from the seed data — `demo_admin` / `demo_member` (see `seed.js` for
passwords, not repeating them here).

## Layout

```
server.js          # express app, routes
db.js               # sqlite setup + queries
seed.js             # creates demo_admin + a couple members, a few sample messages
public/
  index.html
  app.js            # polling loop, render, delete button logic
  styles.css
docs/
  client-note.md         # the original note, verbatim
  contradiction-log.md    # both conflicts, resolution + reasoning for each
  requirements.md         # before/after 1-pager
```

## Running it

```
git clone https://github.com/mayur0204/Internal-chat-tool.git
cd Internal-chat-tool
npm install
node seed.js     # wipes and reseeds chat.db
node server.js   # localhost:3000
```

## Things intentionally not done

- No message editing (see above, this is a decision not an oversight)
- No password hashing bells and whistles — bcrypt is used but there's no reset flow, no email,
  none of that. Not in scope for 4 internal users.
- No multi-channel support. One room does everything Section 4 of the brief needs.
- No pagination on message history. Fine for a team this size over a 2-day-old chat log; would
  need revisiting if this ever became a real product.

## If you're continuing this after me

Read the contradiction log before touching permissions or the message model. If Renee is ever reachable, the open question worth asking is whether admin should be an assignable role instead of a manually-seeded column.
