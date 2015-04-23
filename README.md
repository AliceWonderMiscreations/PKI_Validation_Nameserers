# PKI Validation Nameservers

The Public Key Infrastructure has a design flaw. When a Certificate Authority
revokes a certificate, that information does not always properly make it to
TLS clients that need to validate a certificate.

[Certificate Revocation Lists](http://en.wikipedia.org/wiki/Revocation_list)
do not scale well to the current size of the Internet.

[Online Certificate Status Protocol](http://en.wikipedia.org/wiki/Online_Certificate_Status_Protocol)
(OCSP) is a valuable solution but it poses privacy concerns with the ability of
Certificate Authorities to track what IP addresses are requesting what IP
addresses and have had issues with not being available consistently.

[OCSP Stapling](http://en.wikipedia.org/wiki/OCSP_stapling) helps to address
the privacy concerns but it requires the server software run code that makes a
connection to an external server not under the control of the system
administrator, and that is a potential remote exploit vector if there is a flaw
in the server's OCSP Stapling code and violates the firewall policies of some
web servers. For example I can not run OCSP Stapling due to the potential
security issues that exist.

OCSP Stapling also requires the participation of the server serving the
certificate, servers serving a revoked certificate for fraudulent purposes
will not participate.

Finally OCSP Stapling is really only a solution that works with web servers, it
does not lend itself well to some other protocols that may be using
[X.509](http://en.wikipedia.org/wiki/X.509) certificates.

The DNS system has been shown to scale extremely well. Clients typically make a
request to a local DNS resolver rather than to an authoritative server so the
privacy issue is not there. DNS responses can be secured with DNSSEC making
it a safe system to use distribute information about the status of a
certificate. The server serving secure content does not need to run any
additional code or make any connections to remote sources. Finally, the DNS
system is easy for certificate validation on the client host to use independent
of the protocol being used with the secure communication.

This paper describes a simple yet effective method of using the DNS system to
allow TLS clients to verify that a X.509 certificate has not been revoked by
the certificate authority that signed it.

It is my hope that *someone* with the right contacts takes not of this paper
and that a proper RFC can be written up and put into action.

This is being published on github in the hopes that discussion and improvements
can take plus until this becomes a reality.


## Nutshell Overview

When a TLS client downloads a certificate signed by a participating Certificate
Authority, the client will be able to extract a serial number from the
signed certificate and do a simple DNS query for a TXT record to determine if
the certificate has been revoked or not.

By using the existing DNS system, the results of the query will often be cached
by the DNS resolver that the client is configured to use, greatly reducing the
network traffic needed for a client to have confidence in a certificate.

By using DNSSEC to secure the zones used to provide the results, the client can
have confidence that the result they receive has not been tampered with between
the Certificate Authority's server and the client.

By including a timestamp in the TXT data, the client can know how recently a
valid certificate was declared to still be valid by the certificate authority.

In the event that a valid response can not be obtained, clients can still fall
back to the existing method of using the OCSP protocol to query the validity
from the Certificate Authority.


## Top Level Domain

A new Top Level Domain should be created by ICANN to serve as a utility domain
for PKI Validation nameservers. An easy way for TLS clients to check whether or
not a certificate has been revoked is important enough to the security of the
web that I believe it warrants the creation of a TLD for the purpose.

I would suggest using `pki.` as that TLD is not currently in use and makes the
purpose of this utility TLD fairly obvious to the technically inclined.

Certificate Authorities would then register a zone on this TLD. A company
called "Example TLS Certificates, LTD." could register the zone `example.pki`
for example.

The TLD should be DNSSEC secured as I believe is already required for any new
Top Level Domains.

It is my opinion that the RR types used in the CA zones should be limited to a
subset of the current RR types. This TLD is not intended for running web sites,
it is intended for the needs of the Public Key Infrastructure.

For zone files used by the Certificate Authorities I would limit the RR types
to SOA, NS, RP, TXT, and the RR types needed for DNSSEC.

For proving a record does not exist, the NSEC RR Type should be allowed, I do
not believe zone walking is a security concern for this utility.

RR types like A or AAAA have no purpose in this TLD, unless *maybe* if used in
conjunction with OCSP to resolve domains used with OCSP, but I would suggest
that not be allowed.

The TLD itself will obviously need other RR types, such as the A and AAAA
records needed to create glue records for the Certificate Authority
nameservers.


### Nameservers

The actual nameservers used by the Certificate Authorities to serve the zone
information can exist on any TLD.

To help mitigate Denial of Service attacks I would suggest that each
Certificate Authority use a minimum of five nameservers on a minimum of three
different geographic continents.

The nameservers should operate on both IPv4 and IPv6 with appropriate glue
records.

I would suggest that the TLD authority set up nameservers around the globe that
can act as slaves for the Certificate Authorities masters so that each
Certificate Authority does not have the individual cost of running their own
slaves around the globe.


### DNSSEC Signing Keys

The keys used to sign the zones should not be on a host that has a public IP
address.

To avoid overly large responses, Certificate Authorities should be required to
follow the best practices of using a 1024-bit Zone Signing Key (ZSK) that is
rotated frequently and a stronger 2048-bit Key Signing Key (KSK) that is
rotated less often.

I would require that they rotate the ZSK at least twice a month and suggest
that they rotate it on a weekly basis.

I would require that they rotate the KSK twice a year and suggest that they
rotate it any time an employee who potentially had access to the signing key
leaves the company for any reason.


### SOA Record Serial Number

It is generally considered best practices to use the `YYYYMMDDnn` format for
the serial number in a zone's SOA record. That format only allows a zone to
updated every 15 minutes which may not be enough for this purpose.

I would suggest that Certificate Authorities instead use seconds since UNIX
Epoch for the serial number. The SOA serial number is an unsigned 32-bit
integer and will accommodate seconds from epoch until we are into the 22nd
century, at which point if the 32-bit limitation on zone serial numbers still
exists, the SOA specification allows it wrap back to 0.

Certificate Authorities should be aware that if they are using 32-bit systems
to calculate the UNIX seconds since epoch, UNIX systems use a signed int for
that number and on 32 bit systems it will wrap in 2038. 64-bit systems do not
have this issue but as we approach 2038, certificate authorities should make
sure the software they are using to generate the SOA serial number is doing so
in 64-bit and not 32-bit.


## X.509 Certificates

Certificates issued that use this system should include the following
information in the certificate itself:

1. OCSP validation URI (to be used as a fallback)
2. Serial Number that is not longer than 63 characters and follows the existing
   rules that exist for a valid label in a DNS domain name.
3. The label of the validation domain the certificate authority has registered
   on the .pki TLD for the purpose of validating that a certificate has or has
   not been revoked.
   
OCSP should continue to be available, this system is not intended to replace
OCSP and OCSP is still needed as a fall-back mechanism that clients can use in
the event that they can not get a positive or negative result from the PKI
Validation nameserver.


## Validation TXT Records

The owner of a validation TXT record should be the serial number of the X.509
certificate as described above.

I would suggest the TTL for the TXT records be set to 3600 seconds (one hour).
That will allow resolvers to cache the result long enough to reduce the network
load on the PKI Validation nameservers for frequently requested serial numbers.

The class for the TXT record should be set to IN (Internet Class).

The text RDATA should be of the following format:

    "v=PKIV1; s=n; ts=YYYY-MM-DDTHH:MM:SSZ;"
    
where `n` is 0 if the certificate has been revoked, 1 if the certificate has
not been revoked and where the `ts` value corresponds with the UTC
[W3C Date and Time Format](http://www.w3.org/TR/NOTE-datetime) and indicates
the last time the certificate was known to be valid by the Certificate
Authority.

I would suggest that the Certificate Authority update the `ts` value every time
the zone is updated unless the certificate has been revoked. If the certificate
has been revoked and the `s` value is set to 0, then the `ts` value does not
matter.


### Updating the Zone

The certificate authority should update the zone and push the update to the
slaves every five to ten minutes regardless of whether or not a new certificate
has been issued or an old certificate has been revoked.


## Validation Process

This is the flow of how TLS clients should validate certificates using the PKI
Validation Nameservers.

The client requests a TXT record from the PKI Validation zone using the serial
number of the certificate and the label of the PKI Validation zone that the
client extracted from the signed certificate.

For example, if the certificate has a serial number `42H67QWP` and specifies a
PKI label of `example` then the TLS client would query the DNS system for a
`TXT` record associated with the domain `42h67qwp.example.pki.`


### Step One - Serial Number and CA Validation server.

Does the certificate contain a serial number and CA validation label?

If yes, goto Step Two. If no, goto Fallback.


### Step Two - Request TXT record from the DNS System.

Does the DNS system return a DNSSEC validating response?

If yes, goto Step Three. If no, goto Fallback.


### Step Three - Does DNS Record Exist?

If yes, goto Step Four. If no, goto Fallback.


### Step Four - Is RDATA of the Valid Format?

If yes, goto Step Five. If no, goto Fallback


### Step Five - Is the Status key `s` set to 1?

If yes, goto Step Six. If no, HARD FAIL CERTIFICATE REJECTED.


### Step Six - Is Timestamp within last two hours?

If yes, ACCEPT CERTIFICATE. If no, goto Fallback


### Fallback

Attempt to use the OCSP URI in the certificate to validate the certificate. If
the client can not get a response from the OCSP server and the certificate is
an EV certificate, the client should HARD FAIL REJECT CERTIFICATE. If the client
can not get a response from the OCSP server and the certificate is a DV or OV
certificate, it should be a client preference what to do.

EV certificates should always HARD FAIL REJECT CERTIFICATE when they can not be
validated by either the PKI Validation nameserver or OCSP due to the level of
trust that users are told they can have in web sites that use EV certificates.
If they can not be validated then they do not warrant that level of trust and
the end user should be protected from a potentially revoked EV certificate in
fraudulent MITM use.


## Why A New TLD Is Needed

From a technical perspective, in a perfect world this system could work
without needing a separate TLD. The real world is not perfect, a new TLD allows
at least two potential problems to be solved.


### Certificate Authority Accountability

This system should not be used as a replacement for OCSP as described in
[RFC 6960](http://tools.ietf.org/html/rfc6960). Certificate Authorities who do
not provide a mechanism for OCSP validation or Certificate Authorities who fail
to update the status key `s` when a certificate is revoked should have their
ability to use PKI Validation zones revoked.

A TLD specific to the PKI Validation nameservers allows for proper management
of which Certificate Authorities are allowed to use the system.


### Resolvers Blocking TXT and/or DNSSEC Records

Some users are behind resolvers that block requests to some types of DNS RR
requests. I do not know why some resolvers choose to do this but I suspect it
is a an attempt to reduce the use of their resolver in a distributed Denial of
Service DNS amplification attack.

By using a distinct TLD for PKI Validation zones, some of these resolvers
could allow TXT and DNSSEC requests for the distinct TLD while still choosing
to block those requests on other TLDs. Of course they would also need to allow
DNSSEC related requests to the root nameservers as well.

If resolvers blocking DNS requests continues to be a problem, browsers could
potentially even tunnel DNS requests for the .pki TLD over port 80 to a DNS
server coded in the browser. Bypassing the clients specified resolver is not
something browsers should do for most hostnames but it would be safe to do for
a utility TLD *when* the configured client resolver blocks the resources
needed.


## Certificate Authority Participation

Certificate Authorities that issue EV certificates should be required to fully
participate will all certificates they issue, EV or not.

I would recommend that web browser vendors require Certificate Authority
participation in order to be included in the list of Certificate Authorities
they trust by default, but browser vendors should allow the end user to add
custom Certificate Authorities that do not participate.


## Security Considerations

If an attacker manages to steal the DNSSEC signing keys, then the attacker
could potentially create false records that would DNSSEC validate and pull off
a DNS cache poisoning attack.

This could result in DoS attack on domains that have valid a certificate or it
could result in a revoked fraudulent certificate being accepted by a client.

This threat can be mitigated by using an appropriate DNSSEC key rollover
schedule and proper security procedures when handling the private DNSSEC
signing keys.

If an attacker manages to pull off a Denial of Service attack against the
authoritative nameservers for a PKI Validation zone, then clients may not be
able to use this system to establish the current status of a certificate.

This can be mitigated by using a sufficient number of authoritative slaves
that are geographically spread out, and clients will still be able to use the
OCSP protocol as a fallback.


## Dedication

I would like to dedicate this idea to Peggy Karp, the author of
[RFC 226](http://www.rfc-archive.org/getrfc.php?rfc=226) and inventor of the
`HOSTS.TXT` file that first provided a standardized lookup table of host names
to network addresses.

She usually is only mentioned in a footnote if at all in articles about the
history of DNS and the Internet, but her invention is as fundamental to the
Internet as the invention of the wheel is to mechanical technology.