# IPREF Gateway

IPREF (**IP** addressing with **References**) is a networking protocol ([draft-augustyn-intarea-ipref](https://www.ietf.org/archive/id/draft-augustyn-intarea-ipref-06.html)) that eliminates the need for traditional NAT port forwarding. It provides secure, direct connectivity between hosts using reference-based addressing which provides compatability between IPv4 and IPv6 networks.

IPREF provides means of communication across different address spaces, such as private networks behind NAT, overlapping networks, or even across different protocols. It can traverse NAT, NAT6, and cross protocol IPv4/IPv6. It is inherently peer-to-peer.

This gateway is a reference implementation of the IPREF protocol that integrates an IPREF forwarder with an address mapper. The forwarder handles bidirectional packet translation between local IP addresses and IPREF addresses, transmitting encapsulated packets through UDP tunnels to peer gateways. The address mapper manages the allocation of references and encoded addresses, maintaining mappings between local addresses and their IPREF equivalents.

For complete functionality, the gateway requires two supporting components: the `dns-agent` (which synchronizes DNS records to inform the mapper of locally-hosted services) and CoreDNS with the IPREF plugin (which provides IPREF-aware DNS resolution for local clients). Only the gateway itself needs to understand IPREF—hosts on the local network operate normally without modification.

## Key Advantages

- **No port forwarding required** - Services are accessible via IPREF addresses automatically
- **Simplified configuration** - No complex NAT rules or firewall exceptions

## Quick Start

### Prerequisites

- Linux 64-bit (tested on Rocky Linux, RHEL, Debian, Ubuntu)
- 1 vCPU, 2GB RAM minimum
- UDP port 1045 accessible
- Basic networking tools (`dig`, `ping`, `traceroute`)

### Demo Hosts

Test your IPREF installation with these demo hosts:

| Host | Location |
|------|----------|
| k41.nexsand.us | United States |
| m41.nexsand.ca | Canada |
| o61.nexsand.uk | United Kingdom |

These websites can be viewed with a successful installation of the gateway:
- https://k41.nexsand.us
- https://m41.nexsand.ca
- https://o61.nexsand.uk

### Client Mode Setup

Client mode allows you to access IPREF network resources without publishing services.

```bash
# Create directories
sudo mkdir -p /var/lib/ipref /run/ipref /etc/coredns

# Start gateway in client mode
sudo ipref-gw \
    -data /var/lib/ipref \
    -gateway-bind 0.0.0.0 \
    -gateway-pub 0.0.0.0 \
    -encode-net 10.240.0.0/12 \
    -mapper-socket /run/ipref/mapper.sock

# Start dns-agent (in another terminal)
sudo ipref-dns-agent \
    -ea-ipver 4 \
    -gw-ipver 4 \
    -m unix:///run/ipref/mapper.sock \
    -t 60

# Start CoreDNS (in another terminal)
sudo ipref-coredns -conf /etc/coredns/Corefile
```

### Configure DNS Resolution

Point your system's DNS resolver to the local CoreDNS instance:

```bash
# Temporarily set DNS resolver (will reset on reboot)
echo "nameserver 127.0.0.1" | sudo tee /etc/resolv.conf

# For systemd-resolved systems, alternatively:
sudo mkdir -p /etc/systemd/resolved.conf.d
echo -e "[Resolve]\nDNS=127.0.0.1\nDomains=~." | sudo tee /etc/systemd/resolved.conf.d/ipref.conf
sudo systemctl restart systemd-resolved
```

### Test Connectivity

```bash
# Test DNS resolution
dig k41.nexsand.us

# Test connectivity (should resolve to 10.24x.x.x address)
ping k41.nexsand.us

# Access demo web services
curl http://k41.nexsand.us
```

## Building from Source

For a complete IPREF gateway, you'll need three binaries: `gw`, `dns-agent`, and `coredns`.

### Integrated Build (Recommended)

The easiest way to build all components is using the provided Makefile. First, clone the required repositories alongside the `gw` repository:

```bash
# Clone all required repositories in the same directory
git clone https://github.com/ipref/gw
git clone https://github.com/ipref/dns-agent
git clone https://github.com/coredns/coredns
git clone https://github.com/ipref/coredns-plugin-ipref

# Checkout specific CoreDNS version
cd coredns
git checkout v1.12.1
cd ..

# Build all components
cd gw
make

# Find binaries in bin/ directory
ls bin/
```

This will automatically:
- Build the gateway binary
- Build the DNS agent
- Configure and build CoreDNS with the IPREF plugin
- Place all binaries in `bin/` directory

<details>
<summary><strong>Individual Component Builds</strong> (click to expand)</summary>

You can also build each component separately if needed:

#### Build the gateway
##### Prerequisites

- Go 1.22 or later
- Git

##### Steps

1. Clone the repository:
```bash
git clone https://github.com/ipref/gw.git
cd gw
```

2. Install dependencies:
```bash
go mod download
```

3. Build the project:
```bash
go build -o gw
```

The build will generate an executable named `gw` in your current directory.

##### Verify installation

To verify the build was successful:

```bash
./gw -h
```

#### Build the DNS agent

The DNS agent informs the gateway about the mappings between public IPREF addresses and private IP addresses by periodically querying DNS servers.

Clone the repository and build it:

```sh
git clone https://github.com/ipref/dns-agent.git
cd dns-agent/
go build
```

The binary will be named `dns-agent`. Verify that it was built successfully:

```sh
./dns-agent -h
```

#### Build CoreDNS with the `ipref` plugin

CoreDNS can be used to host the special resolver (using the `ipref` plugin) and also optionally your `*.internal` and/or your public nameservers.

The special resolver receives requests from the local network and translates AA records into A/AAAA records by asking `gw` to dynamically allocate addresses in the encoding network that are mapped to the IPREF address that appears in the AA record.

To build CoreDNS, you'll need to clone the CoreDNS repo and also the ipref plugin repo inside CoreDNS's tree. You'll also need to add the dependencies for the ipref plugin to CoreDNS's `go.mod`.

```sh
git clone https://github.com/coredns/coredns.git
cd coredns/
git checkout v1.12.1
echo "require github.com/ipref/common v1.3.1" >> go.mod
cd plugin/
git clone https://github.com/ipref/coredns-plugin-ipref.git
mv coredns-plugin-ipref/ ipref/ # Rename
```

Additionally, to ensure that CoreDNS's build system can find the plugin, this line needs to be added to the `plugin.cfg` file at the top level of the CoreDNS repo:

```
ipref:ipref
```

The order in `plugin.cfg` determines the order that plugins apply. It is recommended to place the above line after the line `auto:auto`.

Once these steps are complete, you can run `make` to build CoreDNS. Verify that it was build successfully:

```sh
./coredns -plugins
```

Make sure `ipref` is in the list of plugins. If not, then the build system might not have recognized the plugin. Also make sure that the `require` line mentioned above is still in `go.mod` - Go's build system might have removed it if it couldn't find the plugin. Make sure the plugin repo is in the correct place and has the correct name before running `make`.

</details>

<details>
<summary><strong>Detailed Configuration Examples</strong> (click to expand)</summary>

## Configuration

For this example, we'll assume that you've decided to use:

- `*.internal` as your internal, private TLD for hosting local IP addresses
- `*.example.com` as your public domain for hosting IPREF addresses
- `ns1.example.com` and `ns2.example.com` are your public nameservers for `example.com`
- `10.240.0.0/12` as your encoding network (the virtual, local address space that the gateway uses to emulate remote IPREF hosts)
- `1.2.3.4` is your gateway's public IP address

Run the gateway using these arguments:

```sh
gw \
    -data /var/lib/ipref \
    -gateway-bind 0.0.0.0 \
    -gateway-pub 1.2.3.4 \
    -encode-net 10.240.0.0/12 \
    -mapper-socket /run/ipref/mapper.sock
```

The directory `/var/lib/ipref` is where the mapping database will be stored. `/run/ipref/mapper.sock` is the path to the Unix domain socket used for communication between `gw`, `dns-agent`, and the CoreDNS ipref plugin (it will be created by `gw` on startup).

`-gateway-bind` can be used to tell the gateway to only listen for UDP tunnel packets on a specific interface. Specifying `0.0.0.0` will tell it to listen on all interfaces.

Run the DNS agent like so:

```sh
dns-agent \
    -ea-ipver 4 \
    -gw-ipver 4 \
    -m unix:///run/ipref/mapper.sock \
    -t 60 \
    internal:example.com:ns1.example.com,ns2.example.com
```

The options `-ea-ipver` and `-gw-ipver` specify the IP version for the local network and UDP tunnel respectively. The `-t` option specifies the approximate interval in minutes at which the DNS agent will query the nameservers for updates.

CoreDNS requires a Corefile (configuration file). Depending on your use case, there are a variety of ways to configure it. This example demonstrates a basic setup where CoreDNS acts as the special resolver (using the ipref plugin) and hosts the `*.internal` domain from a zone file.

`/etc/coredns/Corefile`:

```Corefile
internal {
    #bind 127.0.0.2 # Optional
    file /etc/coredns/db.internal
    transfer {
        to *
    }
    log
    debug
}
. {
    #bind 127.0.0.2 # Optional
    ipref {
        upstream 8.8.8.8
        ea-ipver 4
        gw-ipver 4
        mapper /run/ipref/mapper.sock
    }
    #forward . 8.8.8.8 8.8.4.4 # Optional
    log
    debug
}
```

The `ea-ipver` and `gw-ipver` options are the same as for `dns-agent`. The `upstream` specifies the nameserver to query for AA records.

The `forward` built-in plugin can optionally be used to forward requests for non-IPREF domains to another nameserver. For requests where the domain name had no AA records in the upstream DNS server or no mappings could be found/created, the DNS query is proxied to the nameservers listed after `forward .`.

If you do not use `forward`, then the DNS server will return SERVFAIL for requests where no mappings could be found/created (even if there was an A or AAAA record on the upstream DNS server). This can be useful if you want to put the special resolver in your list of nameservers before your usual nameserver - most operating systems will try the next nameserver if the first one returned SERVFAIL. In this case, it can be useful to use the `bind` option to bind the nameserver to an alternative address (eg. `127.0.0.2`) so it can exist alongside another nameserver (eg. `systemd-resolved`).

Your `/etc/coredns/db.internal` is a zone file containing your local IP addresses. For example:

```
$ORIGIN internal.
$TTL 120

internal.  IN  SOA  localhost. admin.internal. ( 1 120 120 120 120 )
internal.  IN  NS   localhost.

gw.internal.      IN  A  10.0.0.1 ; The gateway itself
host11.internal.  IN  A  10.0.0.11
host22.internal.  IN  A  10.0.0.22
```

You can then start CoreDNS with:

```sh
coredns -conf /etc/coredns/Corefile
```

Finally, you will need to add AA records to your nameservers for your public domain (`example.com` in this example). Your zone file for this domain might look like:

```
$ORIGIN example.com.
$TTL 3600

example.com.  IN  SOA  ns1.example.com. admin.example.com. ( 2024123101 7200 3600 1209600 3600 )
example.com.  IN  NS   ns1
example.com.  IN  NS   ns2

gw.example.com.      IN  A    1.2.3.4

gw.example.com.      IN  TXT  "AA gw.example.com + 1"    ; By convention, ref 1 is reserved for the gw itself
host11.example.com.  IN  TXT  "AA gw.example.com + 1025" ; Non-gw hosts should begin at 1024+1
host22.example.com.  IN  TXT  "AA gw.example.com + 1-22" ; A hyphen separates groups of 16 bits
```

</details>

## Advanced Setup

For detailed setup instructions including:
- Publishing your own services
- Production deployment with systemd
- DNS configuration for various providers
- Troubleshooting guide

See the comprehensive documentation in this repository's [docs/](docs/) directory.

## Publishing Services

IPREF allows you to publish arbitrary number of services without NAT port forwarding, port manipulation, or global IP addresses. You can publish thousands of services from within a private address space.

### Key Steps

1. **Set up internal DNS server** - Publish local addresses using `.internal` TLD
2. **Set up external DNS server** - Publish IPREF addresses in publicly accessible DNS
3. **Configure gateway** - Gateway matches domain segments to map IPREF to local addresses

### Example Configuration

For detailed examples of publishing services, see the Configuration section above and the [docs/](docs/) directory.

## IPv6 Support

IPREF supports IPv6 connectivity. If your ISP provides IPv6 addresses and your router supports IPv6, you can connect to both IPv4 and IPv6 networks and reach external hosts over either protocol.

The gateway can traverse NAT, NAT6, and cross-protocol IPv4/IPv6 connections without changes to your local network.

## Documentation

- [Setup Guide](docs/SETUP.md) - Detailed setup instructions
- [Troubleshooting](docs/TROUBLESHOOTING.md) - Common issues and solutions
- [Examples](examples/) - Configuration examples
- [Systemd Services](systemd/) - Production deployment

## Related Repositories

- [dns-agent](https://github.com/ipref/dns-agent) - DNS synchronization agent
- [coredns-plugin-ipref](https://github.com/ipref/coredns-plugin-ipref) - CoreDNS IPREF plugin
- [common](https://github.com/ipref/common) - Shared IPREF libraries

## License

[GPL-2.0](https://choosealicense.com/licenses/gpl-2.0/)
