# Osysharp.Mail

Mail an app sends — order confirmations, reset links, replies to a customer — through one call, whichever company
carries it. The message is written in the same commit as whatever it is about, delivered after that commit, retried
when the carrier is down, and **held** (kept, with the reason) when the app has no carrier yet — so an app works from
its first minute, and nothing it meant to send is lost.

```osy
SendMail(new MailMessage { To = [order.CustomerEmail], Subject = "Your order is on its way", Text = "…" }, $"shipped-{order.Id}");
```

## Use it

```osy
// app.osy
use Osysharp.Identity@0;
use Osysharp.Mail@0;
use Osysharp.Workflow;
```

```osy
// Who may send is the app's to say — nobody else can put mail in the outbox:
partial entity OutgoingMail { security { allow create, read when IsClerk; } }

// What carries it — here Resend (the Osysharp.Mail.Resend package). No sender: mail is held.
app.Mail = new MailSetup {
  Sender = new ResendSender { ApiKey = Security.IsSecretSet("ResendApiKey") ? Secret.ResendApiKey : "" },
  From = "Acme <noreply@acme.com>",
};
```

## What it does

- **`SendMail(message, idempotencyKey?)`** writes a row to the outbox (`OutgoingMail`) and answers it. The same key
  twice is one message, so a handler that runs twice mails once.
- **`SendRendered(to, rendered)`** sends an `[Email]` component's answer (Osysharp.EmailTemplates).
- **`ReplyTo(original, text)`** makes a reply in the original's conversation.
- **Threading is first-class.** Every message has its `MessageId` (minted when the row is written, so a reply can
  thread to it at once), `InReplyTo` and `References`, and a sender sends them as the mail's own headers.
- **The outbox is the delivery record**: `Queued`, then `Sent` (when, through which sender, the carrier's id) or
  `Held` (and why).
- **Nobody reads the outbox who was not granted — not even the recipient.** A reset link sits in those rows; reading
  it there would skip the mailbox it was sent to.

## Senders

A sender is anything implementing `IMailSender` — `Key`, `IsReady()`, and `SendMail(message, idempotencyKey)` — so the
app changes carrier by changing one setting, and nothing that sends changes with it.

- **[Osysharp.Mail.Resend](https://osyrin.com/templates/kits/mail-resend/)** — Resend, the transactional provider.
- **Your own** — a class implementing `IMailSender` over `Http.*` for any provider, with its host declared in your
  package's manifest.

## Mailboxes

A mailbox is anything implementing `IMailbox` — a Google Workspace / Gmail inbox, a Microsoft 365 one, or any IMAP
account — named in `app.Mail.Mailboxes`. The app is not a mail server: mail arrives where it always did, and the app
watches that inbox.

- **Reading.** [Osysharp.Mail.Receiving](https://osyrin.com/templates/kits/mail-receiving/) watches each mailbox and
  hands what arrives over in the same `MailMessage` shape — from, to, cc, subject, text, HTML, attachments, headers,
  message id, thread references — so it is matched to its conversation by `InReplyTo`/`References`.
- **Sending as the mailbox.** A message whose `From` is a connected mailbox's address goes out through that mailbox —
  a reply to a customer leaves from `support@acme.com`, threaded, and sits in its Sent folder.
- **Providers:** [Osysharp.Mail.Gmail](https://osyrin.com/templates/kits/mail-gmail/),
  [Osysharp.Mail.Microsoft365](https://osyrin.com/templates/kits/mail-microsoft365/) and
  [Osysharp.Mail.Imap](https://osyrin.com/templates/kits/mail-imap/).
