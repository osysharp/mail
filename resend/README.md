# Osysharp.Mail.Resend

Resend as the carrier for [Osysharp.Mail](https://osyrin.com/templates/kits/mail/): the app's outbox hands each message to Resend, with its
Message-ID and threading headers, its attachments and its pictures, and records Resend's id for it.

## Use it

```osy
// app.osy
use Osysharp.Mail@0;
use Osysharp.Mail.Resend@0 { egress "api.resend.com"; }
use Osysharp.Resend@0 { egress "api.resend.com"; }
use Osysharp.Http;
```

```osy
using Osysharp.Mail;
using Osysharp.Mail.Resend;

app.Mail = new MailSetup {
  Sender = new ResendSender { ApiKey = Security.IsSecretSet("ResendApiKey") ? Secret.ResendApiKey : "" },
  From = "Acme <noreply@acme.com>",          // on a domain you have verified with Resend
};
app.Secrets = [ new Secret("ResendApiKey") ];
```

Then `osy secret set ResendApiKey`. Until the key is set the sender is not ready, and mail waits in the outbox as
**held** instead of failing — the app works in the meantime.

The key is a setting the app fills from its own secret: this package never learns the secret's name, and it reaches no
host but `api.resend.com`.

## Why it is a package of its own

`Osysharp.Resend` is a plain Resend client for any app. This package is the few lines that make it an Osysharp.Mail
sender, so an app that does not use the outbox is not handed its tables.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Http;
using Osysharp.Identity;
using Osysharp.Mail;

[Test(Desk)]
void a_message_reaches_resend_with_its_own_message_id_and_answers_resends_id() {
  runas (Ada) {
    var mail = Notify("cus@example.test", "Your ticket");
    UnitOfWork.Commit();
    Workflow.Settle(mail);
  }
  runas (Ada) {
    var sent = OutgoingMail.Single(m => m.Subject == "Your ticket");
    Assert.Equal(MailStatus.Sent, sent.Status);
    Assert.Equal("re-123", sent.ProviderId);
    var wire = WireLog.Single(w => w.Body.Contains("Your ticket"));
    Assert.Equal("https://api.resend.com/emails", wire.Url);
    Assert.Contains("\"to\":[\"cus@example.test\"]", wire.Body);
    // The JSON escapes the angle brackets (`\u003C…\u003E`), so the id is matched between them.
    Assert.Contains("\"Message-ID\":\"\\u003C" + Bare(sent.MessageId) + "\\u003E\"", wire.Body);
  }
}
```

From [`tests/resend.test.osy`](tests/resend.test.osy).

```osy
using Osysharp.Http;
using Osysharp.Identity;
using Osysharp.Mail;

[Test(Desk)]
void a_reply_carries_its_threading_headers_to_resend() {
  runas (Ada) {
    var first = Notify("cus@example.test", "Your ticket");
    UnitOfWork.Commit();
    var reply = Answer(first);
    UnitOfWork.Commit();
    Workflow.Settle(reply);
  }
  runas (Ada) {
    var first = OutgoingMail.Single(m => m.Subject == "Your ticket");
    var wire = WireLog.Single(w => w.Body.Contains("Re: Your ticket"));
    Assert.Contains("\"In-Reply-To\":\"\\u003C" + Bare(first.MessageId) + "\\u003E\"", wire.Body);
    Assert.Contains("\"References\":\"\\u003C" + Bare(first.MessageId) + "\\u003E\"", wire.Body);
  }
}
```

From [`tests/resend.test.osy`](tests/resend.test.osy).

## Learn more

- [The Osysharp.Mail.Resend reference](https://osysharp.com/reference/ui/mail-kit/)
- [Osysharp.Mail.Resend in the package catalogue](https://osysharp.com/packages/#osysharp-mail-resend)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
