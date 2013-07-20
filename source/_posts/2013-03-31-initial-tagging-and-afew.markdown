---
layout: post
title: "initial tagging and afew"
date: 2013-04-01 09:00
comments: true
categories: notmuch afew
---
*This post is part of my [notmuch of a journey series](/blog/2013/03/notmuch/)*.

I love the way gmail uses labels and I loved using them in sup.  notmuch has tags for the same purpose so I want to use them extensively, and to have most tags added automatically for me.

## notmuch new mail workflow

There are several ways to do this, but the way I've done it is:

* offlineimap runs every X minutes
* the offlineimap `postsynchook` is set to `notmuch new` - so when the IMAP run is complete, notmuch is called
* notmuch new processes all the email it doesn't know about, adding a set of tags to every new message
* notmuch then checks for the file `<maildir>/.notmuch/hooks/post-new` and if it exists then it will run it
* in my case, the hook file calls `afew --new --tag`

The tags added by `notmuch new` are set in the notmuch config file, in the part that looks like:

    [new]
    tags=new

So all new messages will have those tags added.  Your `post-new` hook can then run only against mail that is tagged "new", making it nice and fast.  And as the final step, the `post-new` hooks should remove the "new" tag from all messages that have it, so that you don't have to re-process them next time.  A common idea is also to remove the "new" tag from messages that match filters you want to archive, and at the end of your filter list you replace the "new" tag with the "inbox" tag for any messages still tagged "new".

You can do this with a basic shell script that calls the notmuch command line binary, as shown on the [notmuch initial tagging page](http://notmuchmail.org/initial_tagging/).  However the command line binary only looks at a few headers (From, To, Cc, Subject, Date and message ID as far as I can tell).  So I thought I'd look into:

## afew

[afew](https://github.com/teythoon/afew) is a python command that will run filters against a set of messages and generate tags.  I can install it using pip.  However, at the time of writing the [documentation](https://afew.readthedocs.org/en/latest/manual.html) is not as useful as it could be.  I ended up reading the code to work out what was going on.

One key bit of information is that the config files are stored in `~/.config/afew/config` which meant I could search github for [path:afew/config](https://github.com/search?q=path%3Aafew%2Fconfig&type=Code&ref=advsearch&l=) - which only brought up two hits when I just tried it, but that's a start to see what others do.  The most useful one I found was by [pazz](https://github.com/pazz/configs/blob/master/.config/afew/config) and you might also want to look at [my afew config](https://github.com/foobacca/dotfiles/blob/master/afew/config).

When you're playing with afew, running `afew --help` will get you started reasonably.  And the command you want for the `post-new` hook is:

    afew --tag --new

Which will run against the new messages and run the tag filters.

## afew filters

I had to read the code to work out what some of the filters were, so here are my findings as a first stab at documenting them.  Later I intend to improve the docs.

### afew's built in filters

The default filter set (if you don't specify anything in the config) is:

    [SpamFilter]
    [ClassifyingFilter]
    [KillThreadsFilter]
    [ListMailsFilter]
    [ArchiveSentMailsFilter]
    [InboxFilter]

These can be customised by specifying settings beneath them.  The standard settings are:

* `message` - text that will be displayed while running this filter if the verbosity is high enough.
* `query` - the query to use against the messages, specified in standard notmuch format
* `tags` - the tags to add to messages that match the query
* `tag_blacklist` - if the message has one of these tags, don't add `tags` to it

Note that most of the filters below set their own value for message, query and/or tags, and some ignore some of the above settings.

#### SpamFilter

This looks for the header `X-Spam-Flag` - if it finds it, the `spam` tag is set.  You can override the tag used for spam if you want.

The settings you can use are:

* `spam_tag` is the tag used to identify spam. It defaults to "spam"

#### ClassifyingFilter

This is to do with learning what tags match what without explicit rules.  I haven't worked this one out fully, but the project README has some details on how to use it.  If I work out more I'll blog about and link to that from here.

#### KillThreadsFilter

If the new message has been added to a thread that has already been tagged "killed" then add the "killed" tag to this message.  This allows for ignoring all replies to a particular thread.

#### ListMailsFilter

This filter looks for the `List-Id` header, and if it finds it, adds the list name as a tag, together with the tag "lists".

#### ArchiveSentMailsFilter

Basically does what says it on the tin.  Though more accurately, it looks for emails that are from one of your addresses *and not* to any of your addresses.  It then adds the sent tag and removes the inbox tag.

#### InboxFilter

This removes the new tags, and adds the "inbox" tag to any message that isn't killed or spam.

#### FolderNameFilter

This looks at which folder each email is in and uses that name as a tag for the email.  So if you have a procmail or seive set up that puts emails in folders for you, this might be useful.

### Adding your own filters to afew

You can modify filters, and define your own versions of the base Filter that allow you to tag messages in a similar way to the `notmuch tag` command, using the settings above.  Showing some sample configs is the easiest way to understand.  The [notmuch initial tagging page](http://notmuchmail.org/initial_tagging/) shows a sample config:

    # immediately archive all messages from "me"
    notmuch tag -new -- tag:new and from:me@example.com

    # delete all messages from a spammer:
    notmuch tag +deleted -- tag:new and from:spam@spam.com

    # tag all message from notmuch mailing list
    notmuch tag +notmuch -- tag:new and to:notmuch@notmuchmail.org

    # finally, retag all "new" messages "inbox" and "unread"
    notmuch tag +inbox +unread -new -- tag:new

The equivalent in afew would be:

    [ArchiveSentMailsFilter]

    [Filter.spamcom]
    message = Delete all messages from spammer
    query = from:spam@spam.com
    tags = deleted;-new

    [Filter.notmuch]
    message = Tag all messages from the notmuch mailing list
    query = to:notmuch@notmuchmail.org
    tags = notmuch

    [Filter.myinbox]
    message = My version of the inbox filter
    query = tag:new
    tags = inbox;unread;-new

Not that the queries do not include `tag:new` because this is implied when afew is run with the `--new` flag.

Here are a few more example filters from github dotfiles:

    [Filter.1]
    query = 'sicsa-students@sicsa.ac.uk'
    tags = +sicsa
    message = sicsa

    [Filter.2]
    query = 'from:foosoc.ed@gmail.com OR from:GT Silber OR from:lizzie.brough@eusa.ed.ac.uk'
    tags = +soc;+foo
    message = foosoc

    [Filter.3]
    query = 'folder:gmail/G+'
    tags = +G+
    message = gmail spam

    # skip inbox
    [Filter.6]
    query = 'to:notmuch@notmuchmail.org AND (subject:emacs OR subject:elisp OR "(defun" OR "(setq" OR PATCH)'
    tags = -inbox;-new
    message = notmuch emacs stuff

If you need more powerful processing you can write filters that match regular expressions against any header in the email with the HeaderMatchingFilter, but more on that in a future blog post.
