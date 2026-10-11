# Messaging a teammate's agents

> Published from InternSwarm's code repository, which is not public yet, at 420ba314. Anything about building from source applies there.

Let your agent ask a teammate's agent for something directly, while a person on
each side decides what reaches their own agents.

The scenario this document is for: you and Alice each run InternSwarm on your
own machines. Your agent, working in `api#3`, needs something only Alice's
agent knows. With the hosted relay it can write to her and get an answer back,
sealed on your machine so that only hers can read it. Getting there takes one
invite, one enrollment per machine, and one reading of eight words between the
two of you. After that, the only routine step is deciding about mail that waits
in **Held Mail**.

## What you already have without any of this

- **Sessions on one server already message each other.** Start agent sets that
  up with nothing to configure; see Agent-to-Agent Messaging in
  `docs/operating.md`. This document is about reaching *another person*.
- **Your own machines verify each other.** Two of your machines attached to the
  same app are verified for you, and their mail to a session never waits in
  Held Mail. The eight words below are for somebody else's machine.

## What you need

- **The Team plan.** Messaging between people is a Team feature. On a Personal
  plan, invites are refused and mail between members is refused, while each
  member's own machines still reach each other. The portal's Billing page
  switches plans; a complimentary account is switched by support.
- **Each machine enrolled on the hosted relay**, yours and theirs. A machine
  that is not enrolled cannot be addressed.
- **A channel you already share with your teammate**, such as a call or a chat
  you trust. You use it once per machine, to compare eight words.

## Invite a teammate

An Owner or Admin opens **Members** in the account portal and fills in **Invite
a teammate**: their email address and a role (Member or Admin), then **Send
invite**. The invite waits under Pending invites. Each member and each pending
invite takes a seat; when every seat is taken, the form says so, and Billing
adds seats.

Your teammate gets an email with a link, **good for 7 days**. It opens a
**Join** page where they can enter the name teammates will see, then press
**Join workspace**. The invite is used when they join, not when they open the
page.

The Members table's **Reachable** column tells you whether one of their machines
can be written to. It does not say the machine is online: mail waits at the
relay until it connects, for up to 24 hours.

## Enroll a machine

Do this on every machine that should send or receive, yours and theirs.

1. In Settings, **Share This Mac**: turn on **Enable remote access**, set a
   **Password**, and save. Enrolling is refused until both are done.
2. In Settings, **Server Admin**: open **Choose how this server is reached**
   (**Change how this server is reached**, on a server already reachable),
   pick **Hosted relay**, and press **Connect this device**.
3. Your browser opens the portal's approval page. Check that the code matches
   the one in the app and press **Approve device**. The code is good for
   10 minutes; after that, start again for a new one.
4. Back in the app, issuing the certificate takes up to about 2 minutes. Keep
   the app open until it says **Connected to your hosted relay**.

Teammates see the machine as `device-` followed by six letters and digits. Its
real name appears only in your own portal.

## Find each other

Open Settings, **Peers**. Each teammate with an enrolled machine appears as a
person, with their machines under them, marked **Not verified on this server**.

**Opening Peers reads the member list from the relay, at most once a minute**,
so a teammate who just enrolled shows up when you open it. Someone who joined
but has not enrolled a machine does not appear yet.

## Verify each other with the eight words

Verifying goes both ways, once per machine. You verify Alice's machine so your
mail can be sealed to it; she verifies yours so your mail can be delivered on
her side. One call covers both.

1. Alice opens Settings, **Peers** on her machine. The **This machine** card
   shows eight numbered words. She reads them to you.
2. On yours, open her machine in Peers. Under **Trusted to deliver** you see the
   eight words your server holds for it. **Press These match only if all eight
   match, in order.** If any word differs, stop and ask her.
3. Swap: you read yours, and she confirms them.

Nothing the app or the relay shows can settle this for you. The relay carries
keys and vouches for none of them, which is why the words travel over a channel
you already trust. **Compare the full keys instead** offers the same check on
the complete keys, if you would rather paste than read.

A verification is recorded on the server you made it on, and everyone viewing
that server sees it. Your app copies it to your other attached machines that
list the same keys; a machine it is not attached to verifies on its own.

## Send a message

There is no compose box: you ask your agent, in a session, to write to your
teammate. **Your agent cannot list teammates**, so give it the address.

- **Their email address, such as `alice@example.com`,** reaches every machine of
  theirs you have verified, one sealed copy each. Use it for a first message.
- **A delivered message ends with a reply address,**
  `account/device/session-id`, which reaches that one session. Your agent
  replies to it on its own.

A machine of theirs you have not verified is skipped, and the agent is told
which and why. A send that reaches none of their machines is an error, not a
receipt, and sending again is a second send to every machine, not a retry of the
skipped ones.

If the message is held on their side, your agent is told so, and told not to
wait for a reply. Once they decide, it is told whether the message was delivered
or declined. Mail that never reaches their machine expires at the relay after
24 hours.

In the app, the Navigator's **Agent traffic** bar lists what was sent to
teammates, with each copy's state: held, delivered, declined, withdrawn,
expired, or refused.

## Held Mail

**Mail from another person waits in Held Mail until someone on the receiving
side decides about it.** Until then no agent sees a word of it, and your agents
are told only how many messages are waiting. The exceptions are below, under
Letting mail skip Held Mail.

You can find it in four places:

- the **Held** count in the status bar, highlighted when anything is waiting;
  click it to open Held Mail;
- the **Held mail** bar in the Navigator;
- Settings, **Held Mail**;
- on the phone, the **Held** tab, which does everything except compare words.

**There is no push notification for held mail, and held mail never expires.**
Check the count; a message stays until somebody answers it. The one exception
is a sender who is gone: once their machine is revoked, or they leave the
workspace, the app deletes what that machine sent that is still waiting, along
with anything from it still unread in an agent's inbox. See Removing a machine.

Each card shows who sent it, from which machine, to whom, its first lines, and
why it is waiting. What you can do depends on that reason:

| Action | What it does |
| --- | --- |
| **Verify sender** | Opens that machine in Peers to compare words. Nothing from an unverified machine can be delivered. |
| **Deliver once** | Delivers it into the session it was addressed to. |
| **Deliver to…** | Delivers it into a session you pick on this server. |
| **Deliver and trust…** | Delivers it, and lets that machine's later mail skip Held Mail. See below. |
| **Reject** | Drops it. The sender is told you declined. |
| **Block device** | Asks first, then refuses everything that machine sends from now on, on each machine this view shows a copy on. On the phone that is the one server it is paired with. |

**A block is per machine of yours.** A machine of yours not shown in that view
is not blocked, and a grant there still lets the sender's mail in. To turn away
one message, Reject it instead.

**Unblock** is on the blocked machine's row in Settings, **Peers**, and acts on
the server the picker there has selected: a block made from a message held on
several of your machines is lifted on each of them separately. It takes that
machine's mail again from now on, with the verification it had and any grants
that have not run out. What it sent while blocked was refused and is not kept.

## Letting mail skip Held Mail

**Mail addressed to the person is always held**, whatever you have granted. It
lands on every machine of theirs, and nothing here picks which one should take
it.

Mail addressed to one of your sessions skips Held Mail when its sender's machine
is verified on this server and one of these is true:

- **It is a reply.** Your session wrote to theirs, on that machine, within the
  last 7 days. A conversation your agent starts flows back without triage.
- **A standing grant covers the session.**
- **It is from your own machine.**

There are two ways to make a standing grant:

- **Trust ▾**, on a verified teammate's machine in Peers, trusts the person:
  it writes one grant for each machine of theirs verified on this server at
  that moment. **A machine you verify later is not covered** until you grant
  again. Pick where (one project, or **Every project on this server**) and for
  how long: 1 day, 7 days, 30 days, 90 days, or 1 year.
- **Deliver and trust…**, on a held message, trusts only the machine that sent
  it. Pick **Only this session**, **Its project**, or **Every project on this
  server**, and 1, 7, 30, or 90 days.

A grant holds on this server only. Each machine in Peers lists its grants with
their end dates, and has **Revoke** for each one, or for all of them at once.

## Copies, when someone has several machines

Mail addressed to you arrives as one copy on each machine of yours the sender
has verified, all held. **Answering one copy clears the others.** If those
machines are attached to your app, it clears them as you answer; the sender's
machine also withdraws the rest when it hears your answer.

**A copy that nothing clears stays in Held Mail, with no expiry, until you
Reject it there.** If the app could not clear a copy, it says which machine is
still holding it.

The sender's agent hears the first answer, and hears once more if a later copy
was answered the other way.

## When a teammate's key changes

A machine that presents a different signing key is marked **Key changed —
mail stopped**, and a phone or browser you registered for push is notified. Either they reinstalled, or
somebody is standing in for them. From your side the two look the same, so ask
them over your shared channel. Then press **This is their new key**, which asks
you to compare the eight words again, or **Keep the old key**.

For one of your own machines, the row appears under **Your machines needing
attention**, with **Trust the new key**.

A machine can also go back to **Not verified on this server** with no
notification at all. That is a new sealing key, the one mail is sealed to,
under the same signing key. Until you verify it again, your mail to it is
skipped and its mail to you is held as unverified. Compare the eight words
again, as the first time.

## Removing a machine, and bringing one back

**Revoking is final for that machine's identity.** In the portal, **Devices**,
press **Revoke** on its row. Only the machine's owner can, and there is no
confirmation step. The relay then refuses that machine for good and drops any
undelivered mail it sent. Teammates' apps forget it once the relay has listed it
as gone twice, at least 10 minutes apart, and **delete what it sent that is
still waiting in their Held Mail**, along with anything from it still unread in
an agent's inbox.

To bring a revoked machine back:

1. In Settings, **Server Admin**, open **Advanced** and press **Disconnect from
   relay**, then **Disconnect**.
2. Enroll it again, as in Enroll a machine.

**It comes back as a new device, with a new address, and teammates verify it
again with eight words.** Disconnecting without a revoke is different:
reconnecting gets the same address back, and nobody has to verify anything.

## What the relay can and cannot see

Mail to another person is sealed on your machine so that only their verified
machines can read it. The relay carries it between machines and cannot read it.
The relay still sees who is writing to whom, when, and how much; see
[the relay trust model](https://github.com/agenticfabriq/internswarm/blob/main/docs/relay-trust-model.md) for what it sees and stores.

The portal shows machines by their labels only. Keys and fingerprints are
compared in the app, never in the portal.
