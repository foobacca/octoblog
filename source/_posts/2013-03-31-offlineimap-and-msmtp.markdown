---
layout: post
title: "offlineimap and msmtp"
date: 2013-03-31 16:52
comments: true
categories: notmuch offlineimap msmtp password-storage
---
notmuch doesn't get my email or send it, so I'm turning to two commonly used tools to help me with this.

* [offlineimap](http://docs.offlineimap.org/en/latest/MANUAL.html) fetches the email over IMAP (and syncs it back)
* [msmtp](http://msmtp.sourceforge.net/) sends the email for me

The set up I had is fairly standard, though a key issue was 

## Password Storage

I'm running on a server and connecting over ssh, as I want the same set of tags to be available everywhere.  I don't want my password sitting on disk in clear text (particularly as it is my LDAP password, so used for `sudo`, `ssh` and various other logins).  There are various solutions to this if you are running a desktop - [python-keyring-lib](https://bitbucket.org/kang/python-keyring-lib) will link to Gnome, KDE and OS X keyrings.  But it took a bit more searching to work out how to do this on a server.

I considered a number of options

### GPG encrypted file

I found [this unix stackexchage post](http://unix.stackexchange.com/a/48355/668) about having a python script that could decrypt a GPG encrypted file (one file per password).  gpg-agent would run on the server so you didn't have to enter your passphrase every time you need to run it.  I got this running, generating a GPG key that I wasn't actually going to use for email.  That meant I was OK setting a long time out for gpg-agent - I set it to 24 hours by putting the following in `~/.gnupg/gpg-agent.conf`

    pinentry-program /usr/bin/pinentry-curses
    default-cache-ttl 86400
    max-cache-ttl 86400

However I wanted to be able to leave offlineimap running and at some point gpg-agent would expire, so this wasn't really what I wanted.

That said I did leave this system in place for msmtp, as that will only run when I send an email - so I will be logged in at that point and can enter my GPG passphrase into gpg-agent.  The relevant line for the `.msmtprc` file is:

    passwordeval gpg --use-agent --quiet --batch -d ~/.passwd/account.gpg

### Environment variables

I found a [blog post describing caching the decrypted password in environment variables](https://blog.erroneousthoughts.org/461-2/) which seemed quite cunning.  That would only survive for the session, but then I leave tmux running all the time, so as long as tmux survived, the environment variables would.  But then I thought, why not just:

### Leave offlineimap running in tmux

When you run offlineimap, if there is no password or passwordeval in the `.offlineimaprc` file, then offlineimap will ask for your password (provided the ui isn't set to be Noninteractive).  offlineimap can also be told to run repeatedly, without exiting, by putting `autorefresh = 5` to re-run every 5 minutes.  In between runs, offlineimap will just sit at the terminal waiting, but it remembers your password, keeping it in memory.

This is what I went for, as I can leave offlineimap running for months inside tmux on the server I use.

(Note that offlineimap will not ask for your password if the `ui` is set to be Noninteractive.  And the `autorefresh` setting must be in the `[Account X]` section - I put it in the Repository section originally which didn't work).

### offlineimap preauthtunnel

One other option I found, but didn't use in the end, was to use the `preauthtunnel` option.  This uses a ssh connection to connect to the server and talk to the imap software on the other end.

The relevant command for courier IMAP is:

    preauthtunnel = ssh -q imaphost '/usr/bin/imapd ./Maildir'

And for dovecot it is:

    preauthtunnel = ssh -o Compression=yes -q imaphost 'MAIL=maildir:~/Maildir exec /usr/lib/dovecot/imap'

So I could have set up a password-less ssh key to do this, or left an ssh session running that offlineimap could have [multiplexed its connections](http://protempore.net/~calvins/howto/ssh-connection-sharing/) on to.

## Other notes

### Checking IMAP mailboxes

I was moving folders around and was having trouble working out whether the folders were available but offlineimap was failing to find them, or if dovecot wasn't showing me the folders.  Eventually I found [how to ask for a folder list over telnet](http://wiki.dovecot.org/MissingMailboxes). In short:

    telnet imap.example.org 143
    A login username password
    B list "" *
    C logout

(include the A, B, C).  Or to do this over ssl you could connect with:

    openssl s_client -connect mail.aptivate.org:993

and then run the above commands.
