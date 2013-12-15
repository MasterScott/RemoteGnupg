Introduction:
=============

Motivation: Signing or decryption of data using GnuPG usually
requires that the private key material is available within the
same context as the data to operate on. While this is unproblematic
in most use cases, such a situation might be suboptimal when data
from lower-trusted context, e.g. e-mail client, is processed.
Since many targeted attacks address weaknesses or vulnerabilities
in e-mail or web-client, successful compromise of client will
also allow theft of private key material.

The RemoteGnupg tool allows to forward some GnuPG requests from
lower-trusted to higher-trusted context via network with additional
filtering of the call parameters. Thus it is possible to apply
GnuPG operations to data in lower-trusted context without copying
the private key to that context.

The main difference of this approach to GnuPG agent forwarding
is that the tool allows to control which requests are forwarded
and accept only those, that seem appropriate.


Design:
=======

The RemoteGnupg consists of two programs, GnupgCallForwarder replaces
GnuPG in the lower-trusted context to intercept the call to GnuPG.
Depending on the argument the call is processed locally, e.g. for
signature verification, or forwarded to the remote instance.

Within the higher-trusted context, forwarded request is received
by GnupgCallReceiver. It filters the request command, and if appropriate
invokes GnuPG within this context. The user mas review the request
in the trusted context and decide, if it should be processed.

In the initialization phase, the forwarder encodes the call arguments
as list of 0-terminated strings and forwards them to the receiver.
After that phase, stdin on the forwarder side is sent via the
same link until EOF of stdin is reached.

The receiver processes the encoded call arguments, filters them,
may apply modifications to them and then invoke the local GnuPG
application. While the GnuPG subprocess is alive, stdout and stderr
are read and sent back to remote side by using the single stream
in multiplex mode. When the streams are closed or the application
terminates, remote side is notified so that the forwarder can
pass on that information to the program that invoked it.


Using with Enigmail:
====================

To use RemoteGnupg with Enigmail, open the OpenPGP preferences
basic settings tab and override GnuPG binary with RemoteGnupg
tool.

CAVEAT: I did not succeed in using the alternative GnuPG path
together with the additional parameters configuration item from
advanced settings tab due to unknown reason. Hence additional
parameters has to be kept empty, instead an intermediate script
has to be used, e.g. GnupgCallForwarderWrapper.sh

#!/bin/sh
exec "$(dirname "$0")/GnupgCallForwarder" --GnupgForwardRemoteAddress [host]:[port} --LogFile [your-log-file] "$@"

On the remote side, open a listening port, e.g. by combining socat
with GnupgCallReceiver. At this position, additional parameters
can be added to the GnuPG invocation. See preamble of GnupgCallReceiver
for more information. Command line might look like

socat TCP4-LISTEN:[listen-port],reuseaddr=1,fork=1 "EXEC:[your-path]/GnupgCallReceiver --PrependArg --homedir --PrependArg [your-homedir-if-needed]"


Revision List:
==============

20130519: Initial version introduced to enigmail list, asking
for feedback, see mailing list post:
https://admin.hostpoint.ch/pipermail/enigmail-users_enigmail.net/2013-May/000797.html

20131215: Upload to GitHub
