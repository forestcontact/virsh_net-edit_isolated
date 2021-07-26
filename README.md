# virsh_net-edit_isolated
Configuration for a full isolated LAN in Virt-Manager. Dnsmasq config forwards DNS requests to Stubby, routes traffic over Wireguard.


## Step 1 - setup deps and host

Host configuration: Install [Stubby](https://getdnsapi.net/blog/dns-privacy-daemon-stubby/) and [Virt-Manager].


Create a new isolated network, akin to the following config: (via `virsh net-edit isolated`)

``` xml
<network xmlns:dnsmasq='http://libvirt.org/schemas/network/dnsmasq/1.0'>
  <name>isolated</name>
  <uuid>128bits-of-uuid-here</uuid>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:00:00:b1'/>
  <domain name='network'/>
  <ip address='10.69.0.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='10.69.0.128' end='10.69.0.254'/>
    </dhcp>
  </ip>
  <dnsmasq:options>
    <dnsmasq:option value='dhcp-option=3,10.69.0.2'/>
    <dnsmasq:option value='server=127.0.0.1#5353'/>
  </dnsmasq:options>
</network>
```

Note the last lines - this specifies DNS requests are to be forwarded to port 5353 on the VM host, and the upstream gateway ("router" in dnsmasq parlance) is located at 10.69.0.2. Without these special dnsmasq:option elements, traffic will not be routed correctly.

On the host (10.69.0.1) - launch a `stubby` instance with a config similar to this one: (located at /etc/stubby/stubby.yml, or passed to a stubby instance with the -C flag to specify config path.

``` yaml
resolution_type: GETDNS_RESOLUTION_STUB

dns_transport_list:
  - GETDNS_TRANSPORT_TLS

tls_authentication: GETDNS_AUTHENTICATION_REQUIRED

tls_query_padding_blocksize: 128

edns_client_subnet_private : 1

round_robin_upstreams: 1

idle_timeout: 10000

listen_addresses:
  - 127.0.0.1@5353

upstream_recursive_servers:
  - address_data: 1.1.1.1
    tls_auth_name: "cloudflare-dns.com"
  - address_data: 1.0.0.1
    tls_auth_name: "cloudflare-dns.com"
  - address_data: 9.9.9.9
    tls_auth_name: "dns.quad9.net"
  - address_data: 9.9.9.10
    tls_auth_name: "dns.quad9.net"
```

## Step 2 - Router

* Make a small VM with at least two network interfaces - one with WAN access, one on the isolated LAN created above.
* Install `wireguard-tools` and `wireguard` - we will use Wireguard to route traffic for our LAN.
* Assign a static IP of 10.69.0.2/24 to the NIC on the isolated LAN.

Adjust the wireguard.conf file from your VPN vendor to resemble the following:

``` yaml
[Interface]
PrivateKey = REDACTED==
Address = 1.2.3.4/32

# flush tables
PostUp = iptables -F
PostUp = iptables -t nat -F
PostUp = iptables -t mangle -F

# Enable IPv4 forwarding
PostUp = sysctl -q -w net.ipv4.ip_forward=1

# apply routing
PostUp = iptables -t nat -A POSTROUTING -o %i -j MASQUERADE
PostUp = iptables -t mangle -A PREROUTING -j TTL --ttl-inc 1
PostUp = iptables -A FORWARD -i %i -o ens4 -m state --state RELATED,ESTABLISHED -j ACCEPT
PostUp = iptables -A FORWARD -i ens4 -o %i -j ACCEPT

# flush iptables rules when wg goes down
PostDown = iptables -F
PostDown = iptables -t nat -F
PostDown = iptables -t mangle -F


[Peer]
PublicKey = REDACTED==
AllowedIPs = 0.0.0.0/0,::0/0
Endpoint = 5.6.7.8:51820
```

Launch with `sudo wg-quick up ./wireguard.conf` and enjoy!

NB: The use of MASQUERADE is not 100% required, but it comes in handy...

NB: IPv4-only!



