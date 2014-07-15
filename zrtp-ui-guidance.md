User Interface Guidelines for a ZRTP Secure Phone
=================================================

In this document we'll discuss the elements of a ZRTP secure phone
that must be exposed to the user through the user interface.  We'll
cover the normal gradations of security as well as exceptional
circumstances.  We'll talk about actions required of the user and how
those might be communicated.

To understand the security state of the call, we'll consider whether
the audio of the call (the "media") is encrypted, and if so, how.
We'll consider whether the signaling (the information exchanged to
setup the call) is secure.  And we'll consider whether ZRTP has
negotiated, and if so, the state of ZRTP.  ZRTP interacts with the
signaling and with information from prior calls to provide a range of
security assurances.

## Unencrypted Media

If we're sending unsecured media for any reason (RTP rather than
SRTP), then the call is not secure and the user must be warned loudly
about this.  e.g.:

> This call is NOT secure.  The audio and video of this call may be
> intercepted by anyone.

## TLS Signaling Security

We expect there to be an authenticated TLS tunnel with the server for
signaling.  If we're using UDP signaling, TCP signaling without TLS,
or TLS signaling where there was a certificate authentication failure,
or where no certificate authentication was performed, then we must
warn the user loudly about this.

e.g., in the case of signaling by UDP, TCP without TLS, or where no
TLS certificate authentication was performed:

> Your connection with the server is NOT secure.  Information
> regarding who is calling whom may be intercepted by anyone, and
> calls may not be confidential.

e.g., in the case of a TLS certificate authentication failure:

> Your connection with the server is NOT secure.  The server is being
> impersonated by an untrusted party.  Information regarding who is
> calling whom may be intercepted, and calls may not be confidential.

In either case, we must never enter the Secure to Server state
described below.  Further, we must suppress any treatment of
unverified ZRTP calls as partially secure.  All calls must be treated
as completely insecure until the user verifies the SAS.  Even then, we
must show some ongoing notification that the connection to the server
is untrusted.

(In normal operation, a TLS certificate authentication failure may
simply result in the connection failing completely.)

## SDP Security Descriptions (SDES)

If there is an authenticated TLS connection with the server, and the
media is SRTP keyed by the SDP Security Descriptions (SDP) in the
signaling, then the call is Secure to Server.  The call can be
wiretapped by the server, by parties capable of coercing or subverting
those managing the server, and by anyone who can wiretap the other leg
of the call, such as with PSTN calls.

We might be in this state because this is the best we can do in
general (PSTN calls), or because this is the best we can do so far
(ZRTP is working on setting up), or because this is the best we can do
on this call (ZRTP failed to negotiate).

This is the minimum level of security about which we need not scare
the user.

## ZRTP

If ZRTP negotiates successfully, we have some additional security
states to consider.

### ZRTP Bound to Signaling with a=zrtp-hash

ZRTP provides a mechanism to bind the ZRTP exchange to information in
the signaling.  If the signaling is protected with TLS, this makes it
difficult for an attacker who does not control the server to
compromise the call in any case.

We expect the a=zrtp-hash to be present in any signaling interaction
preceding a successful ZRTP negotiation.

If ZRTP negotiates but there was no a=zrtp-hash, or the a=zrtp-hash
does not match the ZRTP negotiation, then the call is much less secure
and the user must be notified of this in a similar manner to a cache
mismatch, described below.

If there is an a=zrtp-hash value and it does match our negotiation,
then the connection is at least as secure as Secure to Server, though
some particular states below may give us enough information to raise
additional alarms.

### The Short Authentication String (SAS)

On each call the user will see two words (or infrequently four
letters) on her screen.  The other caller will also see two words.
These words are expected to match.

The user interface must provide the user with some way to indicate
that the words have been compared and the outcome of that comparison.

Until the user has verified the SAS, the security state of the call is
indeterminate.  Unless we're raising one of the alarms described
below, or above, we can assert the call is at least as secure as a
Secure to Server call, but possibly not much more.

When the user chooses to verify the SAS, she might be prompted, e.g.:

> Please compare the words on screen with those seen by the other
> caller.  Read the words aloud to each other and verify you see the
> same words.  Select:
>
>   They match | Not a match

If the words match, the call is ZRTP Secure -- the highest level of
assurance, and we can set the Verified flag as described below.

If the words do not match, the call is being wiretapped.  This is an
SAS mismatch.  The user is under an active attack.  This is the worst
possible security state -- even worse than unencrypted media.  We must
communicate to the user in clear and convincing language that she
cannot trust this call and must evaluate her security.  e.g.:

> If you and the other caller do not see the same words on screen,
> this call is NOT secure.  Someone is actively trying to intercept
> your call.  Do not discuss sensitive information.
>
> We would like to hear about this incident.  If you can, please
> contact support and tell us you've had an "SAS mismatch."  We will
> treat this as serious.  We'll help you improve your security, and
> your report will help the security of others.

### The Cache Name

After the user verifies the SAS, she must be prompted to enter the
name of the remote party.  This value can be pre-populated by the SIP
display name or by an entry in the user's address book, but the user
must be given the opportunity to see and edit the value.

This value is called the cache name.  Once it has been set, it must be
prominently displayed to the user on this call and in all future
calls.  The user can be allowed to edit this value at any time.

That the user confirms the name of the remote party, and that this
name is displayed prominently, is critical for security.  There is a
trivial and well known method of performing a man-in-the-middle (MitM)
attack against the user if this value is not displayed.

### The Verified Flag

When a user indicates on the UI that she has successfully verified the
SAS with the other party, the call is now considered Verified locally,
and a flag (the "V flag") is set in a database and will be sent to the
remote peer on the next call.

If both users have verified the SAS on a previous call then the call
is considered Verified and ZRTP Secure without requiring the local
user to verify the SAS -- but only if we have an cache match as we'll
see below.

### The Cache: RS1

After two users make a call, ZRTP will retain for each some shared
secret information on both devices.  This secret is the same on both
sides and is called RS1 or the cached secret.

On the first call between two users both clients will see that there
is no cached secret.  In this case, the Verified state must not be
shown until the user verifies the SAS on the current call.

On subsequent calls the local cached secret will not be empty.  Once
ZRTP has negotiated we'll know whether this value matches the value on
the other end.  Only if it does and both the local and the remote V
flag are set may we show the Verified state at the outset of the call.

If we have a local cached secret and it does not match the cached
secret of our remote peer, then we have a cache mismatch.  This is a
serious and unexpected condition.  The user's security will be
compromised until she verbally compares the SAS with her partner and
informs the UI this has been done.  We must strongly encourage the
user to complete these steps by e.g. automatically presenting the SAS
comparison modal dialog.  We should prefix the normal SAS comparison
dialog by a warning, e.g.:

> There has been an unexpected error while securing your call.  To
> ensure the security of this and future calls, you must verify some
> information with the other caller.

Unlike in the normal case where a call can be treated as having
indeterminate security until the SAS has been verified, in the case of
a cache mismatch the call must be treated as insecure until the SAS is
verified.

Once the user indicates the words do match, we can set the Verified
state; we are now ZRTP Secure.

If the words do not match, we must present a strong warning as shown
above for an SAS mismatch.

If the user does not verify the SAS by the end of the call, the next
call will show a cache mismatch as well.

## Summary

The highest level of security is ZRTP Secure.  It can be shown when
the users verify the SAS on the current call or when they've done so
on earlier calls and we have a cache match.

Users must be advised of the severity of an SAS mismatch.

Users with a ZRTP session who have not verified the SAS have an
indeterminate security state.  If there is no cache this state is
approximately as secure as Secure to Server.  If there is a cache
mismatch the call must be treated as insecure until the users compare
the SAS.

ZRTP stores a cache name for each remote party, and this name must be
displayed prominently to the user to ensure security.

When ZRTP has not been negotiated but TLS and SDES are in place, the
call is Secure to Server.  This is the lowest form of acceptable
security.

Security states known to the software to be lower than Secure to
Server should not routinely happen, but if they do, the user must be
clearly alerted.

---

*Document Version: v0.1*

*Document ID: fcd6839b-b2fb-4e06-8f85-c6c1dcb7e263*
