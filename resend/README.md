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
