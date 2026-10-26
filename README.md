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

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.Mail;
using MailKitTests;

[Test(Desk)]
void a_message_goes_after_the_commit_and_is_recorded_as_sent() {
  runas (Ada) {
    var mail = Notify("cus@example.test", "Your ticket", "We are on it.");
    UnitOfWork.Commit();
    Workflow.Settle(mail);
  }
  runas (Ada) {
    var sent = OutgoingMail.Single(m => m.Subject == "Your ticket");
    Assert.Equal(MailStatus.Sent, sent.Status);
    Assert.Equal("recording", sent.SentWith);
    Assert.Equal("carried-Your ticket", sent.ProviderId);
    var carried = Carried.Single(c => c.Subject == "Your ticket");
    Assert.Equal("cus@example.test", carried.To);
    Assert.Equal(sent.MessageId, carried.MessageId);
    Assert.True(carried.MessageId.EndsWith("@acme.test>"));
  }
}
```

From [`tests/mail.test.osy`](tests/mail.test.osy).

```osy
using Osysharp.Mail;

[Test]
void gmail_html_keeps_the_reply_and_quotes_the_gmail_quote() {
  var mail = Received(
    "Thanks, that fixed it!\n\nOn Thu, Oct 1, 2026 at 9:30 AM Acme Support <support@acme.test> wrote:\n\n> Please try restarting the export.",
    """<div dir="ltr">Thanks, that fixed it!<br></div><br><div class="gmail_quote gmail_quote_container"><div dir="ltr" class="gmail_attr">On Thu, Oct 1, 2026 at 9:30 AM Acme Support &lt;<a href="mailto:support@acme.test">support@acme.test</a>&gt; wrote:<br></div><blockquote class="gmail_quote" style="margin:0px 0px 0px 0.8ex;border-left:1px solid rgb(204,204,204);padding-left:1ex"><div>Please try restarting the export.</div></blockquote></div>""");
  var split = SplitReply(mail.Text, mail.Html);
  Assert.True(split.Sure);
  Assert.Equal("Thanks, that fixed it!", split.New);
  Assert.Contains("Please try restarting the export.", split.Quoted);
  Assert.Contains("wrote:", split.Quoted);
}
```

From [`tests/replies.test.osy`](tests/replies.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Mail;
using MailKitTests;

[Test(Desk)]
void a_reply_threads_to_the_message_it_answers() {
  runas (Ada) {
    var first = Notify("cus@example.test", "Your ticket", "We are on it.");
    UnitOfWork.Commit();
    var second = Answer(first, "Fixed now.");
    UnitOfWork.Commit();
    Workflow.Settle(second);
  }
  runas (Ada) {
    var first = OutgoingMail.Single(m => m.Subject == "Your ticket");
    var carried = Carried.Single(c => c.Subject == "Re: Your ticket");
    Assert.Equal(first.MessageId, carried.InReplyTo);
    Assert.Equal(first.MessageId, carried.References);
  }
}
```

From [`tests/mail.test.osy`](tests/mail.test.osy).

## Learn more

- [The Osysharp.Mail reference](https://osysharp.com/reference/ui/mail-kit/)
- [Osysharp.Mail in the package catalogue](https://osysharp.com/packages/#osysharp-mail)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
