# jfe
Jump to Full Encryption

# TL;DR

Run one command, everything on your system supports TLS.

# What I'm Up To
Possible isn't enough.  If you want it done, make it easy.  I'm trying to make TLS deployment easy.

Every project should be judged by the number of meetings required to get it accomplished.  For a long
time it's required a lot of high-touch interactions to add cryptographic services to Internet systems.
Let's Encrypt / CertBot has finally made machine acquirable certificates freely available (money
requires meetings) and so there's been a steady trend towards services acquiring and maintaining
their own certificates.

Which is great, except everyone does it differently, if at all.  Lot of legacy out there, lot of
protocols not running on HTTP.  There's more to TLS than HTTPS.

Couldn't all this just work?

JFE is an attempt to move us towards system-wide ambient cryptography.  It monitors incoming sessions
for bytes declaring an expectation of cryptography, and silently fulfills that request, systemwide.

It's early code, and I expect it to be rewritten many times.  OK!

# Quick Demo
    # ./jfe  -h
    Usage: 
    JFE (Jump to Full Encryption):  Full System Encryption Agent
    Dan Kaminsky, Chief Scientist, whiteops.com
    
    NOTE:  This is just a demonstration of a system-wide "bump in the wire"
    TLS implementation providing encryption on demand when requested by a
    client, using the new machine acquirable certicates freely available
    from Certbot/Let's Encrypt/CertBot.
    
    Options:
      -h, --help            show this help message and exit
      -s, --staging         Use Staging Server (Not As Rate Limited)
      -c ACCOUNTCACHE, --cache=ACCOUNTCACHE
                            Location of cached credentials (./cache)
      -v, --verbose         Verbose (false)    root@Server:~/jfe# ./jfe.py --staging
    ...
    
    user@Client:~# curl -k http://163.jfe.example
    hello worl
    
    # Zero configuration, that server has TLS
    user@Client:~# curl -k https://163.jfe.example
    hello worl
    
    # ...as does every TCP server on this host
    user@Client:~# curl -k https://163.jfe.example:40000
    <!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
    
    <html>
    ...    

# INSTALL
This needs to be cleaned up, but basically:

    mkdir /var/jfe
    pip install -r requirements.txt

# TODO

1) The particular methodology here yields servers that think they're talking
to 127.0.0.1, and worse, expose their services that are bound exclusively
to localhost.  Probably the most important bug to smash.

2) Performance ain't great, and the finer points of handling proxied sockets
and socket flushing are really tricky to get right.  I wouldn't mind a
much more performant solution nor kernel modules to get there.

3) Docker's exposed sockets would likely be a pretty good place
to integrate this trickery.

4) The clouds tend to have pretty good services around TCP proxying
and TLS wrapping, up to and including certificate integration backed
by their DNS engines.  This could all get wrapped up with a nice bow.

5) Could be interesting to take the Amazon cert exposed on 169.254.169.254
and hook it up to Let's Encrypt.  Oh, you didn't know there was one?
It's on every instance.

6) SSH integration.  Not everyone likes TOFU.

7) Build the better DNS validation backend.

# HOWTO
It's actually mildly tricky to make all these pieces work together,
and I haven't entirely figured out how to get everything I want.

Here's implementation requirements right now.  Don't care about
implementation language -- it's a daemon.

1) Need to be able to setsockopt IP_TRANSPARENT onto the listening
socket, after listen() and before bind() (same at SO_REUSEADDR).

2) Need to be able to do a MSG_PEEK into the accept()'ed socket,
to see if TLS is being requested by the client.  Alternatively, need
to be able to upgrade a socket from TCP to TLS having already read
bytes from it, including the bytes from the CLIENT_HELLO.  It's already
a miracle when an ecosystem supports upgrading a connection already in
progress without already read bytes (as required by Deferred TLS / SMTP's
STARTTLS) so I'm doubtful, but depending on MSG_PEEK is of course
wrong and terrible and suboptimal.

3) Need to be able to change certificate based on the results of a
SNI callback.

4) Need to not block all other sessions in progress while this particular
session goes out and gets a cert.

There's all sorts of issues in the present implementation.  Probably the
biggest one is the client thinks it's talking to localhost, which of course
exposes localhost services.  I'm averse to nothing, including kernel hacking
and modules.  Performance on this threading implementation is also poor,
and the local HTTP server isn't working for reasons.  But heh!  Let's fix things.
