# Day 1: First Packet Capture and Basic Traffic Analysis

**Date:** 27 October 2026
**Tool:** Wireshark
**Category:** Network traffic analysis / SOC fundamentals

---

## 1. Objective

Capture my own network traffic, learn to read packets in Wireshark, and practice the basic SOC workflow of deciding whether unknown traffic is benign or suspicious.

## 2. Environment

| Item | Detail |
|---|---|
| Capture tool | Wireshark |
| Host OS | Linux (VM), Firefox 128 |
| Host IP | 10.0.2.5 (private IP, VirtualBox NAT range) |
| Activity during capture | Browsing Facebook in Firefox |
| Total packets captured | 3041 |

## 3. Methodology

1. Selected the active network interface and started the capture.
2. Browsed Facebook for about 20 seconds to generate traffic, then stopped the capture.
3. Inspected the three Wireshark panels: Packet List, Packet Details, Packet Bytes.
4. Applied display filters to narrow down the traffic.
5. Used Follow HTTP Stream to read a full request and response.
6. Investigated DNS queries and answers, then verified the IP owner using ipinfo.io.

### Display filters used

| Filter | Purpose | Result |
|---|---|---|
| `ip.addr == 10.0.2.5` | Traffic to/from my host | 3040 of 3041 packets |
| `tcp.port == 80` | Unencrypted web traffic | 25 packets |
| `http` | HTTP packets only | 10 packets |
| `ip.addr == 10.0.2.5 && tcp.port == 443` | My HTTPS traffic | 1174 packets |
| `dns` | All DNS traffic | 34 packets |
| `dns.flags.response == 0` | DNS queries (questions) | 34 packets |
| `dns.flags.response == 1` | DNS responses (answers) | 34 packets |

Queries and responses both equal 34, so every DNS question received an answer.

## 4. Findings

### Finding 1: Captive portal check (benign)

- **Packet:** HTTP GET `/success.txt?ipv4` to host `detectportal.firefox.com`
- **Source / Destination:** 10.0.2.5:33402 to 162.159.142.9:80 (Cloudflare range)
- **Response:** `HTTP/1.1 200 OK`, `Content-Type: text/plain`, body `success`
- **Server headers:** `Server: Varnish`, `X-Served-By: cache-maa10235-MAA` (Chennai edge cache)
- **User-Agent:** `Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0`
- **Analysis:** Firefox checks this URL in the background to confirm internet access and detect Wi-Fi login pages. The host, response body, and User-Agent all match normal Firefox behavior.
- **Verdict:** Benign.

### Finding 2: Firefox push notification service (benign)

- **Evidence:** DNS query for `push.services.mozilla.com`
- **Analysis:** Firefox keeps a connection to Mozilla's push service so websites can send notifications.
- **Verdict:** Benign.

### Finding 3: Firefox sponsored tiles service (benign)

- **Evidence:** DNS query for `contile.services.mozilla.com`, answer `151.101.157.91`
- **IP ownership check (ipinfo.io):** AS54113, Fastly, Inc., located in Chennai, Tamil Nadu, IN
- **Analysis:** The domain belongs to Mozilla, the IP belongs to a CDN that Mozilla uses, and the Chennai location matches the `MAA` marker in the earlier header. All evidence is consistent.
- **Verdict:** Benign.

## 5. Investigation workflow practiced

```
Domain seen in DNS  ->  IP it resolved to  ->  Owner of that IP (ASN)  ->  Does it make sense?
```

If the owner does not match the domain (for example a "Mozilla" domain resolving to an unknown host in another country), the traffic would be escalated for investigation.

## 6. Key learnings

- Most modern traffic is HTTPS (port 443), so only metadata is visible: who talked to whom, when, and how much. Content is encrypted.
- Port 80 (HTTP) is plain text, so anyone on the network can read it. This is why unencrypted traffic is a security concern.
- The User-Agent header reveals the OS, browser, and version of the sender. Unusual values (`curl`, `python-requests`, empty) can indicate malware, but attackers can fake them, so it is a clue, not proof.
- HTTP status codes matter in investigations: many 404s from one IP suggests scanning, many 401/403s suggests brute force.
- DNS shows where a machine tried to go, even when the connection itself is encrypted.
- A single unknown connection is not automatically malicious. The skill is verifying it with multiple sources.

## 7. Screenshots<img width="1920" height="1080" alt="screenshots01-packet-list-three-panels" src="https://github.com/user-attachments/assets/66457596-fe79-493f-8b59-973c42a00635" />
<img width="1920" height="1080" alt="screenshots05-ipinfo-fastly" src="https://github.com/user-attachments/assets/15c118b4-8a68-4302-94f5-690061030d07" />
<img width="1920" height="1080" alt="screenshots04-dns-query-response" src="https://github.com/user-attachments/assets/c951dbae-30a2-45a0-aa07-ab90473a02b3" />
<img width="1920" height="1080" alt="screenshots03-follow-http-stream" src="https://github.com/user-attachments/assets/cb6d4cf6-60b4-498e-8ff0-1b375fb43a60" />
<img width="1920" height="1080" alt="screenshots02-http-filter-success-txt" src="https://github.com/user-attachments/assets/deb8a481-985a-43a4-b1f8-fafdf22279ab" />



## 8. Tools and references

- Wireshark
- ipinfo.io (IP and ASN lookup)

## 9. Next steps

- DNS record types (A, AAAA, CNAME, TXT, MX)
- Spotting suspicious DNS (tunneling, long subdomains)
- TCP handshake analysis and port scan detection
