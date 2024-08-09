# Onion Messaging DoS Threat Mitigations

gijswijs | 2024-08-07 06:47:09 UTC | #1

Hey people,


This years Financial Cryptography and Data Security 2024 (FC'24) conference had an interesting paper that explores the DoS threat that is introduced with Onion Messaging. It explores some mitigations that I would like to share here.

The full paper is called "Short Paper: Onion Messages on Leash" by Amin Bashiri and Majid Khabbazian, both from the University of Alberta. It can be obtained here: https://fc24.ifca.ai/preproceedings/104.pdf

## Calculating the maximum number of hops an OM can traverse

Because of the bigger payload the paper claims that an OM can travel 504 hops. Although the paper makes a mistake here in assuming a 65 byte per hop data in the payload. The paper concludes that when assuming the max OM payload length (32834) it leaves space for 504 hops. But 65 bytes is the size of the legacy `hop_data` payload format, which we don't use anymore.

### Calculating per hop payload size

A forwarding node in a blinded path will receive a cipher text payload containing the following data:

```
onionmsg_payloads
    	length (1 byte)
    	onionmsg_tlv
            	next_node_id:
            	type: 4 (1 byte)
            	length: 33 (1 byte)
            	value: 33 bytes
    	hmac (32 bytes)
```

If my calculations are correct the per hop payload size is 68 bytes. Assuming the maximum payload size minus 100 bytes for the header and trailing bytes, it leaves 32734, divided by 68 equals 481, a very large number.

## Hard Leash

The first mitigation is limiting the number of hops an OM can traverse. By allocating a portion of the complete payload as being allowed to contain routing information, we can limit the maximum number of hops a message can travel. An interesting data point to share here is what Tor has implemented. While in Tor three is the default circuit length of hops (entry (or guard) node, a middle node, and an exit node), eight is the maximum. Eight is derived from the number of `RELAY_EARLY` cells you are allowed to send in Tor. https://spec.torproject.org/proposals/110-avoid-infinite-circuits.html

## Soft Leash

The second mitigation is a PoW requirement for adding hops.

The PoW-based algorithm suggested in the paper scales exponentially with the number of hops. This is achieve by linking each hop’s PoW to the preceding hop’s, effectively creating a chain of PoWs. With this approach, each additional hop appended to this chain exponentially increases the computational challenge of calculating the PoW for the entire path. The PoW is linked to the latest Bitcoin block’s hash as a (albeit not true) source of randomness.

Again this is a mitigation that is also implemented in Tor: https://blog.torproject.org/introducing-proof-of-work-defense-for-onion-services/

## Rate Limiting

The third suggestion made in the paper is on the topic of rate limiting. The suggestion made is to limit the rate of OMs being send on any _outgoing_ link. The rate limit should be set proportional to the total channel capacity of the receiving node: $\alpha_{A} \cdot C_{B}$, where $\alpha_{A}$ is an adjustable parameter.

## Routing

The final suggestion is to have routing for OM consider the channel capacities of the nodes when choosing a random path. By choosing random paths weighted based on the sum of capacities of each node’s channels, the route tends towards the more capitalized nodes. Because of the rate limiting suggested above, this has the benefit of being the route with a higher rate limit, so a higher chance of success. But additionally, for an attacker to be on the route would become prohibitively more expensive.

It is good to see LN security being taken seriously in academic literature and I would like to solicit the thoughts and opinions on this subject in the broader LN community.

-------------------------

MattCorallo | 2024-08-09 15:23:02 UTC | #2

Yea, that paper is cool, sadly it doesn't actually make an argument for there being a "DoS threat" from onion messages, it just assumes there is one and then addresses mitigations. It is the case that we should probably reduce the ability for OMs to route-loop, which is the main focus of the paper, however. Still, more formal analysis of what the actual DoS risk is is needed, as well as analysis of the "backpressure" mitigation that lightning developers had agreed to implement if/when this becomes an issue.

-------------------------

