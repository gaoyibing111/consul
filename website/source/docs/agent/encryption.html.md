---
layout: "docs"
page_title: "Encryption"
sidebar_current: "docs-agent-encryption"
description: |-
  The Consul agent supports encrypting all of its network traffic. The exact method of encryption is described on the encryption internals page. There are two separate encryption systems, one for gossip traffic and one for RPC.
---

# Encryption

The Consul agent supports encrypting all of its network traffic. The exact
method of encryption is described on the [encryption internals page](/docs/internals/security.html).
There are two separate encryption systems, one for gossip traffic and one for RPC.

## Gossip Encryption

Enabling gossip encryption only requires that you set an encryption key when
starting the Consul agent. The key can be set via the `encrypt` parameter: the
value of this setting is a configuration file containing the encryption key.

The key must be 16-bytes, Base64 encoded. As a convenience, Consul provides the
[`consul keygen`](/docs/commands/keygen.html) command to generate a
cryptographically suitable key:

```text
$ consul keygen
cg8StVXbQJ0gPvMd9o7yrg==
```

With that key, you can enable encryption on the agent. If encryption is enabled,
the output of [`consul agent`](/docs/commands/agent.html) will include "Encrypted: true":

```text
$ cat encrypt.json
{"encrypt": "cg8StVXbQJ0gPvMd9o7yrg=="}

$ consul agent -data-dir=/tmp/consul -config-file=encrypt.json
==> WARNING: LAN keyring exists but -encrypt given, using keyring
==> WARNING: WAN keyring exists but -encrypt given, using keyring
==> Starting Consul agent...
==> Starting Consul agent RPC...
==> Consul agent running!
         Node name: 'Armons-MacBook-Air.local'
        Datacenter: 'dc1'
            Server: false (bootstrap: false)
       Client Addr: 127.0.0.1 (HTTP: 8500, HTTPS: -1, DNS: 8600, RPC: 8400)
      Cluster Addr: 10.1.10.12 (LAN: 8301, WAN: 8302)
    Gossip encrypt: true, RPC-TLS: false, TLS-Incoming: false
...
```

All nodes within a Consul cluster must share the same encryption key in
order to send and receive cluster information.

## Configuring Gossip Encryption on an existing cluster

As of version 0.8.4, Consul supports upshifting to encrypted gossip on a running cluster
through the following process.

1. Generate an encryption key using [`consul keygen`](/docs/commands/keygen.html)
2. Set the [`encrypt`](/docs/agent/options.html#_encrypt) key in the agent configuration and set
[`encrypt_verify_incoming`](/docs/agent/options.html#encrypt_verify_incoming) and
[`encrypt_verify_outgoing`](/docs/agent/options.html#encrypt_verify_outgoing) to `false`, doing a
rolling update of the cluster with these new values. After this step, the agents will be able to
decrypt gossip but will not yet be sending encrypted traffic.
3. Remove the [`encrypt_verify_outgoing`](/docs/agent/options.html#encrypt_verify_outgoing) setting
to change it back to false (the default) and perform another rolling update of the cluster. The
agents will now be sending encrypted gossip but will still allow incoming unencrypted traffic.
4. Remove the [`encrypt_verify_incoming`](/docs/agent/options.html#encrypt_verify_incoming) setting
to change it back to false (the default) and perform a final rolling update of the cluster. All the
agents will now be strictly enforcing encrypted gossip.

## RPC Encryption with TLS

Consul supports using TLS to verify the authenticity of servers and clients. To enable this,
Consul requires that all clients and servers have key pairs that are generated by a single
Certificate Authority. This can be a private CA, used only internally. The
CA then signs keys for each of the agents, as in
[this tutorial on generating both a CA and signing keys](http://russellsimpkins.blogspot.com/2015/10/consul-adding-tls-using-self-signed.html)
using OpenSSL. 

-> **Note:** Client certificates must have [Extended Key Usage](https://www.openssl.org/docs/manmaster/man5/x509v3_config.html#Extended-Key-Usage) enabled for client and server authentication.

TLS can be used to verify the authenticity of the servers or verify the authenticity of clients.
These modes are controlled by the [`verify_outgoing`](/docs/agent/options.html#verify_outgoing),
[`verify_server_hostname`](/docs/agent/options.html#verify_server_hostname),
and [`verify_incoming`](/docs/agent/options.html#verify_incoming) options, respectively.

If [`verify_outgoing`](/docs/agent/options.html#verify_outgoing) is set, agents verify the
authenticity of Consul for outgoing connections. Server nodes must present a certificate signed
by a common certificate authority present on all agents, set via the agent's
[`ca_file`](/docs/agent/options.html#ca_file) and [`ca_path`](/docs/agent/options.html#ca_path)
options. All server nodes must have an appropriate key pair set using [`cert_file`]
(/docs/agent/options.html#cert_file) and [`key_file`](/docs/agent/options.html#key_file).

If [`verify_server_hostname`](/docs/agent/options.html#verify_server_hostname) is set, then
outgoing connections perform hostname verification. All servers must have a certificate
valid for `server.<datacenter>.<domain>` or the client will reject the handshake. This is
a new configuration as of 0.5.1, and it is used to prevent a compromised client from being
able to restart in server mode and perform a MITM (Man-In-The-Middle) attack. New deployments should set this
to true, and generate the proper certificates, but this is defaulted to false to avoid breaking
existing deployments.

If [`verify_incoming`](/docs/agent/options.html#verify_incoming) is set, the servers verify the
authenticity of all incoming connections. All clients must have a valid key pair set using
[`cert_file`](/docs/agent/options.html#cert_file) and
[`key_file`](/docs/agent/options.html#key_file). Servers will
also disallow any non-TLS connections. To force clients to use TLS,
[`verify_outgoing`](/docs/agent/options.html#verify_outgoing) must also be set.

TLS is used to secure the RPC calls between agents, but gossip between nodes is done over UDP
and is secured using a symmetric key. See above for enabling gossip encryption.

## Configuring TLS on an existing cluster

As of version 0.8.3, Consul supports migrating to TLS-encrypted traffic on a running cluster
without downtime. This process assumes a starting point with no TLS settings configured, and involves
an intermediate step in order to get to full TLS encryption:

1. Generate the necessary keys/certs and set the `ca_file`/`ca_path`, `cert_file`, and `key_file`
settings in the configuration for each agent. Make sure the `verify_outgoing` and `verify_incoming`
options are set to `false`. HTTPS for the API can be enabled at this point by
setting the [`https`](/docs/agent/options.html#http_port) port.
2. Perform a rolling restart of each agent in the cluster. After this step, TLS should be enabled
everywhere but the agents will not yet be enforcing TLS.
3. Change the `verify_incoming` and `verify_outgoing` settings (as well as `verify_server_hostname`
if applicable) to `true`.
4. Perform another rolling restart of each agent in the cluster.

At this point, full TLS encryption for RPC communication should be enabled.