# Osysharp.Mail.Imap

**Any IMAP mailbox** — a hosting company's mail, a self-hosted server — as a mailbox your app reads (IMAP) and replies
from (SMTP). Gmail and Microsoft 365 have their own packages that need no app password; this is the one for everything
else. Pair it with [`Osysharp.Mail.Receiving`](https://osyrin.com/templates/kits/mail-receiving/).

```osy
// app.osy — you name your own servers, so the package reaches only what you grant
use Osysharp.Mail@0;
use Osysharp.Mail.Receiving@0;
use Osysharp.Mail.Imap@0;
use Osysharp.Http;
```

```osy
app.Mail = new MailSetup {
  Mailboxes = [new ImapMailbox { Address = "support@acme.com", ImapHost = "imap.acme.com", SmtpHost = "smtp.acme.com",
                                 Password = Secret.SupportMailboxPassword }],
  Receiver  = new MailIntoConversations(),
};
```

New mail is read with PEEK (nothing is marked read), from where the watch left off; when the server re-creates the
folder the watch says so and resumes from now. Replies are submitted over SMTP with TLS (or STARTTLS on 587). The
transport refuses private and loopback addresses, and never logs the password. Until the password is set the watch
says which setting is missing.

`tests/` stubs `Imap.Fetch` and `Smtp.Send` and asserts what the app kept and what it submitted.

IMAP is POLLED — every five minutes by default (`PollInterval`), and never more often than the app's plan allows. IDLE
would hold a connection open per mailbox for as long as it is watched.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.Mail;
using Osysharp.Mail.Receiving;
using Osysharp.Mail.Imap;

[Test(Inbox)]
void a_watched_imap_mailbox_starts_at_its_newest_uid_and_keeps_what_arrives_after() {
  runas (Ada) {
    var w = WatchMailbox("support@acme.test");
    UnitOfWork.Commit();
    Workflow.Settle(w);
  }
  runas (Ada) {
    Assert.Equal("42:6", MailboxWatch.Single().Cursor);
    // IMAP is polled on its own, slower interval (five minutes by default): a minute on, it has not asked again.
    TestClock.Advance(TimeSpan.FromMinutes(1));
    Workflow.Settle(MailboxWatch.Single());
    Assert.False(WireLog.Any(w => w.Verb == "IMAP" && w.Cursor == "42:6"));
    TestClock.Advance(TimeSpan.FromMinutes(4));
    Workflow.Settle(MailboxWatch.Single());
    foreach (var r in ReceivedMail.ToList()) { Workflow.Settle(r); }
  }
  runas (Ada) {
    var kept = ReceivedMail.Single();
    Assert.Equal("7", kept.ProviderId);
    Assert.Equal("<seven@example.test>", kept.MessageId);
    Assert.Equal(1, Handled.Count(h => h.Subject == "Laptop will not boot"));
    Assert.Equal("42:7", MailboxWatch.Single().Cursor);
    // It asked the right server, as the mailbox, from where the last poll left off.
    var second = WireLog.Single(w => w.Verb == "IMAP" && w.Cursor == "42:6");
    Assert.Equal("imap.acme.test", second.Host);
    Assert.Equal(993, second.Port);
    Assert.Equal("support@acme.test", second.Login);
  }
}
```

From [`tests/imap.test.osy`](tests/imap.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Mail;
using Osysharp.Mail.Receiving;
using Osysharp.Mail.Imap;

[Test(Inbox)]
void a_reply_goes_out_over_smtp_as_the_message_the_outbox_built() {
  runas (Ada) {
    var reply = SendMail(new MailMessage {
      From = "Acme Support <support@acme.test>", To = ["cus@example.test"], Subject = "Re: Laptop will not boot",
      Text = "Hold the power button.", InReplyTo = "<seven@example.test>", References = ["<seven@example.test>"],
    });
    UnitOfWork.Commit();
    Workflow.Settle(reply);
  }
  runas (Ada) {
    var sent = OutgoingMail.Single();
    Assert.Equal(MailStatus.Sent, sent.Status);
    Assert.Equal("imap", sent.SentWith);
    Assert.Equal(sent.MessageId, sent.ProviderId);
    var wire = WireLog.Single(w => w.Verb == "SMTP");
    Assert.Equal("smtp.acme.test", wire.Host);
    Assert.Equal(587, wire.Port);
    var mail = ParseMail(Encoding.UTF8.GetBytes(wire.Message));
    Assert.Equal("<seven@example.test>", mail.InReplyTo);
  }
}
```

From [`tests/imap.test.osy`](tests/imap.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Mail;
using Osysharp.Mail.Receiving;
using Osysharp.Mail.Imap;

[Test(Inbox)]
void a_mailbox_with_no_smtp_server_is_not_ready_and_says_so() {
  var box = new ImapMailbox { Address = "help@acme.test", ImapHost = "imap.acme.test", Password = "p" };
  Assert.False(box.IsReady());
  Assert.Contains("SmtpHost (for replying)", box.NotReadyReason());
}
```

From [`tests/imap.test.osy`](tests/imap.test.osy).

## Learn more

- [The Osysharp.Mail.Imap reference](https://osysharp.com/reference/ui/mail-receiving-kit/)
- [Osysharp.Mail.Imap in the package catalogue](https://osysharp.com/packages/#osysharp-mail-imap)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
