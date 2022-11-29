# NM requirements for prefix delegation

## Part 1: configuration of DHCPv6 on the upstream interface

The flexibility of the IPv6 `auto` method is currently quite limited,
as it only allows to perform configuration strictly according to the
RFCs. What `auto` does is:

 - if the RA has the 'M' flag, DHCPv6 is started in solicit mode
 - if the RA has the 'O' flag, DHCPv6 is started in info-request mode
 - if a prefix is needed for a downstream interface, then DHCPv6 is
   restarted (or started if not already running). The new request
   includes a IA_PD option to request one or more prefixes
 - if no address, otherconf or prefix are needed, DHCPv6 is not started.

However there are other valid use cases not currently supported, such
as:

 1. when the RA doesn't have 'M' or 'O' flags:
 
    then start DHCPv6 in solicit mode to get an additional address

 2. when the RA doesn't have 'M' or 'O' flags:
 
    then start DHCPv6 in info-request mode to get a DNS server

 3. when there aren't any downstream interfaces in shared mode:
 
    then start DHCPv6 to get a prefix so that an external entity can
    lookup the prefix via D-Bus or via the lease file and do something
    with it

 4. when the RA has the 'M' flag or the 'O' flag:
 
    then don't start DHCPv6. This was requested in the past by layered
    products because in that scenario the DHCPv6 server was not under
    their control

 5. when the RA has the 'M' flag:
 
    then start DHCPv6 only to get a prefix not an address. There are
    ISPs requiring this:
    https://gitlab.freedesktop.org/NetworkManager/NetworkManager/-/issues/1121#note_1605127

 6. when there are downstream interfaces in shared mode:
 
    then start DHCPv6 to get a prefix of a given length, or suggest
    what prefix is preferred (in other words, support sending a hint)

### New properties

The use cases above can be supported using the following new
properties (value marked with `*` are the default):

 - `ipv6.dhcp = {auto*,solicit,info,no}`

   - *auto* = the current behavior
   - *solicit* = always start DHCPv6, in solicit mode
   - *info* = always start DHCPv6, in info-request mode
   - *no* = never start DHCPv6, even if 'M' or 'O' are set or if prefixes are needed

 - `ipv6.dhcp-request-prefix = {auto*,yes,no}`

   - *auto* = only request a prefix when needed by a downstream interface
   - *yes* = always request a prefix when DHCPv6 is started
   - *no* = never request a prefix when DHCPv6 is started

 - `ipv6.dhcp-prefix-hint = ADDRESS/PREFIXLEN`

   - Indicates the address and prefix to send in the request to the
     server. The address can be '::' in case we only want to request a
     prefix len. The prefix len is required.

For now if the user sets `ipv6.dhcp=solicit` or `info`, we assume
that DHCPv6 is required and thus the activation will complete only
after NM gets a reply from the server. In the future, we could make
this optional through another property `ipv6.dhcp-required=BOOL`.

### Example configurations for the use cases above

 - when the RA doesn't have 'M' or 'O' flags
 
   then start DHCPv6 in solicit mode to get an additional address
   
```
   ipv6.dhcp=solicit
```

 - when the RA doesn't have 'M' or 'O' flags
 
   then start DHCPv6 in info-request mode to get a DNS server

```
   ipv6.dhcp=info
```

 - when there aren't any downstream interfaces in shared mode
 
   then start DHCPv6 to get a prefix so that an external entity can
   lookup the prefix via D-Bus or via the lease file and do
   something with it

```
   ipv6.dhcp=auto
   ipv6.dhcp-request-prefix=yes
```

 - when the RA has the 'M' flag or the 'O' flag:
 
   then don't start DHCPv6. This was requested in the past by layered
   products because in that scenario the DHCPv6 server was not under
   their control

```
   ipv6.dhcp=no
```

 - when the RA has the 'M' flag
 
   then start DHCPv6 only to get a prefix not an address. Apparently
   there are ISPs that require this:
   https://gitlab.freedesktop.org/NetworkManager/NetworkManager/-/issues/1121#note_1605127

```
   ipv6.dhcp=info
   ipv6.dhcp-request-prefix=yes
```

 - when there are downstream interfaces in shared mode

   then start DHCPv6 to get a prefix of a given length, or suggest
   what prefix is preferred (in other words, support sending a
   hint)

```
   ipv6.dhcp-prefix-hint = 2001:::aa00::/60
```

## Part 2: configuration of downstream interfaces

Currently, NetworkManager implements a `shared` IPv6 mode that does
multiple things together:

 - enables forwarding on the interface
 - requests a IA_PD from a upstream interface
 - announces the learned prefixes via router advertisement

Features currently missing are:

 - announce a static prefix via router advertisement

 - announce custom DNS and domains via RA

 - select the subnet id to distribute from the PD
 
 - tweak the parameters of router advertisements:
   - min/max interval between RAs
   - flags: managed, otherconfig
   - advertised MTU

 - control forwarding independently of shared mode

### New properties

Similarly to what systemd does, we introduce 2 new
settings.

`prefix-delegation` configures when to request PD on other interfaces
and what to do with it. Properties:

 - `enable {yes,no*}`:
    If enabled, the PD will be requested on a different interface
    having `ipv6.dhcp-request-prefix=auto|yes`. The prefix will also
    be announced via RA if it's enabled.
 - `subnet-id NUM`
    Specifies the subnet id to assign to this interface.
 - `assign {yes*,no}`
    Whether to assign an address from the selected prefix to the
    current interface.

The reason why this is a separate setting and it's not folded into
`router-advertisement` is that one may want to configure an interface
according to a prefix without actually advertising it via NM (and for
example, advertise it through external mechanism).

`router-advertisement` configures when to start RA and its parameters:

 - `router-advertisement.enable = {yes,no*}`

    This has interactions with `ipv6.method` and is subject to the
    following normalizations:
    - `ipv6.method=shared` => `router-advertisement.enabled=yes`
    - `ipv6.method=disabled|ignore` => `router-advertisement.enabled=no`
    
    - `ipv6.method=auto` => `router-advertisement.enabled=no` (I don't
      think it's allowed to both send and receive RA?)

 - `router-advertisement.prefixes = [PREFIX,...]`

    Each prefix is a `ADDR/PREFIXLEN` followed by attributes such as:
    - `addr-conf {yes*,no}`
    - `on-link {yes*,no}`
    - `valid-lft NUM`
    - `preferred-lft NUM`

 - `router-advertimement.min-interval = NUM`
   `router-advertisement.max-interval = NUM`

    The min and max interval between sending unsolicited RAs, in seconds.

 - `router-advertisement.ra-flags = [ADVFLAG,...]`

    Where `ADVFLAG` is `default|none|managed|otherconf`.

 - `router-advertisement.ra-mtu = NUM`

    The MTU to advertise.

 - `router-advertisement.dns = ADDR,...`

    A list of static DNS servers to advertise.

 - `router-advertisement.dns-domains = STR,...`

    A list of static DNS domains to advertise.

 - `router-advertisement.auto-dns = {yes*,no}`

    Whether to take the DNS and domains from the upstream interface.

 - `ipv4.forwarding = {ignore*,yes,no}`
 - `ipv6.forwarding = {ignore*,yes,no}`

    Controls the `/proc/sys/net/ipvX/conf/$DEV/forwarding` flag.

### Examples

 - announce a static prefix and stateless DHCP:
 
```
   ipv6.method=link-local
   ipv6.forwarding=yes
   router-advertisement.enabled=yes
   router-advertisement.prefixes="2001:db8:0:1::4/64 on-link=yes valid-lft=600 preferred-lft=300"
   router-advertisement.max-interval=30
   router-advertisement.ra-flags=otherconf
```

 - configure shared mode, but with customized settings:
```
   ipv6.method=link-local          # or =shared
   ipv6.forwarding=yes
   prefix-delegation.enable=yes
   prefix-delegation.subnet-id=4
   router-advertisement.enabled=yes
   router-advertisement.use-pd=yes
   router-advertisement.max-interval=30
   router-advertisement.ra-mtu=1460
```

