---
layout: post
title: "offlineimap and msmtp"
date: 2013-03-31 16:52
comments: true
categories: notmuch offlineimap msmtp password-storage
---
*This post is part of my [notmuch of a journey series](/blog/2013/03/notmuch/)*.

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

When you run offlineimap, if there is no password or passwordeval in the `.offlineimaprc` file, then offlineimap will ask for your password (provided the ui isn't set to be Noninteractive).  offlineimap can also be told to run repeatedly, without exiting, by putting `autorefresh = 5` to re-run every 5 minutes.  In between runs, offlineimap will just sit at the terminal waiting, but it remembers your password, keeping it in memory.  Using this I could leave offlineimap running for months inside tmux on the server I use.

(Note that offlineimap will not ask for your password if the `ui` is set to be Noninteractive.  And the `autorefresh` setting must be in the `[Account X]` section - I put it in the Repository section originally which didn't work).

### ssh keys and offlineimap preauthtunnel

Finally I worked out that I could talk directly to an imap command line utility on the remote server over ssh - using a passwordless ssh key to avoid any passwords.  offlineimap is instructed to use it by the `preauthtunnel` option.

There are two main IMAP servers in the linux world, courier and dovecot.  The relevant commands are:

    # Courier IMAP (Debian - you should check the path on CentOS)
    preauthtunnel = ssh -o Compression=yes -q IMAPHOST '/usr/bin/imapd ./Maildir'
    
    # Dovecot IMAP (CentOS)
    preauthtunnel = ssh -o Compression=yes -q IMAPHOST 'MAIL=maildir:~/Maildir exec /usr/libexec/dovecot/imap'
    # Dovecot IMAP (Debian)
    preauthtunnel = ssh -o Compression=yes -q IMAPHOST 'MAIL=maildir:~/Maildir exec /usr/lib/dovecot/imap'

Replace `IMAPHOST` and `~/Maildir` with your own values.

There are two ways to do this so that you don't have to enter the ssh key passphrase all the time.  You could set up a password-less ssh key to do this, or you could leave an ssh session running so that offlineimap can [multiplex its connections](http://protempore.net/~calvins/howto/ssh-connection-sharing/) on to it.

A password-less ssh key leads to the possibility of abuse, but you can generate a new ssh key just for imap and specify it's use by modifying the above command to look like:

    preauthtunnel = ssh -q -i /home/hamish/.ssh/id_mail_imap -o Compression=yes IMAPHOST '/usr/libexec/dovecot/imap'

On the mail server side you can then lock down this key by configuring what command can be run in the `.ssh/authorized_keys` file.  Mine looks like:

    command="/usr/libexec/dovecot/imap",no-X11-forwarding,no-agent-forwarding,no-port-forwarding,no-pty ssh-dss AAAAB3Nza...

Obviously this will only work for you if you have ssh access to your mail server, so it won't be an option for all.  But I've given some other options above, so I hope you can find something that works for you.

## Other notes

### sendmail over ssh

Another option for sending email to a remote server is to use the sendmail command over ssh (assuming you have ssh access to the server in question).  First you need to generate a passwordless ssh key and copy it to the mail server:

    ssh-keygen -t dsa -f ~/.ssh/id_mail_smtp
    scp ~/.ssh/id_mail_smtp.pub smtpserver.example.org:.ssh/

Then, on the mail server, put the key in the authorized_keys file:

    cat ~/.ssh/id_mail_smtp.pub >> ~/.ssh/authorized_keys

Next we edit the authorized keys file, adding the command to run to the beginning of the last line (which is currently our smtp key):

    command="/usr/sbin/sendmail -bm -t -oem -oi",no-X11-forwarding,no-agent-forwarding,no-port-forwarding,no-pty ssh-dss AAA...

In this case the `command` is the sendmail command for exim - I'm afraid you'll have to work out what your own sendmail command is.  Finally, in your mail client, tell it to use ssh as the sendmail command.  The following works for both mutt and alot:

    ssh -q -i /home/hamish/.ssh/id_mail_smtp smtpserver.example.org

Now whenever your mail client wants to send email, it will start up the ssh connection and print the email to it.  This will be picked up by the sendmail command on the far end and off the email will go.

### Checking IMAP mailboxes

I was moving folders around and was having trouble working out whether the folders were available but offlineimap was failing to find them, or if dovecot wasn't showing me the folders.  Eventually I found [how to ask for a folder list over telnet](http://wiki.dovecot.org/MissingMailboxes). In short:

    telnet imap.example.org 143
    A login username password
    B list "" *
    C logout

(include the A, B, C).  Or to do this over ssl you could connect with:

    openssl s_client -connect mail.aptivate.org:993

and then run the above commands.

### Other ways of sending to multiple accounts

In my research, I also found that you can set up [postfix to use different servers for different accounts](http://paul.frields.org/2009/07/12/best-in-show/) (though I'm not sure how well it would handle encrypted passwords) and I may improve my msmtp set up by using [msmtp queue](https://wiki.archlinux.org/index.php/Msmtp#Using_msmtp_offline) at some point, and see how well it deals with encrypted passwords.
