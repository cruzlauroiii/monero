# Anonymity Networks with Monero

Currently only Tor and I2P have been integrated into Monero. The usage of
these networks is still considered experimental - there are a few pessimistic
cases where privacy is leaked. The design is intended to maximize privacy of
the source of a transaction by broadcasting it over an anonymity network, while
relying on IPv4 for the remainder of messages to make surrounding node attacks
(via sybil) more difficult.


## Behavior

If _any_ anonymity network is enabled, transactions being broadcast that lack
a valid "context" (i.e. the transaction did not come from a P2P connection),
will only be sent to peers on anonymity networks. If an anonymity network is
enabled but no peers over an anonymity network are available, an error is
logged and the transaction is kept for future broadcasting over an anonymity
network. The transaction will not be broadcast unless an anonymity connection
is made or until `monerod` is shutdown and restarted with only public
connections enabled.

Anonymity networks can also be used with `monero-wallet-cli` and
`monero-wallet-rpc` - the wallets will connect to a daemon through a proxy. The
daemon must provide a hidden service for the RPC itself, which is separate from
the hidden service for P2P connections.


## P2P Commands

Only handshakes, peer timed syncs and transaction broadcast messages are
supported over anonymity networks. If one `--add-exclusive-node` P2P address
is specified, then no syncing will take place and only transaction broadcasting
can occur. It is therefore recommended that `--add-exclusive-node` be combined
with additional exclusive IPv4 address(es).


## Usage

### Blockchain sync

Monerod does not support synchronizing the blockchain over onion or I2P hidden services.
You may sync the blockchain using a socks proxy.

```bash
monerod --proxy 127.0.0.1:9050 --p2p-bind-ip 127.0.0.1 --no-igd
```

### Hidden Services

Hidden services - onion and I2P domains - are available to use for transation broadcasts.
You may use the below options with or without `--proxy`.

#### Outbound Connections

Connecting to an anonymous address requires the command line option
`--tx-proxy` which tells `monerod` the ip/port of a socks proxy provided by a
separate process. On most systems the configuration will look like:

```bash
monerod \
    --tx-proxy tor,127.0.0.1:9050,10 \
    --tx-proxy i2p,127.0.0.1:4447
```

which tells `monerod` to connect to ".onion" P2P addresses using a socks
proxy at IP 127.0.0.1 port 9050 with a max of 10 outgoing connections and
".b32.i2p" P2P addresses using a socks proxy at IP 127.0.0.1 port 4447
with the default max outgoing connections.

If desired, peers can be manually specified:

```bash
--add-exclusive-node 5tymba6faziy36md5ffy42vatbjzlye4vyr3gyz6lcvdfximnvwpmwqd.onion:18084
--add-priority-node 5tymba6faziy36md5ffy42vatbjzlye4vyr3gyz6lcvdfximnvwpmwqd.onion:18084
--add-peer 5tymba6faziy36md5ffy42vatbjzlye4vyr3gyz6lcvdfximnvwpmwqd.onion:18084
```

Either option can be listed multiple times, and can specify any mix of Tor,
I2P, and IPv4 addresses. Using `--add-exclusive-node` will prevent the usage of
seed nodes on ALL networks, which will typically be undesirable.  
If you specify `add-exclusive-node` for onion or I2P, make sure to do so for clearnet nodes as well, otherwise you will be unable to sync.

#### Inbound Connections

Receiving anonymity connections is done through the option
`--anonymous-inbound`. This option tells `monerod` the inbound address, network
type, and max connections:

```bash
--anonymous-inbound 5tymba6faziy36md5ffy42vatbjzlye4vyr3gyz6lcvdfximnvwpmwqd.onion:18084,127.0.0.1:18084,25 \
--anonymous-inbound cmeua5767mz2q5jsaelk2rxhf67agrwuetaso5dzbenyzwlbkg2q.b32.i2p,127.0.0.1:18085
```

which tells `monerod` that a max of 25 inbound Tor connections are being
received at address "5tymba6faziy36md5ffy42vatbjzlye4vyr3gyz6lcvdfximnvwpmwqd.onion:18084" and forwarded to `monerod`
localhost port 18084, and a default max I2P connections are being received at
address "cmeua5767mz2q5jsaelk2rxhf67agrwuetaso5dzbenyzwlbkg2q.b32.i2p" and
forwarded to `monerod` localhost port 18085. Using `tx-proxy`(required), these
addresses will be shared with peers over the same network type, otherwise your
peers will not be notified of your onion or I2P address.

**_Note: The specified port for `anonymous-inbound` must be unique (not 18080 etc). `anonymous-inbound` is not for blockchain sync!_**
Peers will use their own `tx-proxy` to relay transactions, which originate on their node,
to your `anonymous-inbound`.

#### Wallet RPC

An anonymity network can be configured to forward incoming connections to a
`monerod` RPC port - which is independent from the configuration for incoming
P2P anonymity connections. The anonymity network (Tor/I2P) is
configured in the same manner as [below](#configuration), except this excludes P2P.

```text
HiddenServiceDir /var/lib/tor/data/monero-rpc
HiddenServicePort 18089 127.0.0.1:18089
```

Then the wallet will be configured to use a Tor/I2P address:
```bash
monero-wallet-cli \
    --proxy 127.0.0.1:9050 \
    --daemon-address 5tymba6faziy36md5ffy42vatbjzlye4vyr3gyz6lcvdfximnvwpmwqd.onion:18089
```

The proxy must match the address type - a Tor proxy will not work properly with
I2P addresses, etc.

I2P hidden service (b32.i2p) and Tor Hidden service (.onion) addresses provide the information necessary to authenticate and
encrypt the connection from end-to-end. If desired, SSL can also be applied to
the connection with `--daemon-address https://5tymba6faziy36md5ffy42vatbjzlye4vyr3gyz6lcvdfximnvwpmwqd.onion` which
requires a server certificate that is signed by a "root" certificate on the
machine running the wallet. Alternatively, `--daemon-ssl-certificate` can be used to
specify a certificate to authenticate the server.

Proxies can also be used to connect to "clearnet" (IPv4 addresses or ICANN
domains), but `--daemon-ssl-certificate` _must_ be used for authentication and
encryption, or bypassed with `--daemon-ssl-allow-any-cert`.

### Network Types

#### Tor & I2P

Options `--add-exclusive-node`, `--add-priority-node`, and `--add-peer` recognize ".onion" and
".b32.i2p" addresses, and will properly forward those addresses to the proxy
provided with `--tx-proxy tor,...` or `--tx-proxy i2p,...`.

Option `--anonymous-inbound` also recognizes ".onion" and ".b32.i2p" addresses,
and will automatically be sent out to outgoing Tor/I2P connections so the peer
can distribute the address to its other peers.

#### Configuration

Tor must be configured for hidden services. An example configuration ("torrc")
might look like:

```text
# P2P Hidden service
HiddenServiceDir /var/lib/tor/data/monero
HiddenServicePort 18084 127.0.0.1:18084  # anonymous-inbound

# RPC Hidden service
HiddenServiceDir /var/lib/tor/data/monero-rpc
HiddenServicePort 18089 127.0.0.1:18089  # rpc-restricted-bind-port
```

This will store key information in `/var/lib/tor/data/monero` and `/var/lib/tor/data/monero-rpc`
and will forward "Tor port" 18084 and 18089 to ports 18084 and 18089 of ip 127.0.0.1, respectively. The file
`/usr/lib/tor/data/monero/hostname` will contain the ".onion" address for use with `--anonymous-inbound`, and
`/var/lib/tor/data/monero-rpc/hostname` will contain the ".onion" address for use with RPC.

I2P must be configured with a standard server tunnel. Configuration differs by
I2P implementation.  
You can find guides for i2pd [here](https://docs.getmonero.org/running-node/monerod-tori2p/#__tabbed_1_2).

## Privacy Limitations

There are currently some techniques that could be used to _possibly_ identify
the machine that broadcast a transaction over an anonymity network.

### Timestamps

The peer timed sync command sends the current time in the message. This value
can be used to link an onion address to an IPv4/IPv6 address. If a peer first
sees a transaction over Tor, it could _assume_ (possibly incorrectly) that the
transaction originated from the peer. If both the Tor connection and an
IPv4/IPv6 connection have timestamps that are approximately close in value they
could be used to link the two connections. This is less likely to happen if the
system clock is fairly accurate - many peers on the Monero network should have
similar timestamps.

#### Mitigation

Keep the system clock accurate so that fingerprinting is more difficult. In
the future a random offset might be applied to anonymity networks so that if
the system clock is noticeably off (and therefore more fingerprintable),
linking the public IPv4/IPv6 connections with the anonymity networks will be
more difficult.

### Intermittent Monero Syncing

If a user only runs `monerod` to send a transaction then quit, this can also
be used by an ISP to link a user to a transaction.

#### Mitigation

Run `monerod` as often as possible to conceal when transactions are being sent.
Future versions will also have peers that first receive a transaction over an
anonymity network delay the broadcast to public peers by a randomized amount.
This will not completely mitigate a user who syncs up sends then quits, in
part because this rule is not enforceable, so this mitigation strategy is
simply a best effort attempt.

### Active Bandwidth Shaping

An attacker could attempt to bandwidth shape traffic in an attempt to determine
the source of a Tor/I2P connection. There isn't great mitigation against
this, but I2P should provide better protection against this attack since
the connections are not circuit based.

#### Mitigation

The best mitigation is to use I2P instead of Tor. However, I2P
has a smaller set of users (less cover traffic) and academic reviews, so there
is a trade off in potential issues. Also, anyone attempting this strategy really
wants to uncover a user, it seems unlikely that this would be performed against
every Tor/I2P user.

### I2P/Tor Stream Used Twice

If a single I2P/Tor stream is used 2+ times for transmitting a transaction, the
operator of the hidden service can conclude that both transactions came from the
same source. If the subsequent transactions spend a change output from the
earlier transactions, this will also reveal the "real" spend in the ring
signature. This issue was (primarily) raised by @secparam on [Twitter](https://twitter.com/secparam/status/1153411657214910469).

#### Mitigation

`monerod` currently selects two outgoing connections every 5 minutes for
transmitting transactions over I2P/Tor. Using outgoing connections prevents an
adversary from making many incoming connections to obtain information (this
technique was taken from Dandelion). Outgoing connections also do not have a
persistent public key identity - the creation of a new circuit will generate
a new public key identity. The lock time on a change address is ~20 minutes, so
`monerod` will have rotated its selected outgoing connections several times in
most cases. However, the number of outgoing connections is typically a small
fixed number, so there is a decent probability of re-use with the same public
key identity.

@secparam ([Twitter](https://twitter.com/secparam/status/1153411968147042304)) recommended changing circuits (Tor) as an additional
precaution. This is likely not a good idea - forcibly requesting Tor to change
circuits is observable by the ISP. Instead, `monerod` should likely disconnect
from peers occasionally. Tor will rotate circuits every ~10 minutes, so
establishing new connections will use a new public key identity and make it
more difficult for the hidden service to link information. This process will
have to be done carefully because closing/reconnecting connections can also
leak information to hidden services if done improperly.

At the current time, if users need to frequently make transactions, I2P/Tor
will improve privacy from ISPs and other common adversaries, but still have
some metadata leakages to unknown hidden service operators.
