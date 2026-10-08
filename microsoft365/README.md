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
