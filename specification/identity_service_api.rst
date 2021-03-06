.. Copyright (C) 2017 Kamax Sàrl
.. https://www.kamax.io/
..
.. This program is free software: you can redistribute it and/or modify
.. it under the terms of the GNU Affero General Public License as published by
.. the Free Software Foundation, either version 3 of the License, or
.. (at your option) any later version.
..
.. This program is distributed in the hope that it will be useful,
.. but WITHOUT ANY WARRANTY; without even the implied warranty of
.. MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
.. GNU Affero General Public License for more details.
..
.. This file incorporates work covered by the following copyright and  
.. permission notice:  
..
..   Copyright 2016 OpenMarket Ltd
..   Copyright 2017 New Vector Ltd
..
..   Licensed under the Apache License, Version 2.0 (the "License");
..   you may not use this file except in compliance with the License.
..   You may obtain a copy of the License at
..
..     http://www.apache.org/licenses/LICENSE-2.0
..
..   Unless required by applicable law or agreed to in writing, software
..   distributed under the License is distributed on an "AS IS" BASIS,
..   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
..   See the License for the specific language governing permissions and
..   limitations under the License.

.. WARNING::
  This is not the official Matrix spec. This is a documentation of the current
  behaviour of implementations in spec format.

Identity Service API
====================

The Matrix client-server and server-server APIs are largely expressed in Matrix
user identifiers. From time to time, it is useful to refer to users by other
("third-party") identifiers, or "3pid"s, e.g. their email address or phone
number. This identity service specification describes how mappings between
third-party identifiers and Matrix user identifiers can be established,
validated, and used. This description technically may apply to any 3pid, but in
practice has only been applied specifically to email addresses.

.. contents:: Table of Contents
.. sectnum::

Specification version
---------------------

This version of the specification is generated from
`matrix-doc <https://github.com/matrix-org/matrix-doc>`_ as of Git commit
`{{git_version}} <https://github.com/matrix-org/matrix-doc/tree/{{git_rev}}>`_.

General principles
------------------

The purpose of an identity service is to validate, store, and answer questions
about the identities of users. In particular, it stores associations of the form
"identifier X represents the same user as identifier Y", where identities may
exist on different systems (such as email addresses, phone numbers,
Matrix user IDs, etc).

The identity service has some private-public keypairs. When asked about an
association, it will sign details of the association with its private key.
Clients may validate the assertions about associations by verifying the signature
with the public key of the identity service.

In general, identity services are treated as reliable oracles. They do not
necessarily provide evidence that they have validated associations, but claim to
have done so. Establishing the trustworthiness of an individual identity service
is left as an exercise for the client.

3PID types are described in `3PID Types`_ Appendix.

Privacy
-------

Identity is a privacy-sensitive issue. While the identity service exists to
provide identity information, access should be restricted to avoid leaking
potentially sensitive data. In particular, being able to construct large-scale
connections between identities should be avoided. To this end, in general APIs
should allow a 3pid to be mapped to a Matrix user identity, but not in the other
direction (i.e. one should not be able to get all 3pids associated with a Matrix
user ID, or get all 3pids associated with a 3pid).

Status check
------------

{{ping_is_http_api}}

Key management
--------------

An identity service has some long-term public-private keypairs. These are named
in a scheme ``algorithm:identifier``, e.g. ``ed25519:0``. When signing an
association, the Matrix standard JSON signing format is used, as specified in
the server-server API specification under the heading "Signing Events".

In the event of key compromise, the identity service may revoke any of its keys.
An HTTP API is offered to get public keys, and check whether a particular key is
valid.

The identity server may also keep track of some short-term public-private
keypairs, which may have different usage and lifetime characteristics than the
service's long-term keys.

{{pubkey_is_http_api}}

Association Lookup
------------------

{{lookup_is_http_api}}

Establishing Associations
-------------------------

The flow for creating an association is session-based.

Within a session, one may prove that one has ownership of a 3pid.
Once this has been established, the user can form an association between that
3pid and a Matrix user ID. Note that this association is only proved one way;
a user can associate *any* Matrix user ID with a validated 3pid,
i.e. I can claim that any email address I own is associated with
@billg:microsoft.com.

Sessions are time-limited; a session is considered to have been modified when
it was created, and then when a validation is performed within it. A session can
only be checked for validation, and validation can only be performed within a
session, within a 24 hour period since its most recent modification. Any
attempts to perform these actions after the expiry will be rejected, and a new
session should be created and used instead.

The flow is based on the following steps:

1. Create a session by requesting a token for the 3PID address to validate.
2. Perform the said validation by providing the token related to the 3PID address.
3. Once the session validated, bind the Matrix ID to the session.
4. The 3PID mapping is now available for lookup.

Creating a session
~~~~~~~~~~~~~~~~~~

Creating a session is done by providing generic values and 3PID-specific values.

Generic values are:

- ``client_secret``: a client secret to protect the session.
- ``send_attempt``: an optional attempt number, controlling if a token should
  be sent again.
- ``next_link``: An optional next link URL to which the client is redirected
  after validation of the session. 

If a send_attempt is specified, the server will only send an email if the
send_attempt is a number greater than the most recent one which it has seen (or
if it has never seen one), scoped to that email address + client_secret pair.
This is to avoid repeatedly sending the same email in the case of request
retries between the POSTing user and the identity service. The client should
increment this value if they desire a new email (e.g. a reminder) to be sent.

3PID-specific values are specific to the 3PID medium to give flexibility in the
3PID address representation, either as a canonical value or as a set of values
to be parsed/resolved into a canonical value.

If the request is accepted by the identity server, it will return the session ID
to be used in the next actions alongside the client secret, acting as credentials.

Note that Home Servers offer APIs that proxy these API, adding additional
behaviour on top, for example, ``/register/email/requestToken`` is designed
specifically for use when registering an account and therefore will inform
the user if the email address given is already registered on the server.

{{session_token_request_is_http_api}}

Validating ownership of an 3PID
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A user may make a request to the 3PID-specific token submision endpoint to
verify their ownership over the given 3PID.

Such request would typically contain the following set of data:

- ``sid`` the sid for the session, generated by the ``requestToken`` call.
- ``client_secret`` the client secret which was supplied to the ``requestToken`` call.
- ``token`` the token related to the ``requestToken`` call and communicated to the user.

If the provided values are consistent with a set generated by a ``requestToken``
call, ownership of the 3PID address is considered to have been validated. This
does not publish any information publicly, or associate the 3PID address with
any Matrix user ID. Specifically, calls to ``/lookup`` will not show a binding.

Otherwise, an error will be returned.

{{session_token_submit_is_http_api}}

Checking for a validated association
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

{{threepid_validated_is_http_api}}

Publishing a validated association
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

{{threepid_bind_is_http_api}}

Invitation Storage
------------------

.. WARNING::
  Ephemeral keys' specification may be misleading or inaccurate.

An identity service can store pending invitations to a user's 3pid, which will
be retrieved and can be either notified on or look up when the 3pid is
associated with a Matrix user ID.

The service will look up whether the 3pid is bound to a Matrix user ID. If it is,
the request will be rejected. If the medium is not supported, the request will be rejected.

Otherwise, the service will then generate a random string called ``token``, and an ephemeral public key.
The ephemeral public key will be listed as valid on requests to ``/_matrix/identity/api/v1/pubkey/ephemeral/isvalid``.

The service also generates a display name for the inviter, which is a redacted
version of 3PID address which does not leak the full contents of the address.

The service records persistently all of the above information.

Finally, it will generates a notification containing any relevant data
sent to the 3PID address, notifying the user of the invitation.

At a later point, if the owner of that particular 3pid binds it with a Matrix user ID,
the identity server will attempt to make a request to the Matrix user's homeserver
using the endpoint ``/_matrix/federation/v1/3pid/onBind``.

{{invite_store_is_http_api}}

Ephemeral invitation signing
----------------------------

.. WARNING::
  This section may be misleading or inaccurate.

To aid clients who may not be able to perform crypto themselves, the identity service offers some crypto functionality to help in accepting invitations.
This is less secure than the client doing it itself, but may be useful where this isn't possible.

The identity service will happily sign invitation details with a request-specified ed25519 private key for you, if you want it to. It takes URL-encoded POST parameters:
- mxid (string, required)
- token (string, required)
- private_key (string, required): The private key, encoded as `Unpadded base64`_.

It will look up ``token`` which was stored in a call to ``store-invite``, and fetch the sender of the invite. It will then respond with JSON which looks something like::

 {
   "mxid": "@foo:bar.com",
   "sender": "@baz:bar.com",
   "signatures" {
     "my.id.server": {
       "ed25519:0": "def987"
     }
   },
   "token": "abc123"
 }

.. _`Unpadded Base64`:  ../appendices.html#unpadded-base64
.. _`3PID Types`:  ../appendices.html#pid-types
