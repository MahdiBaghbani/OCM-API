---
title: 'Open Cloud Mesh'
docname: draft-lopresti-open-cloud-mesh-00
category: std

ipr: trust200902
area: Security
keyword: Internet-Draft

stand_alone: yes

author:
  - ins: G. Lo Presti
    name: Giuseppe Lo Presti
    organization: CERN
    email: giuseppe.lopresti@cern.ch
    uri: http://cern.ch/lopresti

  - ins: M.B. de Jong
    name: Michiel de Jong
    organization: Ponder Source
    email: michiel@pondersource.com
    uri: https:/pondersource.com

  - ins: M. Baghbani
    name: Mahdi Baghbani
    organization: Ponder Source
    email: mahdi@pondersource.com
    uri: https:/pondersource.com

  - ins: M. Nordin
    name: Micke Nordin
    organization: SUNET
    email: kano@sunet.se
    uri: https://code.smolnet.org/micke

--- abstract

Open Cloud Mesh is a server federation protocol that is used to notify a Receiving Party that they have
been granted access to some Resource. It has similarities with authorization flows such as OAuth, as well as with social internet protocols such as ActivityPub and email.

Open Cloud Mesh only handles the necessary interactions up to the point where the Receiving Party is informed that they were granted access to the Resource. The actual resource access is then left to protocols such as WebDAV and others.

--- middle

# Terms
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
RFC 2119.

We define the following concepts (with some non-normative references to related concepts from OAuth and elsewhere):

* __Resource__ - the piece of data or interaction to which access is being granted, e.g. a file, folder, video call, or printer queue
* __Share__ - a policy rule stating that certain actors are allowed access to a Resource. Also: a record in a database representing this rule
* __Sending Party__ - a person or party who is authorized to create Shares (similar to "Resource Owner" in OAuth)
* __Receiving Party__ - a person, group or party who is granted access to the Resource through the Share (similar to "Requesting Party / RqP" in OAuth-UMA)
* __Sending Server__ - the server that:
  * holds the Resource ("file server" or "Entreprise File Sync and Share (EFSS) server" role),
  * provides access to it (by exposing at least one "API"),
  * takes the decision to create the Share based on user interface gestures from the Sending Party (the "Authorization Server" role in OAuth)
  * takes the decision about authorizing attempts to access the Resource (the "Resource Server" role in OAuth)
  * sends out Share Creation Notifications when appropriate (see below)
* __Receiving Server__ - the server that:
  * receives Share Creation Notifications (see below)
  * actively or passively notifies the receiving user or group of any incoming Share Creation Notification
  * acts as an API client, allowing the receiving user to access the Resource through an API (e.g. WebDAV) of the sending server
* __Sending Gesture__ - a user interface interaction from the Sending Party to the Sending Server, conveying the intention to create a Share
* __Share Creation__ - the addition of a Share to the database state of the Sending Server, in response to a successful Sending Gesture or for another reason
* __Share Creation Notification__ - a server-to-server request from the sending server to the receiving server, notifying the receiving server that a Share has been created
* __FQDN__ - Fully Qualified Domain Name, such as `"cloud.example.com"`
* __OCM Server__ - a server that supports OCM.
* __Discovering Server__ - a server that tries to obtain information in OCM API discovery
* __Discoverable Server__ - a server that tries to supply information in OCM API discovery
* __OCM Address__ - a string of the form `<Receiving Party's identifier>@<fqdn>` which can be used to uniquely identify a user or group "at" an OCM Server. `<Receiving Party's identifier>` is an opaque string,
unique at the server. `<fqdn>` is the Fully Qualified Domain Name by which the server is identified. This can, but doesn't need to be, the domain at which the OCM API of that server is hosted.
* __Vanity OCM Address__ - a string that looks like an OCM Address but with an alternative (generally shorter or nicer) FQDN. This FQDN does not support HTTP-based discovery, but it does provide an SRV record in its DNS zone, pointing to the (generally longer or uglier) FQDN of the OCM Server.
* __Regular OCM Address__ - an OCM Address that is not a Vanity OCM Address
* __OCM Notification__ - a message from the Receiving Server to the Sending Server or vice versa, using the OCM Notifications endpoint.
* __Invite Message__ - out-of-band message used to establish contact between parties and servers in the Invite Flow, containing an Invite Token (see below) and the Invite Sender's OCM Address
* __Invite Sender__ - the party sending an Invite
* __Invite Receiver__ - the party receiving an Invite
* __Invite Sender OCM Server__ - the server holding an address book used by the Invite Sender, to which details of the Invite Receiver are to be added
* __Invite Receiver OCM Server__ - the server holding an address book used by the Invite Receiver, to which details of the Invite Sender are to be added
* __Invite Token__ - a hard-to-guess string used in the Invite Flow, generated by the Invite Sender OCM Server and linked uniquely to the Invite Sender's OCM Address
* __Invite Creation Gesture__ - gesture from the Invite Sender to the Invite Sender OCM Server, resulting in the creation of an Invite Token.
* __Invite Acceptance Gesture__ - gesture from the Invite Receiver to the Invite Receiver OCM Server, supplying the Invite Token as well as the OCM Address of the Invite Sender, effectively allowlisting the Invite Sender OCM Server for sending Share Creation Notifications to the Invite Receiver OCM Server.
* __Invite Acceptance Request__ - API call from the Invite Receiver OCM Server to the Invite Sender OCM Server, supplying the Invite Token as well as the OCM Address of the Invite Receiver, effectively allowlisting the Invite Sender OCM Server for sending Share Creation Notifications to the Invite Receiver OCM Server.
* __Invite Acceptance Response__ - HTTP response to the Invite Acceptance Request
* __Share Name__ - a human-readable string, provided by the Sending Party or the Sending Server, to help the Receiving Party understand which Resource the Share grants access to
* __Share Permissions__ - protocol-specific allowances granted to the Receiving Party on the modes of accessing the Resource
* __Share Requirements__ - protocol-specific restrictions on the modes of accessing the Resource

# General Flow
The lifecycle of an Open Cloud Mesh Share starts with prerequisites such as
establishing trust, establishing contact, and OCM API discovery.

Then the share creation involves the Sending Party making a Sending Gesture to the Sending Server,
the Sending Server carrying out the actual Share Creation,
and the Sending Server sending a Share Creation Notification to the Receiving Server.

After this, the Receiving Server MAY notify the Receiving Party and/or the Sending Server, and will act as an API client
through which the Receiving Party can access the Resource. After that, the Share may be updated, deleted, and/or reshared.

# Establishing Contact
Before the Sending Server can send a Share Creation Notification to the Receiving Server, it needs to establish the Receiving Party's OCM Address (containing the Receiving Server's FQDN, and the Receiving Party's identifier), among other things.
Some steps may preceed the Sending Gesture, allowing the Sending Party to establish (with some level of trust) the OCM Address of the Receiving Party. In other cases, establishing the OCM Address
of the Receiving Party happens as part of the Sending Gesture.

## Direct Entry
The simplest way for this is if the Receiving Party shares their OCM Address with the Sending Party through some out-of-band means, and the Sending Party enters this string into the user interface of the Sending Server, by means of typing or pasting into an HTML form, or clicking a link to a URL that includes the string in some form.

## Address books
The Sending Server MAY offer the Sending Party an address book tool, where OCM Addresses can be stored over time in a labeled and/or searchable way. This decouples the act by which the OCM Address string is passed into the Sending Server's database from the selection of the Receiving Party in preparation for Share Creation.

## Public Link Flow
An interface for anonymously viewing a Resource on the Sending Server MAY allow any internet user to type or paste an OCM address into an HTML form, as a Sending Gesture. This means that the Sending Party and the Receiving Party could be the same person, so contact between them does not need to be explicitly established.

## Public Invite Flow
Similarly, an interface on the Sending Server MAY allow any internet user to type or paste an OCM address into an HTML form, as a Sending Gesture for a given Resource, without itself providing a way to access that particular Resource. A link to this interface could then for instance be shared on a mailing list, allowing all subscribers to effectively request access to the Resource by making a Sending Gesture to the Sending Server with their own OCM Address.

## Invite Flow
### Rationale
Many methods for establishing contact allow unsolicited contact with the prospective Receiving Party whenever that party's OCM Address is known. The Invite Flow requires the Receiving Party to explicitly accept it before it can be used, which establishes bidirectional trust between the two parties involved.

OCM Servers MAY enforce a policy to only accept Shares between such trusted contacts, or MAY display a warning to the Receiving Party when a Share Creation Notification from an unknown Sending Party is received

### Steps

* the Invite Sender OCM Server generates a unique Invite Token and helps the Invite Sender to create the Invite Message
* the Invite Sender uses some out-of-band communication to send the Invite Message, containing the Invite Token and the Invite Sender OCM Server FQDN, to the Invite Receiver
* the Invite Receiver navigates to the Invite Receiver OCM Server (possibly using a Where-Are-You-From page provided as part of the Invite Message) and makes the Invite Acceptance Gesture
* the Invite Receiver OCM Server discovers the OCM API of the Invite Sender OCM Server using generic OCM API Discovery (see section below)
* the Invite Receiver OCM Server sends the Invite Acceptance Request to the Invite Sender OCM Server

### Invite Acceptance Request Details
Whereas the precise syntax of the Invite Message and the Invite Acceptance Gesture will differ between implementations, the Invite Acceptance Request SHOULD be a HTTP POST request:

* to the `/invited-accepted` path in the Invite Sender OCM Server's OCM API
* using `application/json` as the `Content-Type` HTTP request header
* its request body containing a JSON document representing an object with the following string fields:
  * `recipientProvider` - FQDN of the Invite Receiver OCM Server
  * `token` - the Invite Token. The Invite Sender OCM Server SHOULD recall which Invite Sender OCM Address this token was linked to
  * `userID` - the Invite Receiver's identifier at their OCM Server
  * `email` - non-normative / informational; an email address for the Invite Receiver. Not necessarily at the same FQDN as their OCM Server
  * `name` - human-readable name of the Invite Receiver, as a suggestion for display in the Invite Sender's address book
* using TLS
* using [httpsig](https://datatracker.ietf.org/doc/html/draft-cavage-http-signatures-12)

The Invite Receiver OCM Server SHOULD apply its own policies for trusting the Invite Sender OCM Server before making the Invite Acceptance Request.

Since the Invite Flow does not require either Party to type or remember the `userID`, this string does not need to be human-memorable. Even if the Invite Receiver has a memorable username at the Invite Receiver OCM Server, this `userID` that forms part of their OCM Address does not need to match it.

Also, a different `userID` could be given out to each contact, to avoid correlation of identities.

### Invite Acceptance Response Details
The Invite Acceptance Response SHOULD be a HTTP response:

* in response to the Invite Acceptance Request
* using `application/json` as the `Content-Type` HTTP response header
* its response body containing a JSON document representing an object with the following string fields:
  * `userID` - the Invite Sender's identifier at their OCM Server
  * `email` - non-normative / informational; an email address for the Invite Sender. Not necessarily at the same FQDN as their OCM Server
  * `name` - human-readable name of the Invite Sender, as a suggestion for display in the Invite Receiver's address book

A 200 response status means the Invitation Acceptance Request was successful.
A 400 response status means the Invitation Token is invalid or does not exist.
A 403 response status means the Invite Receiver OCM Server is not trusted to accept this Invite.
A 409 response status means the Invite was already accepted.

The Invite Sender OCM Server SHOULD verify the HTTP Signature on the Invite Acceptance Request and apply its own policies for trusting the Invite Receiver OCM Server before processing the Invite Acceptance Request and sending the Invite Acceptance Response.

As with the `userID` in the Invite Acceptance Request, the one in the Response also doesn't need to be human-memorable, doesn't need to match the Invite Sender's username at their OCM Server.

### Addition into address books
Following these step, both servers MAY display the `name` of the other party as a trusted or allowlisted contact, and enable selecting them as a Receiving Party. OCM Servers MAY enforce a policy to only accept Share Creation Notifications from such trusted contacts, or MAY display a warning to users when a Share Creation Notification from an unknown party is received.

Both servers MAY also allowlist each other as a server with which at least one of their users wishes to interact.

Note that Invites act symmetrically, so once contact has been established, both the Invite Sender and the Invite Receiver may take on either the Sending Party or the Receiving Party role in subsequent Share Creation events.

Both parties may delete the other party from their address book at any time without notifying them.

### Security Advantages
It is important to underscore the value of the Invite in this scenario, as it provides four important security advantages. First of all, if the Receiving Server blocks Share Creation Notifications from Sending Parties who are not in the address book of the Receiving Party, then this protects the Receiving Party from receiving unsolicited Shares. An attacker could still send the Receiving Party an unsolicited Share, but they would first need to convince the Receiving Party through an out-of-band communication channel to accept their invite. In many use cases, the Receiving Party has had other forms of contact with the Sending Party (e.g. in-person or email back-and-forth). The out-of-band Invite Message thus leverages the filters and context which the Receiving Party may already benefit from in that out-of-band communication. For instance, a careful Receiving Party may choose to only accept Invites that reach them via a private or moderated messaging platform.

Second, when the Receiving Party accepts the Invite, the Receiving Server knows that the Sending Server they are about to interact with is trusted by the Sending Party, which in turn is trusted by the Receiving Party, which in turn is trusted by them. In other words, one of their users is requesting the allowlisting of a server they wish to interact with, in order to interact with a party they know out-of-band. This gives the Receiving Server reason to put more trust in the Sending Server than it would put into an arbitrary internet-hosted server.

Third, equivalently, the Sending Server knows it is essentially registering the Receiving Server as an API client at the request of the Receiving Party, to whom the right to request this has been traceably delegated by the Sending Party, which is one of its registered users.

Fourth, related to the second one, it removes the partial 'open relay' problem that exists when the Sending Server is allowed to include any Receiving Server FQDN in the Sending Gesture. Without the use of Invites, a Distributed Denial of Service attack could be organised if many internet users collude to flood a given OCM Server with Share Creation Notifications which will be hard to distinguish from legitimate requests without human interaction. An unsolicited (invalid) Invite Acceptance Request is much easier to filter out than an unsolicited (possibly valid, possibly invalid) OCM request, since the Invite Acceptance Request needs to contain an Invite Token that was previously uniquely generated at the Invite Sender OCM server.

# OCM API Discovery
## Introduction
After establishing contact as discussed in the previous section, the Sharing User can send the Share Creation Gesture to the Sending Server, providing the Sending Server with the following information:

* Resource to be shared
* Protocol to be offered for access
* Sending Party's identifier
* Receiving Party's identifier
* Receiving Server FQDN
* OPTIONAL: Share Requirements
* OPTIONAL: Share Name
* OPTIONAL: Share Permissions

The next step is for the Sending Server to additionally discover:

* if the Receiving Server is trusted
* if the Receiving Server supports OCM
* if so, which version and with which optional functionality
* at which URL
* the public key the Receiving Server will use for HTTP Signatures (if any)

The Sending Server MAY first perform denylist and allowlist checks on the FQDN.

If a finite allowlist of Receiving Servers exists on the Sending Server side, then this list may already contain all necessary information.

If the FQDN passes the denylist and/or allowlist checks, but no details about its OCM API are known, the Sending Server can use the following process to try to fetch this information from the Receiving Server.

This process MAY be influenced by a VPN connection and/or IP allowlisting.

When OCM API discovery can occur in preparation of a Share Creation Notification, the Sending Server takes on the 'Discovering Server' role and the Receiving Server plays the role of 'Discoverable Server'.

## Process
At the start of the process, the Discovering Server has either an OCM Address, or just an FQDN from for instance the `recipientProvider` field of an Invite Acceptance Request.

Step 1: In case it has an OCM Address, it should first extract `<fqdn>` from it (the part after the `@` sign).
Step 2: The Discovering Server SHOULD attempt OCM API discovery a HTTP GET request to `https://<fqdn>/.well-known/ocm`.
Step 3: If that results in a valid HTTP response with a valid JSON response body within reasonable time, go to step 8.
Step 4: If not, try a HTTP GET with `https://<fqdn>/ocm-provider` as the URL instead.
Step 5: If that results in a valid HTTP response with a valid JSON response body within reasonable time, go to step 8.
Step 6: If not, and the `<fqdn>` came from an OCM Address, then it SHOULD check if the OCM Address in question was a Vanity OCM Address.
This can be checked with a `type=SRV` DNS query to `_ocm._tcp.<fqdn>`. If that returns `service = 10 10 443 <regular-fqdn>` then repeat from step 2, using `<regular-fqdn>` instead of the `<fqdn>` that appeared in the Vanity OCM Address.
Step 7: If not, fail.
Step 8: The JSON response body is the data that was discovered.

## Fields
The JSON response body offered by the Discoverable Server SHOULD contain the following information about its OCM API:

* REQUIRED: enabled (boolean) - Whether the OCM service is enabled at this endpoint
* REQUIRED: apiVersion (string) - The OCM API version this endpoint supports. Example: `"1.1.0"`
* REQUIRED: endPoint (string) - The URI of the OCM API available at this endpoint. Example: `"https://my-cloud-storage.org/ocm"`
* OPTIONAL: provider (string) - A friendly branding name of this endpoint. Example: `"MyCloudStorage"`
* REQUIRED: resourceTypes (array) - A list of all supported resource types with their access protocols. Each item in this list should
itself be an object containing the following fields:
  * name (string) -  A supported resource type (file, folder, calendar, contact, ...).
                Implementations MUST support `file` at a minimum. Each resource type is identified by its `name`: the list MUST NOT
          contain more than one resource type object per given `name`.
  * shareTypes (array of string) -
                The supported recipient share types.
                MUST contain `"user"` at a minimum, plus optionally `"group"` and `"federation"`.
                Example: `["user"]`
  * protocols (object) - The supported protocols for accessing shared resources.
                Implementations MUST support at least `webdav` for `file` resources,
                any other combination of resources and protocols is optional. Example:
                ```json
                {
                  "webdav": "/remote/dav/ocm/",
                  "webapp": "/app/ocm/",
                  "talk": "/apps/spreed/api/"
                }
                ```
                Fields:
    * webdav (string) - The top-level WebDAV path at this endpoint. In order to access
                    a remote shared resource, implementations MAY use this path
                    as a prefix, or as the full path (see sharing examples).
    * webapp (string) - The top-level path for web apps at this endpoint. This value
                    is provided for documentation purposes, and it SHALL NOT
                    be intended as a prefix for share requests.
    * datatx (string) - The top-level path to be used for data transfers. This
                    value is provided for documentation purposes, and it SHALL
                    NOT be intended as a prefix. In addition, implementations
                    are expected to execute the transfer using WebDAV as
                    the wire protocol.
    * Any additional protocol supported for this resource type MAY
                    be advertised here, where the value MAY correspond to a top-level
                    URI to be used for that protocol.
              
* OPTIONAL: capabilities (array of string) - The optional capabilities supported by this OCM Server.
          As implementations MUST accept Share Creation Notifications to be compliant,
          it is not necessary to expose that as a capability.
          Example: `["/notifications"]`. The array MAY include for instance:
    * `"/notifications"` - to indicate this OCM server is capable of processing OCM Notifications
    * `"/invite-accepted"` - to indicate that this OCM server is capable of processing Invite Acceptance Requests.
    * `"/mfa-capable"` - to indicate that this OCM server can apply a Sending Server's MFA requirements for a Share on their behalf.
        
* OPTIONAL: publicKey (object) - The signatory used to sign outgoing request to confirm its origin. The 
          signatory is optional, but if present, it MUST contain two string fields, `id` and `publicKeyPem`.
        properties:
  * REQUIRED id (string) unique id of the key in URI format. The hostname set the origin of the 
              request and MUST be identical to the current discovery endpoint.
            Example: https://my-cloud-storage.org/ocm#signature
  * REQUIRED publicKeyPem (string) - PEM-encoded version of the public key.
            Example: "-----BEGIN PUBLIC KEY-----\nMII...QDD\n-----END PUBLIC KEY-----\n"

# Share Creation Notification
To create a share, the sending server SHOULD make a HTTP POST request

* to the `/shares` path in the Receiving Server's OCM API
* using `application/json` as the `Content-Type` HTTP request header
* its request body containing a JSON document representing an object with the fields as described below
* using TLS
* using [httpsig](https://datatracker.ietf.org/doc/html/draft-cavage-http-signatures-12)

## Fields

* REQUIRED shareWith (string)
          Consumer specific identifier of the user, group or federation the provider
          wants to share the resource with. This is known in advance.
          Please note that the consumer service endpoint is known in advance
          as well, so this is no part of the request body.
        Example: "51dc30ddc473d43a6011e9ebba6ca770@geant.org"
* REQUIRED name (string)
       Name of the resource (file or folder).
        Example: "resource.txt"
* OPTIONAL description (string)
        Optional description of the resource (file or folder).
        Example: "This is the Open API Specification file (in YAML format) of the Open
          Cloud Mesh API."
* REQUIRED providerId (string)
          Identifier to identify the shared resource at the provider side. This is
          unique per provider such that if the same resource is shared twice,
          this providerId will not be repeated.
        Example: 7c084226-d9a1-11e6-bf26-cec0c932ce01
* REQUIRED owner (string) -
          Provider specific identifier of the user who owns the resource.
        Example: "6358b71804dfa8ab069cf05ed1b0ed2a@apiwise.nl"
* REQUIRED sender (string) -
          Provider specific identifier of the user that wants to share the
          resource. Please note that the requesting provider is being
          identified on a higher level, so the former `remote` property
          is not part of the request body.
        Example: "527bd5b5d689e2c32ae974c6229ff785@apiwise.nl"
* OPTIONAL ownerDisplayName (string)
          Display name of the owner of the resource
        Example: "Dimitri"
* OPTIONAL senderDisplayName (string)
          Display name of the user that wants to share the resource
        Example: "John Doe"
* REQUIRED shareType (string)
        SHOULD have a value of "user", "group", or "federation", to indicated that the first part
        of the `shareWith` OCM Address refers to a Receiving Party who is a single user of the Receiving Server,
        a group of users at the Receiving Servers, or a group of users that is spread out over various servers,
        including at least one user at the Receiving Server.
* REQUIRED resourceType (string)
          Resource type (file, folder, calendar, contact, ...)
* OPTIONAL expiration (integer)
          The expiration time for the OCM share, in seconds
          of UTC time since Unix epoch. If omitted, it is assumed
          that the share does not expire.
* OPTIONAL code (string)
          A nonce to be exchanged for a (potentially short-lived) bearer token at the Sending Server's /token endpoint.
* REQUIRED protocol (object)
          JSON object with specific options for each protocol.
          The supported protocols are:
          - `webdav`, to access the data
          - `webapp`, to access remote web applications
          - `datatx`, to transfer the data to the remote endpoint

          Other custom protocols might be added in the future.

          In case a single protocol is offered, there are three ways to specify this object:
          Option 1: Set the `name` field to the name of the protocol, and put the protocol
          details in a field named `options`.
          Option 2: Set the `name` field to the name of the protocol, and put the protocol
          details in a field carrying the name of the protocol.
          Option 3: Set the `name` field to `multi`, and put the protocol
          details in a field carrying the name of the protocol.

                Option 1 using the `options` field now deprecated. Implementations are encouraged to
                transition to the new optional properties defined below, such that
                this field may be removed in a future major version of the spec.

          When specifying more than one protocol as different ways to access the Share, the `name`
          field needs to be set to `multi`.

If `multi` is given, one or more protocol
                endpoints are expected to be defined according to the optional
                properties specified below.
                Otherwise, at least `webdav` is expected to be supported, and
                its options MAY be given in the opaque `options` payload for
                compatibility with v1.0 implementations (see examples). Note
                though that this format is deprecated.
                Warning: client implementers should be aware that v1.1 servers
                MAY support both `webdav` and `multi`, but v1.0 servers MAY
                only support `webdav`.

   * Protocol details for `webdav` MAY contain:
     * OPTIONAL sharedSecret (string) - required if no `code` field is given for the Share as a whole (see above).
                    An optional secret to be used to access the resource,
                    such as a bearer token.
                    To prevent leaking it in logs it MUST NOT appear in any URI.
     * OPTIONAL permissions (array of strings) -
                      The permissions granted to the sharee. A subset of:
                      - `read` allows read-only access including download of a copy.
                      - `write` allows create, update, and delete rights on the resource.
                      - `share` allows re-share rights on the resource.
                      - `mfa-enforced` requires the user accessing the resource to be
                        MFA-authenticated. This permission MAY be used if the
                        provider exposes the `mfa-capable` capability.
     * OPTIONAL uri (string)
                    An URI to access the remote resource. The URI MAY be relative,
                    in which case the prefix exposed by the `/ocm-provider` endpoint MUST
                    be used, or it may be absolute (recommended). Additionally, the URI
                    MAY include a secret hash in the path, in which case there MAY be
                    no associated `sharedSecret`.
   * Protocol details for `webapp` MAY contain:
     * REQUIRED uriTemplate (string)
                    A templated URI to a client-browsable view of the shared resource,
                    such that users may use the web applications available at the site.
                    The URI MAY include a secret hash in the path. If the path includes
                    a `{relative-path-to-shared-resource}` placeholder, implementations
                    MAY replace it with the actual path to ease user interaction.
     * REQUIRED viewMode (string)
                    The permissions granted to the sharee. A subset of:
                    - `view` allows access to the web app in view-only mode.
                    - `read` allows read and download access via the web app.
                    - `write` allows full editing rights via the web app.
                    - `mfa-enforced` requires the user accessing the resource to be
                      MFA-authenticated. This permission MAY be used if the
                      provider exposes the `mfa-capable` capability.
     * OPTIONAL sharedSecret (string)
                    An optional secret to be used to access the remote web app,
                    for example in the form of a bearer token.
   * Protocol details for `datatx` MAY contain:
     * OPTIONAL sharedSecret (string)
                    An optional secret to be used to access the resource,
                    for example in the form of a bearer token.
                    To prevent leaking it in logs it MUST NOT appear in any URI.
     * REQUIRED srcUri (string)
                    An URI to access the remote resource. The URI MAY be relative,
                    in which case the prefix exposed by the `/ocm-provider` endpoint MUST
                    be used, or it may be absolute (recommended). Additionally, the
                    URI MAY include a secret hash in the path.
     * OPTIONAL size (integer)

## Decision to Discard
The Receiving Server MAY discard the notification if any of the following hold true:

* the HTTP Signature is missing
* the HTTP Signature is not valid
* no keypair is trusted or discoverable from the FQDN part of the `sender` field in the request body
* the keypair used to generate the HTTP Signature doesn't match the one trusted or discoverable from the FQDN part of the `sender` field in the request body
* the Sending Server is denylisted
* the Sending Server is not allowlisted
* the Sending Party is not trusted by the Receiving Party (e.g. no Invite was exchanged and/or the Sending Party's OCM Address does not appear in the Receiving Party's address book)
* the Receiving Server is unable to act as an API client for (any of) the protocol(s) listed for accessing the resource
* an initial check shows that the resource cannot successfully accessed through (any of) the protocol(s) listed

# Receiving Party Notification
If the Share Creation Notification is not discarded by the Receiving Server, they MAY notify the Receiving Party passively by adding the Share to some inbox list, and MAY also notify them actively through for instance a push notification or an email message.

They could give the Receiving Party the option to accept or reject the share, or add the share automatically and only send an informational notification that this happened.

# Share Acceptance Notification
In response to a Share Creation Notification, the Receiving Server MAY discover the OCM API of the Sending Server,
starting from the `<fqdn>` part of the `sender` field in the Share Creation Notification.
Note that the `sender` field is not allowed to contain a Vanity OCM Address, so the fallback to SRV DNS records is not necessary in this OCM API discovery process.

If the OCM API of the Sending Server is successfully discovered, the Receiving Server MAY
make a HTTP POST request

* to the `/notifications` path in the Sending Server's OCM API
* using `application/json` as the `Content-Type` HTTP request header
* its request body containing a JSON document representing an object with the fields as described below
* using TLS
* using [httpsig](https://datatracker.ietf.org/doc/html/draft-cavage-http-signatures-12)

## Fields

* REQUIRED notificationType (string) - in a Share Acceptance Notification it SHOULD be one of:
  * 'SHARE_ACCEPTED'
  * 'SHARE_DECLINED'
* REQUIRED providerId (string) - copied from the Share Creation Notification for the Share this notification is about
* OPTIONAL resourceType (string) - copied from the Share Creation Notification for the Share this notification is about
* OPTIONAL notification (object) - optional additional parameters, depending on the notification and the resource type


### Receiving Party Notification
If the Share Creation Notification is not discarded by the Receiving Server, they MAY notify the Receiving Party passively by adding the Share to some inbox list, and MAY also notify them actively through for instance a push notification or an email message.

They could give the Receiving Party the option to accept or reject the Share, or add the Share automatically and only send an informational notification that this happened.

# Resource Access
To access the Resource, the Receiving Server MAY use multiple ways, depending on the body of the Share Creation Notification and on the `protocol.name` property in there:

* If `protocol.name` = `multi`, the receiver MUST make a HTTP PROPFIND request to `protocol.webdav.uri` to access the remote share.
If `code` is not empty, the receiver SHOULD discover the sender's OCM endpoint and make a signed POST request to the `/token` path inside the Sending Server's OCM API, to exchange
the code for a short-lived bearer token,
and then use that bearer token to access the Resource.
Otherwise, if `protocol.webdav.sharedSecret` is not empty, the receiver MUST pass it as a `Authorization: bearer` header.

* If `protocol.name` = `webdav`, the receiver SHOULD inspect the `protocol.options` property. If it contains a `sharedSecret`, as in the [legacy example](https://cs3org.github.io/OCM-API/docs.html?branch=develop&repo=OCM-API&user=cs3org#/paths/~1shares/post), then the receiver SHOULD make a HTTP PROPFIND request to `https://<sharedSecret>:@<host><path>`, where `<host>` is the remote server, and `<path>` is obtained by querying the [Discovery](#ocm-api-discovery) endpoint at the Sending Server and getting `resourceTypes[0].protocols.webdav`. Note that this access method is _deprecated_ and may be removed in a future release of the Protocol.

In both cases, when the Resource is a folder and the Receiving Server accesses a resource within that shared folder, it SHOULD append its relative path to that URL.

Additionally, if `protocol.<protocolname>.requirements` include `mfa-enforced`, the Receiving Server MUST ensure that the Receiving Party has been authenticated with MFA.

# Share Deletion
A `"SHARE_ACCEPTED"` notification followed by a `"SHARE_UNSHARED"` notification is
equivalent to a `"SHARE_DECLINED"` notification.

Note that the Sending Server MAY at any time revoke access to a Resource (effectively undoing or deleting the Share) without notifying the Receiving Server.

# Share Updating
TODO: document `"RESHARE_CHANGE_PERMISSION"`

# Resharing
The `"REQUEST_RESHARE"` and `"RESHARE_UNDO"` notification types MAY be used by the
Receiving Server to persuade the Sending Server to share the same Resource with another Receiving Party.
TODO: document how the Receiving Party can know if the Sending Party understood and processed the
reshare request.

## Appendix A: Multi Factor Authentication
If a Receiving Server exposes the capability `/mfa-capable`, it indicates that it will try and comply with a MFA requirement set on a Share. If the Sending Server trusts the Receiving Server, the Sending Server MAY set the requirement `mfa-enforced` on a Share, which the Receiving Server MUST honor. A compliant Receiving Server that signals that it is MFA-capable MUST not allow access to a resource protected with the `mfa-enforced` requirement, if the Receiving Party has not provided a second factor to establish their identity with greater confidence.

Since there is no way to guarantee that the Receiving Server will actually enforce the MFA requirement, it is up to the Sending Server to establish a trust with the Receiving Server such that it is reasonable to assume that the Receiving Server will honor the MFA requirement. This establishment of trust will inevitably be implementation dependent, and can be done for example using a pre approved allow list of trusted Receiving Servers. The procedure of establishing trust is out of scope for this specification: a mechanism similar to the [ScienceMesh](https://sciencemesh.io) integration for the [Invite](#invite-flow) capability may be envisaged.

# Appendix B: Request Signing

A request is signed by adding the signature in the headers. The sender also needs to expose the public key used to generate the signature. The receiver can then validate the signature and therefore the origin of the request.
To help debugging, it is recommended to also add all properties used in the signature as headers, even if they can easily be re-generated from the payload.

Note: Signed requests prove the identity of the sender but do not encrypt nor affect its payload.

Here is an example of headers needed to sign a request.

~~~~~
  {
    "(request-target)": "post /path",
    "content-length": 380,
    "date": "Mon, 08 Jul 2024 14:16:20 GMT",
    "digest": "SHA-256=U7gNVUQiixe5BRbp4Tg0xCZMTcSWXXUZI2\\/xtHM40S0=",
    "host": "hostname.of.the.recipient",
    "Signature": "keyId=\"https://author.hostname/key\",algorithm=\"rsa-sha256\",headers=\"content-length date digest host\",signature=\"DzN12OCS1rsA[...]o0VmxjQooRo6HHabg==\""
  }
~~~~~

* '(request-target)' contains the reached endpoint and the used method,
* 'content-length' is the total length of the payload of the request,
* 'date' is the date and time when the request has been sent,
* 'digest' is a checksum of the payload of the request,
* 'host' is the hostname of the recipient of the request (remote when signing outgoing request, local on incoming request),
* 'Signature' contains the signature generated using the private key and details on its generation:
  * 'keyId' is a unique id, formatted as an url. hostname is used to retrieve the public key via custom discovery
  * 'algorithm' specify the algorithm used to generate signature
  * 'headers' specify the properties used when generating the signature
  * 'signature' the signature of an array containing the properties listed in 'headers'. Some properties like content-length, date, digest, and host are mandatory to protect against authenticity override.


## How to generate the Signature for outgoing request

After properties are set in the headers, the Signature is generated and added to the list.

This is a pseudo-code example for generating the `Signature` header for outgoing requests:

~~~~~
headers = {
    '(request-target)': 'post /path',
    'content-length': length_of(payload),
    'date': current_gmt_datetime(),  # Use a function to get the current GMT date as 'D, d M Y H:i:s T'
    'digest': 'SHA-256=' + base64_encode(hash('sha256', utf8_encode(payload))),
    'host': 'recipient-fqdn',
}

signed = ssl_sign(concatenate_with_newlines(headers), private_key, 'sha256')
signature = {
    'keyId': 'sender-fqdn',  # The sending server's FQDN; find its public key through OCM API discovery
    'algorithm': 'rsa-sha256',
    'headers': 'content-length date digest host',
    'signature': signed,
}

headers['Signature'] = format_signature(signature)
~~~~~

### How to confirm Signature on incoming request

The first step would be to confirm the validity of each properties:

* `(request-target)` and `host` are immutable to the type of the request and the local/current host,
* `content-length` and `digest` can be re-generated and compared from the payload of the request,
* a maximum TTL must be applied to `date` and current timestamp,
* regarding data contained in the `Signature` header:
  * using `keyId` to get the public key from remote signatory,
  * `headers` is used to generate the clear version of the signature and must contain at least `content-length`, `date`, `digest` and `host`,
  * `signature` is the encrypted version of the signature.

Here is an example of how to verify the signature using the headers, the signature and the public key:

~~~~~
clear = {
    '(request-target)': 'post /path',
    'content-length': length_of(payload),
    'date': 'Mon, 08 Jul 2024 14:16:20 GMT',  # The date used in the verification process
    'digest': 'SHA-256=' + base64_encode(hash('sha256', utf8_encode(payload))),  # Recompute the digest for verification
    'host': 'sender-fqdn',
}

signed = headers['Signature']
verification_result = ssl_verify(concatenate_with_newlines(clear), signed, public_key, 'sha256')

if not verification_result then
    raise InvalidSignatureException
~~~~~

## Validating the payload

Following the validation of the signature, the host should also confirm the validity of the payload, that is ensuring that the actions implied in the payload actually initiated on behalf of the source of the request.

As an example, if the payload is about initiating a new share the file owner has to be an account from the instance at the origin of the request.

--- back
