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

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Http;
using Osysharp.Identity;
using Osysharp.Mail;
using Osysharp.Mail.Receiving;
using Osysharp.Mail.Gmail;

[Test(Inbox)]
void a_new_watch_starts_at_the_inboxs_history_id_and_imports_nothing_older() {
  runas (Ada) {
    Arrive("m-old", 999, Mail("<old@example.test>", "Before"));
    Watch();
  }
  runas (Ada) {
    Assert.Equal("999", MailboxWatch.Single().Cursor);
    Assert.False(ReceivedMail.Any());
    // The refresh token went to Google's token endpoint in the form body, nowhere else.
    var token = WireLog.First(w => w.Url == "https://oauth2.googleapis.com/token");
    Assert.Contains("grant_type=refresh_token", token.Body);
    Assert.Contains("refresh_token=refresh-1", token.Body);
  }
}
```

From [`tests/gmail.test.osy`](tests/gmail.test.osy).

```osy
using Osysharp.Http;
using Osysharp.Identity;
using Osysharp.Mail;
using Osysharp.Mail.Receiving;
using Osysharp.Mail.Gmail;

[Test(Inbox)]
void mail_added_to_the_inbox_is_fetched_whole_and_kept_and_sent_mail_is_not() {
  runas (Ada) { Watch(); }
  runas (Ada) {
    Arrive("m-1", 1001, Mail("<one@example.test>", "Broken export"));
    Arrive("m-2", 1002, Mail("<ours@acme.test>", "Re: Broken export"), "SENT");
    UnitOfWork.Commit();
    TestClock.Advance(TimeSpan.FromMinutes(1));
    Poll();
  }
  runas (Ada) {
    var kept = ReceivedMail.Single();
    Assert.Equal("m-1", kept.ProviderId);
    Assert.Equal("<one@example.test>", kept.MessageId);
    Assert.Equal("Broken export", kept.Subject);
    Assert.Equal(1, Handled.Count(h => h.MessageId == "<one@example.test>"));
    Assert.Equal("1002", MailboxWatch.Single().Cursor);
    Assert.True(WireLog.Any(w => w.Url == "https://gmail.googleapis.com/gmail/v1/users/me/messages/m-1?format=raw"));
    Assert.False(WireLog.Any(w => w.Url.Contains("/messages/m-2")));
  }
}
```

From [`tests/gmail.test.osy`](tests/gmail.test.osy).

```osy
using Osysharp.Http;
using Osysharp.Identity;
using Osysharp.Mail;
using Osysharp.Mail.Receiving;
using Osysharp.Mail.Gmail;

[Test(Inbox)]
void a_reply_goes_through_the_send_api_as_the_message_the_outbox_built() {
  runas (Ada) {
    var reply = SendMail(new MailMessage {
      From = "Acme Support <support@acme.test>", To = ["cus@example.test"], Subject = "Re: Broken export",
      Text = "Fixed.", InReplyTo = "<one@example.test>", References = ["<one@example.test>"],
    });
    UnitOfWork.Commit();
    Workflow.Settle(reply);
  }
  runas (Ada) {
    var sent = OutgoingMail.Single();
    Assert.Equal(MailStatus.Sent, sent.Status);
    Assert.Equal("gmail", sent.SentWith);
    Assert.Equal("sent-1", sent.ProviderId);
    var wire = WireLog.Single(w => w.Url == "https://gmail.googleapis.com/gmail/v1/users/me/messages/send");
    var raw = JsonSerializer.Deserialize<GmailRaw>(wire.Body).Raw;
    var mail = ParseMail(FromBase64Url(raw));
    Assert.Equal("<one@example.test>", mail.InReplyTo);
    Assert.Equal(sent.MessageId, mail.MessageId);
    Assert.Equal("Acme Support <support@acme.test>", mail.From);
  }
}
```

From [`tests/gmail.test.osy`](tests/gmail.test.osy).

## Learn more

- [The Osysharp.Mail.Gmail reference](https://osysharp.com/reference/ui/mail-receiving-kit/)
- [Osysharp.Mail.Gmail in the package catalogue](https://osysharp.com/packages/#osysharp-mail-gmail)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
