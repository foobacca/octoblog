---
layout: post
title: "python 2.7 on debian squeeze"
date: 2013-03-31 18:35
comments: true
categories: 
---
*This post is part of my [notmuch of a journey series](/blog/2013/03/notmuch/)*.

There were various python libraries and apps I wanted to use, and [afew](https://github.com/teythoon/afew) and [alot](https://github.com/pazz/alot/) in particular requires python 2.7

I am running all this on a machine that is running Debian Squeeze which ships with python 2.6 and a little research reveals that it would be unwise to grab python 2.7 from testing/Wheezy as it will then become the system python and break lots of things.

But then I came across [pythonbrew](https://github.com/utahta/pythonbrew) which is inspired by the ruby rvm - it allows you to install different versions of python inside your home directory.  However, doing this straight off might lead you to be missing some vital functionality related to compression or unicode (as I did the first couple of attempts).  So the debian packages to install (covering all the packages I need for my notmuch journey) are:

    sudo apt-get install build-essential libbz2-dev libc6-dev libexpat1-dev \
        libgcrypt11-dev libglib2.0-dev libgmime-2.4-dev libgpg-error-dev \
        libgpgme11-dev libncurses5-dev libncursesw5-dev libreadline-dev \
        libsqlite3-dev libssl-dev zlib1g-dev python-pip
    sudo apt-get install -t testing libnotmuch-dev notmuch libnotmuch3

Then you can do:

    pip install pythonbrew
    pythonbrew install 2.7.3
    pythonbrew use 2.7.3   # this starts using python 2.7 in the current shell

I also wanted to have the notmuch python bindings, so I did:

    git clone git://notmuchmail.org/git/notmuch
    cd notmuch/bindings/python
    python setup.py install

At this point I was pretty much good to go, although I should also mention that you can save having to remember which version of python you are currently using by having wrapper scripts with contents like:

    #!/bin/sh
    $HOME/.pythonbrew/pythons/Python-2.7.3/bin/alot $@

Finally, I also came across a fork of pythonbrew called [pythonz](https://github.com/saghul/pythonz) that I may look into further at some point.  It support PyPy and Jython amongst others, but as it says in the README "[pythonbrew] has some extra features which I don't really need, so I made this for to make something a bit simpler that works for me" so some things might be broken ...
