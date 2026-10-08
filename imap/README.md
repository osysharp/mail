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
