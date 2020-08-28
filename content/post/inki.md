---
title: Inki
date: 2016-12-15T16:53:47.000+00:00
tags:
- opensource
- github
- golang
- security
categories:
- Projects
draft: true

---
[Inki][] is a small proof of concept project I've been working on which is
designed to manage transient, single-use, SSH keys for an automated remediation
tool our team is in the process of building.

In this blog post I'll go over some of the design decisions motivating
a tool like [Inki][], some of its interesting implementation details and
the questions we're hoping it will allow us to answer.

<!--more-->

## What is Inki?
[Inki][] is, at its core, a small HTTP application server which provides an
API through which SSH keys can be registered for a specific user and
timeframe. Those keys are then intended to be pulled onto a server
during the SSH connection handshake through the use of the `AuthorizedKeysCommand`
option in `sshd_config`.

This approach enables us to, in realtime, add and remove keys from servers
safely - even when the host is in a state which prevents it from writing
to disk.

It is important to note that Inki itself is primarily a research project,
designed to give us a platform on which to answer questions rather than
a production-ready tool.

{% alert warning %}
Inki isn't a production ready tool, and may never be, keep your servers
safe and audit the code yourself before deploying it.
{% endalert %}

## Why is Inki required?
Automation, especially infrastructure level, can be a risky business, just
reading through Google's (brilliant) SRE Manual will yield a number of interesting
stories on how automation has caused some pretty major problems.

As a result, it's important to protect your systems not just from bad actors,
but against rogue automation as well. To this end, you want to ensure that only
automation operations that have been explicitly allowed can make changes to the
hosts in your datacenters.

In one of our proposed security models, hosts would have a final veto vote on
whether remediation is performed on them - this would be decided based on
the presence of failing healthchecks, so a host with no failing healthchecks
wouldn't allow any remediation tools to access it.

This raises an interesting question though, what happens if we've allowed one
remediation to proceed and another attempts to access the host? In an ideal
world, the second remediation operation would be denied access. To support that
use case we need each remediation operation to have its own unique SSH key,
which means we need to automate the propagation of those keys to hosts in need
of remediation.

This is exactly the problem Inki is intended to solve.

## What were Inki's requirements?
Inki, being primarily intended as a tool for remediation, had a couple of features
which we placed particular emphasis on from the start.

1. **Reliability**, as our remediation would depend on Inki being available we had
   to be able to guarantee that it would keep running at all times.

1. **Simplicity** so that we could manually review the code-base to confirm that there
   were no unhandled failure conditions.

1. **Security** as it would be (in theory) granting access to servers on our infrastructure.
   This meant preventing bad actors from being able to use the system to gain access
   to our infrastructure.

1. **Auditability** to allow us to keep track of every operation that was taken for
   security and failure review.

1. **Compatibility** with existing tools, allowing anybody to interact with it easily
   and without needing to hunt down special scripts or applications. If things go wrong,
   you want your SREs to be able to solve the problem as quickly as possible, not present
   them with more problems.

## How we addressed those requirements

### Reliability
Reliability, especially in the realm of software development, is an interesting problem
to solve. In our case, we opted to use a language which places an emphasis on dealing
with issues where they happen - [Go][golang].

By dealing with errors immediately, we avoid the common pitfall of exceptions, which 
is that your application can be left in an inconsistent state. It also forces us to
consider how best to handle a failure case at every step - something which greatly
increases awareness of failure modes.

[Go][golang] also has a brilliant set of core libraries which are well tested, exceptionally
stable and very easy to use correctly. All this combines to make it a great language
to develop highly resilient applications on.

### Simplicity
Going hand-in-hand with the reliability requirement, keeping things as simple as possible
ensures that we can manually audit the code for security risks and possible failure
conditions far more easily - reducing the time it takes to deploy it to production and
minimizing the cost of changes down the line.

In addition to that, it means the memory cost of running Inki is miniscule, its CPU
load is essentially nonexistent and startup times are kept reasonable.

### Security
When you're building a service which will grant access to your servers, ensuring it only
grants access to the right people is of the upmost importance. With that in mind, it was
critical that we have a strong chain of trust for all operations.

To ensure this, we're using asymmetric cryptography to sign all requests, ensuring that
they come from an authorized source and that they have not been tampered with. To keep
things accessible, we're using `pgp` for this and passing the `gpg --clearsign` output
straight to Inki's server, allowing you to interact using nothing but `gpg` and `curl`.

### Auditability
Auditing of services like this is paramount to ensuring that they are maintainable in
a production environment as well as for catching breaches early. To this end, Inki logs
every operation it conducts in a structured format through the brilliant [logrus][] package.

By logging every operation, as well as the data associated with it, we are able to analyze
patterns and review events through our ELK stack. It also provides our security team with
an easy source of data should they ever need it.

### Compatibility
The final aspect, ensuring that Inki was compatible with existing tools and didn't get in
the way, was primarily an issue of API design. By focussing on using standards compliant
input and output formats, no weird custom headers and implementing CORS support on the server,
it ensured that things were trivially easy to use from any HTTP client.

As it happened, initial tests were performed using [Keybase][keybase] and `curl` to sign
and send the data respectively. By keeping things this simple, we ensure that any of our
SREs can, with just the tools on their laptop, access and interact with Inki at a moment's
notice.

## Inki's trust model
Inki assumes that its server will be running on every host in your datacenter. We start with
the assumption that the local host has not been compromised and that we wish to prevent it from
being compromised as a result of Inki running there. To defend against an external attacker we
are required to ensure the following:

 - An external attacker cannot influence Inki's key list
 - An external attacker cannot influence the keys appearing to originate from Inki
 - An external attacker cannot influence the list of authorities that Inki trusts

### Preventing MITM Attacks
By running Inki on the local server and using `127.0.0.1` as the server address for all authorized_key
queries, it becomes impossible for an external attacker to perform a MITM attack against the
local Inki client and server.

### Preventing Impersonation
Protecting Inki's key list from tampering then becomes a matter of authenticating the requesting
entity. This authentication needs to be handled in a way that doesn't introduce additional security
risks and preferably doesn't increase complexity. Ideally, another compromised server shouldn't give
an attacker any additional tools with which to compromise another. This either means that we must use
a secure, external, authentication service which we trust or that we need to make use of asymmetric
cryptography to prevent a compromised machine from revealing authentication information.

At this point the astute among you will be asking why we didn't settle for something like `scrypt`
to secure the credentials used. Remember that we're attempting to protect against information disclosure
on a compromised node. Such a compromised node could silently record the data it receives using `pcap`
and ship that to an attacker, giving them access to the credentials in cleartext (or encrypted using
a certificate that exists on the compromised node anyway).

To avoid this risk, we adopt asymmetric cryptography to sign all requests using a private key which
is stored on a secure server. The corresponding public key doesn't disclose any useful information to
an attacker and as a result, a compromised node's data doesn't make it easier for an attacker to compromise
another.

### Preventing Configuration Modifications
The final piece of the puzzle is protecting Inki's configuration, in which the trusted public key is
specified, from an external attacker. If we trust our configuration management system (Ansible/Puppet/SaltStack)
to be secure, or that it being compromised would render any potential benefits of compromising Inki moot,
then we can assume that our configuration will not be compromised by an external attacker.

### Potential Risks
Even with this trust model, there is still the possibility that an attacker manages to guess or derive
a suitable private key with which they can sign requests to Inki nodes. By using 4096-bit RSA keys we
assume that the workload of deriving a suitable key is prohibitively large, while the probability of
a key accidentally matching should be small enough that it can be assumed to be non-existent.

Another risk is in the engineering of a payload which successfully matches an existing hash. Known as
a hash collision, this would likely represent the most plausible avenue of attack, however we are making
use of SHA-512 to mitigate this risk and there are currently no known weaknesses in this hash implementation.

Realistically, the biggest risk we could face with Inki is that a machine with the private key becomes
compromised, in which case we would be required to perform a key-rotation by pushing new trusted public keys
to every node (a process managed by our configuration management system) and adopting a new private key.

## Interacting with Inki
Inki's HTTP API has been designed to make interacting with it as straightforward as possible, enabling an
SRE to use just their web browser and `curl` to perform most operations. That being said, it's always
easier to use a purpose built tool to accomplish things. What follows are a few commands using the `inki`
command line tool and `curl` performing the same tasks.

### Get a user's authorized keys
```sh
# Get the list of keys for ~bpannell in a format compatible with the authorized_keys file
inki key list http://bpannell@127.0.0.1:3000 --authorized-keys

# ...and do the same using curl
curl http://127.0.0.1:3000/api/v1/user/bpannell/authorized_keys
```

### Add a key for a specific user
```sh
# Add the id_rsa.pub key to ~bpannell on the server "inki" using inki.gpg to sign
# the request, telling the key to expire after 1 hour
inki key add http://bpannell@inki:3000 -f id_rsa.pub -p inki.gpg -x 1h

# ...and do someting similar using curl
cat <<JSON
{
    "username": "bpannell",
    "expire": "2016-12-25T00:00:00Z",
    "key": "$(cat id_rsa.pub)"
}
JSON | gpg --clearsign -u inki | curl -X POST http://inki:3000/api/v1/keys
```

## What's Next?
Well, as I mentioned at the start, Inki is a research project intended to get us asking the right questions.
Specifically, we want to look at where it works well and where it doesn't, what possible failure modes exist
which are outside the control of Inki's process and how we'd deal with them, what possible security
flaws still exist and what can be done to counter those.

As it turns out, we may even end up in a situation where Inki's core requirement (the ability to enable
remote processes to perform remediation against a host) isn't necessary.

Suffice to say, if we do decide to adopt Inki, we'll spend time putting in place automated tests and cleaning
up the code (it was thrown together in an afternoon).

[Inki]: https://github.com/SierraSoftworks/inki
[golang]: https://golang.org
[logrus]: https://github.com/Sirupsen/logrus
[keybase]: https://keybase.io