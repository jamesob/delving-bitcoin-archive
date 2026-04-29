# Fingerprinting nodes: Possible Solutions

naiyoma | 2026-04-29 18:12:25 UTC | #1

This is an update on the  [Addr fingerprinting attack](https://delvingbitcoin.org/t/fingerprinting-nodes-via-addr-requests/1786) and potential mitigation strategies. It’s still a work in progress, so if you see any blind spots or have ideas on how to test these approaches, please let us know.

Tldr: Nodes that are reachable over multiple networks (e.g., IPv4 and Tor) can be fingerprinted by comparing ADDR responses across different connections. A correlation between addresses is created and further strengthened using shared timestamps.

Since the previous post, we’ve eliminated some solutions and identified others for consideration. We’ve gained deeper insights into the network topology and identified new factors to consider.

One key factor is that AddrMan has a 30-day horizon for addresses; that is, how old an address can be before it is considered stale. A stale address is one whose timestamp hasn’t been updated recently, meaning it hasn’t been seen through a direct connection, received from a peer, or self-announced. Such an address is likely offline and no longer part of the network.

These old addresses are filtered out when:

* Sending a GetAddr response.

* Adding a received address (on collision, we evict a terrible address from the bucket) to our Addrman.

This filtering prevents old addresses from circulating indefinitely across the network. With this in mind, a good solution would prevent correlation while maintaining the usefulness of timestamps.

**Note**:

I made an attempt here https://github.com/bitcoin/bitcoin/pull/33498 , but closed it due to concerns that it could make old addresses appear newer https://github.com/bitcoin/bitcoin/pull/33498#pullrequestreview-3319680730

There are two factors to consider:

* Refreshing old timestamps to newer ones may cause old addresses to be continuously gossiped.

* Conversely, making timestamps older may cause addresses to stop being gossiped prematurely, as they would be filtered out.

## **Possible Solutions**

### **1. Simple Fuzzing (±5 days)**

This solution applies random distortion to each address timestamp within a range of \[-5 days, +5 days\], and the result persists within the cache window.

**Example:**

getaddr received from peer 1 (IPv4) → cache miss → build new cache

* addr1:1776768979,timestamp + random(-5d, +5d) = timestamp + 2.3d → stored in cache

* addr3:1776965871 timestamp + random(-5d, +5d) = timestamp + 1.7d → stored in cache

* addr4: 1774102817timestamp + random(-5d, +5d) = timestamp - 4.2d → stored in cache

**Concern:**
This may average out over time. However, it is difficult to reason about the average because timestamps are continuously updated.

### **2. Fixed Timestamps Across Networks**

When responding to a getaddr request, we preserve the real timestamps for addresses on the same network as the requester, and replace timestamps of addresses on other networks with a randomized value in the past (now - 8 to 13 days).

**Example:**

If a clearnet (IPv4) peer sends getaddr:

* IPv4 addresses retain real timestamps (age naturally and are evicted via IsTerrible() at 30 days)

* Onion/IPv6/I2P addresses receive a fixed randomized timestamp

**addrman response before change**

* 1.1.1.1 | 5 days ago

* 2.2.2.2 | 10 days ago

* 3.3.3.3 | 8 days ago

* abc. onion | 3 days ago

* xyz. onion | 20 days ago

* def. onion | 28 days ago

**After applying the change above(only distorting timestamps from a different network**)

* 1.1.1.1 | 5 days ago

* 2.2.2.2 | 10 days ago

* 3.3.3.3 | 8 days ago

* abc. onion | 10 days ago -< fixed

* xyz. onion | 10 days ago -< fixed

* def. onion | 10 days ago-< fixed

**Concern:**

Old addresses might still remain in circulation longer than if we applied fixed timestamps. For example, an address that was 28 days old could become 10 days old and then be updated again and again.

### **3. Fuzzing (Making Addresses Only Older)**

We could use a range \[1, 10\] and make the address older, never newer.

For example:

* An address is 10 days old

* One hop adds +10 days → now 20 days old

* Next hop adds +8 days → now 28 days old

**Concern:**

After only two hops, the address is near the 30-day threshold and may soon be marked as “terrible” and stop being relayed.
This could reduce the number of addresses returned in getaddr responses over time, as many addresses may age out too quickly.

### **4. Fuzzing (Aging-Biased Timestamp Noise)**

We choose a bounded range of \[+4 days, -1 day\]:

* Maximum aging: +4 days

* Maximum “freshening”: −1 day

This ensures that addresses are mostly made older, with only a small chance of becoming newer.

**Concern:**

Addresses may still either circulate indefinitely or get stuck in addrman.

### **5. Hybrid Approach**

Another option is to combine approaches, for example, merge solutions 2 and 3.

At the moment, the range suggested in the solutions above is based on intuition, aiming for a middle ground that avoids pushing timestamps too far into the future or making them too recent.

I am currently testing solution 2 in my fork https://github.com/naiyoma/bitcoin/pull/16 , and also running an experiment on how getaddr/addr timestamps are updated. If these timestamps are updated frequently enough, then it would be easier to understand the range and the effects of the timestamp change.

Thoughts and feedback are welcome

-------------------------

