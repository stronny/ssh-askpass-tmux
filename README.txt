BACKGROUND
==========

This script serves the purpose of ssh-askpass, using tmux as a kind of
a window manager so that no X server is required to be able to confirm
the ssh-agent actions. The passphrase input is also provided, but is not
the problem this script tries to address.

The exact use case that I had is the ability to verify and confirm each
usage of ssh identities loaded into ssh-agent which is not something that
ssh can do natively. This is not a problem if you only use your keys on
a secure machine, but consider for example agent forwarding: anyone can
make use of your agent as long as he's able to access its socket.

So the only practical security mechanism seems to be the SSH_ASKPASS
infrastructure built into OpenSSH, but using it is not this simple. The
program is invoked with a message from ssh as its first argument and its
STDOUT piped back so that the user input can be captured by ssh.
Additionally the exit status is examined and if not zero is considered
a failure. What further complicates things is that this program is used
for two distinct actions: a) input a passphrase; b) confirm key usage.
There is no indication from its invocation what is expected from it.
There is also no indication what machine requests the key usage, only
the key itself is presented.

Okay, so pure console solution seemed hard if even possible, and I do
not run an X server. After some thought I've decided that as I'm inside
tmux anyway I could use it as a sort of a window manager that can
provide a decent facility for my needs. So upon its invocation the
script opens a new tmux window that shows a dialog, takes user input
and closes, returning you to the state before. The user input is piped
back via a FIFO. The mode of operation (passphrase or confirmation) is
chosen upon the message from ssh.


REQUIREMENTS
============

* tmux, for obvious reasons
* bash 4.0+, for mapfile (seriously, keep your software up-to-date)
* whiptail, for a dialog window


INSTALL
=======

1. Drop the script somewhere, e. g. /usr/local/bin (system-wide) or ~/bin (user-wide)
2. Set DISPLAY and SSH_ASKPASS variables


KEYCHAIN INTEGRATION
====================

You need to modify your profile string, here is an example:

[ -n "$TMUX" ] && eval $(DISPLAY=:0 SSH_ASKPASS='/path/to/ssh-askpass-tmux' keychain --eval --confirm id_rsa other_identity)


LOCAL KEY USAGE
===============

Typically most of your ssh activity originates from a secure workstation,
and constant key usage confirmation is really irritating, so there is
a built-in facility to confirm the key usage programmatically. The script
will run a command from an environment variable SSH_ASKPASS_TMUX_CHECK and
will exit with 0 if that command exited with 0. You are free to implement
anything that suits your needs, here is what I came up with:

$ env | grep SSH_ASK
SSH_ASKPASS_TMUX_CHECK=/home/stronny/bin/ssh-askpass-tmux-check

$ cat bin/ssh-askpass-tmux-check
#!/usr/bin/env bash
MT="$(stat "$HOME/.ssh/config_read" --printf='%Y' 2>/dev/null)"
(( $? )) || (( $(date +'%s') - MT <= 5 ))

$ incrontab -l
/home/stronny/.ssh/config IN_ACCESS touch $@_read

So in a nutshell whenever .ssh/config is accessed, incrond will touch
a file, checkscript will then grant access for key usage unless the file's
mtime is older than 5 seconds. Why not just use atime? Well, I run with
-o rw,noatime, so, yeah.
