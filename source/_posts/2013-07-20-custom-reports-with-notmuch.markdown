---
layout: post
title: "Custom reports with notmuch"
date: 2013-07-20 15:23
comments: true
categories: notmuch
---

It's just a little thing, but I like to see counts of messages in certain categories.  As notmuch provides a count command, it is simple to write a script that will provide you with the counts you want.  Here's one I prepared earlier:

    #!/bin/sh

    # I'm interested in the thread count rather than individual messages
    NMC="/usr/bin/notmuch count --output=threads"

    # I break it down by total, how many are flagged and how many are unread
    # This function produces a single line
    function tag_report(){
      TAG=$1
      TOTAL_COUNT=$($NMC tag:$TAG)
      FLAGGED_COUNT=$($NMC tag:$TAG AND tag:flagged)
      UNREAD_COUNT=$($NMC tag:$TAG AND tag:unread)

      echo "$TAG: total $TOTAL_COUNT flagged $FLAGGED_COUNT unread $UNREAD_COUNT"
    }

    # then these are the tags I'm interested in, so print the reports
    # one line per tag
    function full_report(){
      tag_report inbox
      tag_report action
      tag_report waiting
      tag_report readlater
    }

    # finally, pipe through column to make the results line up nicely
    full_report | column -t

I've chosen to do this by having a small tmux window above my mail client, using `watch` to refresh.  The command I use is:

    watch -t -n 300 bin/mailreport

The `-t` means don't bother showing the time (saving a line of output) and the `-n 300` means only run the report once every 5 minutes.  The output (currently) looks like:

    inbox:      total  28  flagged  4  unread  0
    action:     total  55  flagged  4  unread  11
    waiting:    total  13  flagged  2  unread  1
    readlater:  total  60  flagged  2  unread  3
