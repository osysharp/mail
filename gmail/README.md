# Osysharp.Mail.Gmail

A **Google Workspace or Gmail inbox** as a mailbox your app reads and sends from — through the Gmail API, with **your
own Google credentials**. Pair it with [`Osysharp.Mail.Receiving`](https://osyrin.com/templates/kits/mail-receiving/).

```osy
// app.osy
use Osysharp.Mail@0;
use Osysharp.Mail.Receiving@0;
use Osysharp.Mail.Gmail@0 { egress "oauth2.googleapis.com"; egress "gmail.googleapis.com"; }
```

```osy
app.Mail = new MailSetup {
  Mailboxes = [new GmailMailbox { Address = "support@acme.com", ClientId = "…apps.googleusercontent.com",
                                  ClientSecret = Secret.GoogleClientSecret, RefreshToken = Secret.SupportMailbox }],
  Receiver  = new MailIntoConversations(),
};
```

## Setting it up

1. In your Google Cloud project, create an **OAuth client** and enable the Gmail API.
2. Have the mailbox's owner grant it once, with the scopes `gmail.readonly` and `gmail.send`; keep the **refresh token**.
3. Store the client secret and the refresh token as secrets (`osy secret set GoogleClientSecret`, `… SupportMailbox`).

New mail is read from Gmail's history (nothing is marked read); a reply whose `From` is the mailbox is sent from it,
threaded. When Google no longer holds the history the watch was at, the watch says so and resumes from now.

Both secrets are yours: the package never learns their names, and reaches no host but Google's token endpoint and the
Gmail API. `tests/` stubs both and asserts what reached Google and what the app kept.

## Push through Cloud Pub/Sub (0.2.0)

Set `PushTopic` (`projects/<project>/topics/<topic>`, with `gmail-api-push@system.gserviceaccount.com` a Publisher on
it) and `PushServiceAccount` (the account the topic's authenticated push subscription signs with), and give
`app.Mail.PushPath` and the push door (`MailboxPushed`, see `Osysharp.Mail.Receiving`). The inbox is then watched with
`users.watch`, renewed before Gmail's seven days are up, and each push — whose Google-signed token the PLATFORM
verifies — wakes a fetch of the history. A half-hourly poll still runs, so a dropped push makes a mail late, never lost.
The whole set-up, step by step: `osy docs ui-mail-receiving-push`.

## Connect from a page

Instead of setting the refresh token by hand, an admin can connect the inbox: declare the OAuth client in
`app.OAuthClients` (`OAuthCapability.Connection`, the scopes above), set `ConnectWith` (its name) and `TokenSecret` (the
app secret its grant is kept in — the one `RefreshToken` reads), and have the page call
`Security.OAuthConnectUrl(...)` with what `Connection()` answers. See `osy docs config-oauth-connect`.
