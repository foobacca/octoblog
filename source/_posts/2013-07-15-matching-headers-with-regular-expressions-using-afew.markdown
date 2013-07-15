---
layout: post
title: "Matching headers with regular expressions using afew"
date: 2013-07-15 22:43
comments: true
categories: 
---
*This post is part of my [notmuch of a journey series](/blog/2013/03/notmuch/)*.

notmuch is somewhat limited in the choice of headers it processes, despite that being [an oft requested feature](http://notmuchmail.org/pipermail/notmuch/2012/012501.html).  However afew parses the entire email itself so it can do what it likes.

The recently added HeaderMatchingFilter will match a regular expression you provide against an arbitrary header you specify.  The text matched can be used as the tag.  A simple version of the [ListMailsFilter](https://afew.readthedocs.org/en/latest/filters.html#listmailsfilter) could be implemented as:

    [HeaderMatchingFilter.1]
    header = List-Id
    pattern = <(?P<list_id>.*)>
    tags = +lists +{list_id}

So if the `List-Id` header field is found, and it starts and ends with `<>` then the contents will be used as a tag, and in addition the tag `list_id` will be added to the message.

My initial motivation was that we use email addresses of the form:

    <projectname>-team@aptivate.org

for various projects, so I wanted to use the `projectname` as the tag without having to write a filter for every project.  With HeaderMatchingFilter I can do:

    [HeaderMatchingFilter.1]
    header = To
    pattern = (?P<team_name>[a-z0-9]+)-team@aptivate.org
    tags = +{team_name}

    [HeaderMatchingFilter.2]
    header = Cc
    pattern = (?P<team_name>[a-z0-9]+)-team@aptivate.org
    tags = +{team_name}

Also, we use redmine heavily.  It adds a header saying what project the email is associated with.  I can use that by doing:

    [HeaderMatchingFilter.3]
    header = X-Redmine-Project
    pattern = (?P<project>.*)
    tags = +{project}

On a related note, I have now written more [documentation for afew](https://afew.readthedocs.org/en/latest/manual.html), so hopefully the path after me will be smoother.
