# Osysharp.Mail.Microsoft365

A **Microsoft 365 / Exchange Online mailbox** as a mailbox your app reads and sends from — through Microsoft Graph,
with **your own app registration**. Pair it with [`Osysharp.Mail.Receiving`](https://osyrin.com/templates/kits/mail-receiving/).

```osy
// app.osy
use Osysharp.Mail@0;
use Osysharp.Mail.Receiving@0;
use Osysharp.Mail.Microsoft365@0 { egress "login.microsoftonline.com"; egress "graph.microsoft.com"; }
```

```osy
app.Mail = new MailSetup {
  Mailboxes = [new Microsoft365Mailbox { Address = "support@acme.com", TenantId = "…", ClientId = "…",
                                         ClientSecret = Secret.GraphClientSecret }],
  Receiver  = new MailIntoConversations(),
};
```

## Setting it up

1. Register an app in **Entra ID** and give it the **application** permissions `Mail.Read` and `Mail.Send`; an
   administrator of the tenant consents once.
2. Scope it to the support mailbox with an Exchange **application access policy**, so it can read no other mailbox.
3. Store the client secret as a secret (`osy secret set GraphClientSecret`).

New mail is read from the inbox's delta (nothing is marked read); a reply whose `From` is the mailbox is sent from it as
MIME, threaded. When Graph expires the delta, the watch says so and resumes from now.

The secret is yours: the package never learns its name, and reaches no host but Entra ID's sign-in and Graph. `tests/`
stubs both and asserts what reached Graph and what the app kept.

## Push through Graph change notifications (0.2.0)

With `app.Mail.PushPath` and the push door (`MailboxPushed`, see `Osysharp.Mail.Receiving`) published over public
HTTPS, the inbox is SUBSCRIBED to new messages: Graph validates the door once (it answers the handshake), then
announces each new mail with the subscription's secret `clientState`, which the watch checks. The subscription is
renewed (a `PATCH` of its expiry) well before Graph's ~3 days; reauthorization and removal notices are honoured. A
half-hourly poll still runs. `Push = false` keeps the mailbox polled. See `osy docs ui-mail-receiving-push`.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Http;
using Osysharp.Identity;
using Osysharp.Mail;
using Osysharp.Mail.Receiving;
using Osysharp.Mail.Microsoft365;

[Test(Inbox)]
void a_new_watch_starts_a_delta_from_now_and_imports_nothing_older() {
  runas (Ada) { Watch(); }
  runas (Ada) {
    Assert.Equal(DeltaLink(0), MailboxWatch.Single().Cursor);
    Assert.False(ReceivedMail.Any());
    var start = WireLog.Single(w => w.Url.Contains("/messages/delta?$select=id"));
    Assert.Contains("$filter=receivedDateTime+ge+", start.Url);
    // An app-only token, from the tenant's own sign-in endpoint, with the client secret in the form body.
    var token = WireLog.First(w => w.Verb == "POST");
    Assert.Equal("https://login.microsoftonline.com/tenant-1/oauth2/v2.0/token", token.Url);
    Assert.Contains("grant_type=client_credentials", token.Body);
  }
}
```

From [`tests/microsoft365.test.osy`](tests/microsoft365.test.osy).

```osy
using Osysharp.Http;
using Osysharp.Identity;
using Osysharp.Mail;
using Osysharp.Mail.Receiving;
using Osysharp.Mail.Microsoft365;

[Test(Inbox)]
void new_mail_in_the_delta_is_fetched_as_mime_and_kept_and_a_removal_is_not() {
  runas (Ada) { Watch(); }
  runas (Ada) {
    Arrive("AAMk-1", Mail("<one@example.test>", "Printer on fire"));
    Arrive("AAMk-2", Mail("<gone@example.test>", "Deleted"), removed: true);
    UnitOfWork.Commit();
    TestClock.Advance(TimeSpan.FromMinutes(1));
    Poll();
  }
  runas (Ada) {
    var kept = ReceivedMail.Single();
    Assert.Equal("AAMk-1", kept.ProviderId);
    Assert.Equal("<one@example.test>", kept.MessageId);
    Assert.Equal(1, Handled.Count(h => h.Subject == "Printer on fire"));
    Assert.Equal(DeltaLink(1), MailboxWatch.Single().Cursor);
    Assert.True(WireLog.Any(w => w.Url.EndsWith("/users/support%40acme.test/messages/AAMk-1/$value")));
    Assert.False(WireLog.Any(w => w.Url.Contains("/messages/AAMk-2/")));
  }
}
```

From [`tests/microsoft365.test.osy`](tests/microsoft365.test.osy).

```osy
using Osysharp.Http;
using Osysharp.Identity;
using Osysharp.Mail;
using Osysharp.Mail.Receiving;
using Osysharp.Mail.Microsoft365;

[Test(Inbox)]
void a_reply_goes_through_sendmail_as_mime_with_its_thread() {
  runas (Ada) {
    var reply = SendMail(new MailMessage {
      From = "Acme Support <support@acme.test>", To = ["cus@example.test"], Subject = "Re: Printer on fire",
      Text = "Unplug it.", InReplyTo = "<one@example.test>", References = ["<one@example.test>"],
    });
    UnitOfWork.Commit();
    Workflow.Settle(reply);
  }
  runas (Ada) {
    var sent = OutgoingMail.Single();
    Assert.Equal(MailStatus.Sent, sent.Status);
    Assert.Equal("microsoft365", sent.SentWith);
    var wire = WireLog.Single(w => w.Url == "https://graph.microsoft.com/v1.0/users/support%40acme.test/sendMail");
    var mail = ParseMail(Convert.FromBase64String(wire.Body));
    Assert.Equal("<one@example.test>", mail.InReplyTo);
    Assert.Equal(sent.MessageId, mail.MessageId);
  }
}
```

From [`tests/microsoft365.test.osy`](tests/microsoft365.test.osy).

## Learn more

- [The Osysharp.Mail.Microsoft365 reference](https://osysharp.com/reference/ui/mail-receiving-kit/)
- [Osysharp.Mail.Microsoft365 in the package catalogue](https://osysharp.com/packages/#osysharp-mail-microsoft365)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
