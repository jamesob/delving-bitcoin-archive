# Hardcoded seeds, DNS seeds and Darknet nodes

virtu | 2024-09-12 09:53:46 UTC | #1

Since darknet-only nodes cannot use DNS seeds[^1] to learn about addresses of Bitcoin nodes to connect to, I wanted to get a feel on how such nodes fare on hardcoded-seed data. To this end, I created some statistics on hardcoded seed nodes[^2] to investigate how fast the number of reachable seeds decreases over time.

![hardcoded_seeds|690x343](upload://2gGLdxo1P03Eweufksjb1VnObcx.jpeg)


Each chart shows the number of reachable hardcoded seeds over time for a particular network type (I excluded Cjdns because I've only been collecting this data for a couple of weeks) and includes data for the last four Bitcoin Core releases (red lines correspond to release dates).

Some observations:
- Despite IPv4 appearing to be the most "short-lived" network type, of the seed nodes hardcoded around 21 months ago into v24 still ~50 are reachable.
- I2P nodes seem to be the most "long-lived" ones (the sharp drop shortly after v27 release in April 24 was due to a [DoS attack on the I2P network](https://geti2p.net/de/blog/post/2024/04/25/stormy_weather)).
- It looks like there was an oversight during the v26 release to include new seed nodes.
- Before v27, the number of Onion and I2P seed nodes was rather low (nodes for these networks were manually curated up until recently, and the number of nodes included depended on the data source).

To answer my original question about darknet-only nodes: they should be doing fine. Still I wondered, why they're excluded from taking advantage of the DNS seed-mechanism, which has several advantages: for one, DNS seeds provide more up-to-date on reachable nodes, increasing the probability of quickly finding a reachable address; for another, they might improve privacy by advertising from a larger pool of nodes, thus reducing the likelihood of a hardcoded seed collecting statistics about bootstrapping nodes; etc.

In an old GitHub comment I read that darknet addresses are too large for seeding, it would be unnecessary and it's not practical to access DNS over these networks. I'd like to address the first and last points with a [PoC darknet seeder](https://github.com/virtu/darkseed) I wrote which is capable of serving Onion, I2P and Cjdns addresses using a BIP155-like encoding and is reachable via IPv4 and Cjdns (DNS/UDP) as well as Onion and I2P (DNS/TCP).

The point about necessity is one I don't feel qualified to answer. If most people are running mixed clearnet-darknet nodes, there's no necessity. For darknet-only nodes, it might be worth the effort (~100 loc to create custom DNS `NULL` queries in Bitcoin Core since right now we conveniently use `getaddrinfo` which does not require any low-level DNS functionality). Happy about feedback before taking this any further!

[^1]: For those who don't know, DNS seeds are used by a new Bitcoin node as the default way to learn about Bitcoin nodes it can connect to. The node sends a DNS query to one or more of the DNS seeds whose addresses are hardcoded into the binary, and receive a DNS reply containing a number of IPv4 and IPv6 addresses of nodes believed to be reachable. The node then connects to one or more of these addresses, sends a `getaddr` message to the node it connected to and (ideally) receives an `addr` reply from the node containing around 1000 addresses of other Bitcoin nodes.
[^2]: For those in need of a refresher, hardcoded seeds are used as a fallback when a new node who doesn't know about any peers fails to solicit peer addresses via DNS seeds; for such instances, the Bitcoin Core binary contains a number of hardcoded addresses which the node can connect to and ask for other nodes' addresses by sending a `getaddr` message.

-------------------------

sipa | 2024-09-10 12:51:20 UTC | #2

@virtu Interesting, can `getaddrinfo` read these NULL encodings, on all supported platforms?

-------------------------

1440000bytes | 2024-09-10 13:13:40 UTC | #3

[quote="virtu, post:1, topic:1123"]
To answer my original question about darknet-only nodes: they should be doing fine. Still I wondered, why they’re excluded from taking advantage of the DNS seed-mechanism, which has several advantages: for one, DNS seeds provide more up-to-date on reachable nodes, increasing the probability of quickly finding a reachable address; for another, they might improve privacy by advertising from a larger pool of nodes, thus reducing the likelihood of a hardcoded seed collecting statistics about bootstrapping nodes; etc.
[/quote]

I think it should be the other way around. We should completely remove the dependency on DNS, and DNS seeds should be replaced with the IP addresses of nodes run by those developers. These nodes can respond to `getaddr` for bootstrapping.

 
[quote="virtu, post:1, topic:1123"]
I’d like to address the first and last points with a [PoC darknet seeder ](https://github.com/virtu/darkseed) I wrote which is capable of serving Onion, I2P and Cjdns addresses using a BIP155-like encoding and is reachable via IPv4 and Cjdns (DNS/UDP) as well as Onion and I2P (DNS/TCP).
[/quote]

Interesting use of NULL records. IIUC TXT records could also be used although laanwj recently shared issues related with such DNS records: https://github.com/bitcoin/bitcoin/pull/30007#issuecomment-2094289500

-------------------------

virtu | 2024-09-11 07:43:47 UTC | #4

As far as I know `getaddrinfo` will only return A and AAAA records.

I understand we don't want to add some dependency library for this. But since we only need to send a particular query I don't think that's necessary. From what I learned writing the seeder, DNS is refreshingly straightforward. Here's some C++ to send and receive a NULL query to demonstrate.

```C++
#include <arpa/inet.h>
#include <cstring>
#include <iostream>
#include <netinet/in.h>
#include <sstream>
#include <string>
#include <sys/socket.h>
#include <unistd.h>
#include <vector>

struct DNSHeader {
  uint16_t id;
  uint16_t flags;
  uint16_t q_count;
  uint16_t ans_count;
  uint16_t auth_count;
  uint16_t add_count;
};

struct DNSQuestion {
  std::vector<unsigned char> qname;
  uint16_t qtype;
  uint16_t qclass;

  DNSQuestion(const std::string &domain, uint16_t type, uint16_t cls)
      : qtype(type), qclass(cls) {
    // Convert domain to DNS format: prefix parts with their length and
    // end with null byte (e.g. dnsseed.21.ninja -> 7dnsseed2215ninja0)
    std::stringstream ss(domain);
    std::string segment;
    while (getline(ss, segment, '.')) {
      qname.push_back(static_cast<uint8_t>(segment.size()));
      qname.insert(qname.end(), segment.begin(), segment.end());
    }
    qname.push_back(0);
  }

  std::vector<unsigned char> serialize() const {
    std::vector<unsigned char> serialized;
    serialized.insert(serialized.end(), qname.begin(), qname.end());
    serialized.insert(
        serialized.end(), reinterpret_cast<const unsigned char *>(&qtype),
        reinterpret_cast<const unsigned char *>(&qtype) + sizeof(qtype));
    serialized.insert(
        serialized.end(), reinterpret_cast<const unsigned char *>(&qclass),
        reinterpret_cast<const unsigned char *>(&qclass) + sizeof(qclass));
    return serialized;
  }
};

int main() {
  const std::string domain = "dnsseed.21.ninja";
  const std::string nameserver = "89.116.30.184";

  // Prepare DNS query
  std::vector<unsigned char> query;
  DNSHeader header = {static_cast<uint16_t>(getpid() % 65536), htons(0x0100), htons(1), 0, 0, 0}; // 0x0100 for recursion desired
  query.insert(query.end(), reinterpret_cast<unsigned char *>(&header), reinterpret_cast<unsigned char *>(&header) + sizeof(DNSHeader));
  DNSQuestion question = {domain, htons(10), htons(1)}; // 10 for NULL record, 1 for IN class
  auto serializedQuestion = question.serialize();
  query.insert(query.end(), serializedQuestion.begin(), serializedQuestion.end());

  int sock = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
  struct sockaddr_in dest = { .sin_family = AF_INET, .sin_port = htons(53), .sin_addr = {.s_addr = inet_addr(nameserver.c_str())}};
  ssize_t bytes_sent = sendto(sock, query.data(), query.size(), 0, (struct sockaddr *)&dest, sizeof(dest));
  printf("Sent query (size=%ld, header id=%d)\n", bytes_sent, reinterpret_cast<const DNSHeader *>(query.data())->id);

  std::vector<unsigned char> response(512);
  socklen_t addr_len = sizeof(dest);
  ssize_t bytes_received = recvfrom(sock, response.data(), response.size(), 0, (struct sockaddr *)&dest, &addr_len);
  printf("Received reply (size=%ld, header id=%d)\n", bytes_received, reinterpret_cast<const DNSHeader *>(response.data())->id);

  close(sock);
  return 0;
}
```

-------------------------

virtu | 2024-09-11 08:02:10 UTC | #5

[quote="1440000bytes, post:3, topic:1123"]
Interesting use of NULL records. IIUC TXT records could also be used although laanwj recently shared issues related with such DNS records: [chainparams: Add achow101 DNS seeder by achow101 · Pull Request #30007 · bitcoin/bitcoin · GitHub ](https://github.com/bitcoin/bitcoin/pull/30007#issuecomment-2094289500)
[/quote]

Concerning TXT records, I actually tried them first because I thought there might be an advantage to having "human-readable" data in the record. I used a base85 encoding to represent the 256-bit keys/hashes of Onion/Tor address. But using human-readable letters doesn't make the data human interpretable, so I went with NULL records which are more efficient because they allow for binary data.

Good point about the lack of support for caching in the comment though. I'll look into to. But given the 60-second TTL used by most DNS seeds, I highly doubt  more than a handful of bootstrapping nodes ever used cached DNS data.

-------------------------

sipa | 2024-09-11 18:05:17 UTC | #6

All else being equal, we'd certainly prefer not needing a dependency or ad-hoc DNS implementation to do DNS seed queries, but that on itself isn't the main point here IMHO.

The reasons for wanting DNS-based seeding in the first place over more obvious alternatives (e.g., make a P2P connection to the seeder and send them a `GETADDR` request...) is the worldwide caching infrastructure and the ubiquitous access through ~every operating system for it. The caching makes it cheap to operate, and adds some notion of privacy: when you're using your ISP's recursive resolver, the DNS seed operator doesn't see exactly what IPs are running Bitcoin nodes there, or exactly how many are present.

Not using the OS's resolver and configuration means losing some of these advantages. A dependency or ad-hoc DNS resolver implementation means complexity to make it work on all supported platforms. Making such an approach find the system's configured DNS server adds to that, or alternatively when sending the query directly to the seed, loses the caching/privacy benefits. So does switching to non-A/AAA records unless they're reliably cached too.

In my view, if we're going to be losing these advantages anyway, it's simpler to switch to P2P-style seeding (already used when running on Tor, FWIW).

-------------------------

virtu | 2024-09-12 06:03:21 UTC | #7

Thanks for the feedback.

Although not exhaustive, I've looked at various public nameservers (Google, Control D, Quad9, OpenDNS Home, Cloudflare, AdGuard, CleanBrowsing, Alternate DNS), and they all seem to cache TXT and NULL records if the authoritative server's answer matches certain criteria.

If the inclusion of a custom DNS lookup is a blocker though, I'll not pursue the TXT/NULL record approach any further.

Something that just came to mind but I haven't fully thought through: One could encode the data inside AAAA records (with a reserved prefix perhaps, to avoid confusing them with actual IPv6 addresses). In this case, the lookup in Bitcoin Core would remain unchanged (ie, `getaddrinfo`); there'd just have to be some extra decoding logic triggered by the reserved prefix.

-------------------------

1440000bytes | 2024-09-12 18:41:16 UTC | #8

[quote="virtu, post:7, topic:1123"]
Something that just came to mind but I haven’t fully thought through: One could encode the data inside AAAA records (with a reserved prefix perhaps, to avoid confusing them with actual IPv6 addresses). In this case, the lookup in Bitcoin Core would remain unchanged (ie, `getaddrinfo`); there’d just have to be some extra decoding logic triggered by the reserved prefix.
[/quote]

This is interesting and I have never thought about it before.

Onion v3 and i2p addresses are 256 bits, while IPv6 addresses are 128 bits, so I'm not sure if encoding would help. However, it could be useful for sharing IP addresses with port numbers, for nodes using non-default ports.

Encoding IPv4 address and port number in IPv6 address: https://gitlab.com/-/snippets/3746764

-------------------------

virtu | 2024-09-12 19:40:15 UTC | #9

[quote="1440000bytes, post:8, topic:1123"]
Onion v3 and i2p addresses are 256 bits, while IPv6 addresses are 128 bits, so I’m not sure if encoding would help. However, it could be useful for sharing IP addresses with port numbers, for nodes using non-default ports.
[/quote]

Well, once you start misappropriating AAAA records there's no reason to only do it lightly. With successive records, one could encode arbitrary data.

-------------------------

sipa | 2024-09-12 21:47:52 UTC | #10

You'd need some encoding that lets you reconstruct the order, because recursive resolvers may change the order of entries in a response.

-------------------------

virtu | 2024-09-23 14:33:41 UTC | #11

Just implemented this to see if there might be any caveats.

The only con is the poor encoding efficiency of around 50%. NULL records can have an arbitrary length, so you're paying the resource record overhead of ten bytes (2B for type, class and length each plus 4B for TTL) only once. With AAAA records, there's ~12B of overhead (the ten-byte record overhead plus one or two bytes for a hardcoded restricted IPv6 prefix and the ordering information) for ~14B of payload.

On the upside, this approach works with `getaddrinfo` and does not interfere with server-side caching.

The demo implementation uses the ff00::/8 prefix to signal custom encoding and the next 8 bits for ordering (in theory, this could be reduced to just five bits, since you cannot fit more than 17 24-byte records into a 512B DNS message with a 12B header).

Note that regular IPv6 addresses are note affected: `2a02:4780:28:a4bb::1` and `2a01:4ff:1f0:8d5c::1` are regular AAAA records. Everything following that (`ff00:...`, `ff01:...`, `ff02:...`) is the new encoding. Overall, 11 AAAA records are required to encode two onion, two I2P and two CJDNS addresses.

```bash
» nix shell . -c darkdig seed.21.ninja. --nameserver 127.0.0.1 --port 8053

; <<>> darkdig 0.13 <<>> @127.0.0.1 -p 8053 -l INFO seed.21.ninja.
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51259
;; flags: qr rd, QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 0

;;QUESTION SECTION:
; domain=seed.21.ninja., rdclass=IN, rdtype=ANY

;;ANSWER SECTION:
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=A, data=162.249.228.218
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=A, data=84.247.133.104
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=A, data=152.117.88.43
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=A, data=109.210.214.5
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=A, data=67.207.84.53
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=AAAA, data=2a02:4780:28:a4bb::1
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=AAAA, data=2a01:4ff:1f0:8d5c::1
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=AAAA, data=ff00:604:1a1b:53af:68b2:1754:ca5b:9b8e
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=AAAA, data=ff01:dbed:7368:3d5f:e79d:28d3:55bb:be39
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=AAAA, data=ff02:e8a8:4961:ef5c:4f6:5b5f:de9d:aa90
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=AAAA, data=ff03:fcc2:cbfa:87ba:d504:43d7:d8e9:4902
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=AAAA, data=ff04:469f:725d:1312:32b0:aea:8405:6070
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=AAAA, data=ff05:5e8f:753a:f0a4:b20f:50d1:6e6a:4f08
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=AAAA, data=ff06:7f4e:bf5:a7d3:4acb:7828:6c08:4d2d
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=AAAA, data=ff07:837b:5d8:a70e:a682:650d:74d1:9ab8
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=AAAA, data=ff08:18c2:8726:1b25:7d6c:adb:c945:7b81
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=AAAA, data=ff09:fd4d:edb:99f9:e106:fc1f:22c3:95dc
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=AAAA, data=ff0a:a3af:4a93:8251:beb9:1858:6fc:d1c2
domain=seed.21.ninja., ttl=60, rdclass=IN, rdtype=AAAA, data=ff0b:9bc:31a:7580:d828:38a1:f148:c400
;; ->>custom AAAA encoding<<-
;; ->>custom AAAA-encoded address <<- record: 0, net_type: onion_v3, address: dinvhl3iwilvjss3tohnx3ltna6v7z45fdjvlo56hhukqslb55of7iyd.onion
;; ->>custom AAAA-encoded address <<- record: 1, net_type: onion_v3, address: 6znv7xu5vkipzqwl7kd3vvieipl5r2kjajdj64s5cmjdfmak5kcht6qd.onion
;; ->>custom AAAA-encoded address <<- record: 2, net_type: i2p, address: mbyf5d3vhlykjmqpkdiw42spbb7u4c7vu7juvs3yfbwaqtjnqn5q.b32.i2p
;; ->>custom AAAA-encoded address <<- record: 3, net_type: i2p, address: 3ctq5jucmugxjum2xammfbzgdmsx23ak3peuk64b7vgq5w4z7hqq.b32.i2p
;; ->>custom AAAA-encoded address <<- record: 4, net_type: cjdns, address: fc1f:22c3:95dc:a3af:4a93:8251:beb9:1858
;; ->>custom AAAA-encoded address <<- record: 5, net_type: cjdns, address: fcd1:c209:bc03:1a75:80d8:2838:a1f1:48c4

;; Query time: 4 msec
;; SERVER: 127.0.0.1#8053
;; WHEN: Mon Sep 23 16:23:57 CEST 2024
;; MSG SIZE  rcvd: 503
```

-------------------------

