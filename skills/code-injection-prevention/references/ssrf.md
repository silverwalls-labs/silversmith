# SSRF (Server-Side Request Forgery)

An attacker tricks the server into making HTTP requests to unintended
destinations — internal services, cloud metadata endpoints, or
localhost — by controlling a URL or hostname that the server fetches.
The server's network position (inside the firewall, with access to
internal APIs and cloud instance metadata) makes it a powerful proxy
for the attacker. The defense is to validate the resolved IP address
against blocked ranges, and — ideally — restrict destinations to an
allowlist.

The principle is language-agnostic: resolve the hostname, validate the
IP, and re-validate on redirect. The examples below use Python; the
same pattern applies to any language with DNS resolution and HTTP
libraries.

## ❌ Before (vulnerable)

```python
import requests

# UNSAFE — fetches any URL the user provides, including internal
# services and cloud metadata.
# An attacker can supply: url = "http://169.254.169.254/latest/meta-data/"
# and read cloud instance credentials.

def fetch_url(url):
    response = requests.get(url, timeout=10)
    return response.text
```

The server will happily connect to internal IPs, the cloud metadata
service, or localhost — the user controls the destination.

## ✅ After (hardened)

```python
import ipaddress
import socket
import requests
from urllib.parse import urlparse, urljoin

BLOCKED_NETWORKS = [
    ipaddress.ip_network("0.0.0.0/8"),         # "this" network (reaches localhost on most OSes)
    ipaddress.ip_network("10.0.0.0/8"),
    ipaddress.ip_network("100.64.0.0/10"),     # Carrier-Grade NAT (RFC 6598)
    ipaddress.ip_network("127.0.0.0/8"),
    ipaddress.ip_network("169.254.0.0/16"),    # link-local / cloud metadata
    ipaddress.ip_network("172.16.0.0/12"),
    ipaddress.ip_network("192.168.0.0/16"),
    ipaddress.ip_network("::1/128"),
    ipaddress.ip_network("fc00::/7"),           # unique local (RFC 4193)
    ipaddress.ip_network("fe80::/10"),          # link-local IPv6
]

MAX_REDIRECTS = 5


def is_private(address: str) -> bool:
    addr = ipaddress.ip_address(address)
    # Unwrap IPv4-mapped IPv6 (::ffff:127.0.0.1 → 127.0.0.1) and
    # 6to4 addresses (2002:c0a8:0101:: → 192.168.1.1) so the check
    # hits the IPv4 blocked networks.
    if addr.ipv4_mapped is not None:
        addr = addr.ipv4_mapped
    elif getattr(addr, "sixtofour", None) is not None:
        addr = ipaddress.ip_address(addr.sixtofour)
    return any(addr in net for net in BLOCKED_NETWORKS)


def validate_url(url: str, base_url: str | None = None) -> str:
    """Resolve hostname, validate the IP, return the absolute URL if safe.

    Resolves relative URLs against base_url when provided (handles
    relative Location headers in redirects).
    """
    if base_url and not urlparse(url).scheme:
        url = urljoin(base_url, url)
    parsed = urlparse(url)
    if parsed.scheme not in ("http", "https"):
        raise ValueError(f"Blocked scheme: {parsed.scheme}")
    hostname = parsed.hostname
    if not hostname:
        raise ValueError("No hostname in URL")

    # Use explicit None check — parsed.port is 0 for ":0", which is
    # falsy but distinct from "no port specified".
    port = parsed.port if parsed.port is not None else (
        443 if parsed.scheme == "https" else 80
    )

    # Resolve and check every returned address.
    addrs = socket.getaddrinfo(hostname, port, type=socket.SOCK_STREAM)
    if not addrs:
        raise ValueError(f"DNS resolution returned no results for {hostname}")
    for info in addrs:
        address = info[4][0]
        if is_private(address):
            raise ValueError(f"Blocked destination: {address}")
    return url


# HARDENED — validate the resolved IP before connecting.
#
# NOTE: validate_url resolves DNS, then requests.get re-resolves
# independently. A sophisticated attacker can exploit the gap via
# DNS rebinding (see Key points). Production code should pin the
# validated IP and connect to it directly — the mechanism is
# HTTP-library-specific (e.g., a custom transport adapter).

def fetch_url(url: str) -> str:
    current_url = validate_url(url)
    for _ in range(MAX_REDIRECTS):
        response = requests.get(
            current_url, timeout=10, allow_redirects=False,
        )
        if not response.is_redirect:
            return response.text
        # Re-validate each redirect hop. urljoin (inside validate_url)
        # resolves relative Location headers against the current URL.
        location = response.headers.get("Location", "")
        current_url = validate_url(location, base_url=current_url)
    raise ValueError("Too many redirects")
```

The hostname is resolved via `getaddrinfo`, and every returned address
is checked against the blocked ranges. IPv4-mapped IPv6 addresses (e.g.,
`::ffff:127.0.0.1`) are unwrapped before checking. Redirects are not
followed automatically — each hop is re-validated, including relative
`Location` headers, up to a maximum of `MAX_REDIRECTS`.

## Key points

- **Validate the IP, not the hostname.** A hostname like
  `internal.attacker.com` can resolve to `10.0.0.1`. Always resolve
  first, then validate the resulting address. Unwrap IPv4-mapped IPv6
  addresses (`::ffff:10.0.0.1`) and 6to4 addresses (`2002:c0a8:0101::`
  encodes `192.168.1.1`) before checking — they bypass IPv4 network
  comparisons in most standard libraries.
- **Re-validate on every redirect hop.** A first request to a public IP
  can return a `302` pointing to `http://169.254.169.254/`. Disable
  automatic redirects, loop with a max-hops cap, resolve relative
  `Location` headers against the current URL, and validate each target.
- **Prefer allowlists.** If the application only needs to fetch from a
  known set of domains or IP ranges, allow only those — a denylist of
  blocked ranges is always one new range behind.
- **Block all private and special-use ranges.** `0.0.0.0/8` ("this"
  network — reaches localhost on most OSes), RFC 1918 (`10/8`,
  `172.16/12`, `192.168/16`), RFC 6598 (`100.64/10`, Carrier-Grade
  NAT — used internally by cloud VPCs), loopback (`127/8`), link-local
  (`169.254/16`), IPv6 unique-local (`fc00::/7`), IPv6 link-local
  (`fe80::/10`), and cloud metadata (`169.254.169.254`,
  `fd00:ec2::254`).
- **DNS rebinding.** An attacker's DNS server can return a public IP on
  first resolution and a private IP on second. The example above has a
  TOCTOU gap between `validate_url` (resolution) and `requests.get`
  (re-resolution). Production code must pin the validated IP and
  connect to it directly — typically via a custom HTTP transport adapter
  or by connecting to the IP with a `Host` header — so the library
  never re-resolves the hostname.
- **Scheme restriction.** Block `file://`, `gopher://`, `dict://`, and
  other non-HTTP schemes — some HTTP libraries follow them, and they
  can access local files or internal services.
