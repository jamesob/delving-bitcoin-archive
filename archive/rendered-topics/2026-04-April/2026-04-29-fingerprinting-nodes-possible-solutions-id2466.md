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

danielabrozzoni | 2026-05-07 14:54:27 UTC | #2

Thanks for sharing the various solutions we've been considering :)

Regarding fuzzing: I think it makes sense to only fuzz to make the timestamps older, never fresher. This means risking that addresses age too quickly, but the self announcements are hopefully enough to refresh the timestamps. The opposite problem, refreshing timestamps and potentially having departed nodes never leave addrman is far worse. (As linked in the post already, you can read how this can happen here: [#33498 (comment)](https://github.com/bitcoin/bitcoin/pull/33498#pullrequestreview-3319680730))

I partially like solution 2 - I think the idea of sending different timestamps based on whether the requester and we are on the same network or not is very interesting, and a clean way to fix the fingerprinting. 

I wonder whether, instead of implementing what you described, we could apply **fuzzing** only when the requester is on a different network, rather than using a fixed timestamp. This would essentially combine solutions 2 and 3:
- same network: real timestamp
- different network: real timestamp aged by random number of minutes between 1 and 10 days

This would remove the concern you pointed out of refreshing timestamps.

-------------------------

ArmchairCryptologist | 2026-05-08 09:36:15 UTC | #3

I may be missing something obvious, but would it not be an option to restrict GETADDR to only return entries corresponding to the network the request arrives from?

Obviously, this means a fresh node would have to connect to at least one IPv6 node to get more IPv6 node addresses and at least one Tor node to get more Tor node addresses (etc), so it has the drawback of relying more on the hardcoded seed nodes to get going initially, but making the returned address sets fully distinct should solve this type of cross-network fingerprinting.

-------------------------

danielabrozzoni | 2026-05-12 12:21:20 UTC | #4

Hey @ArmchairCryptologist, I think that’s a very insightful question :) I realized that @murch asked us the same question during [Optech Podcast #360](https://bitcoinops.org/en/podcast/2025/07/01/#fingerprinting-nodes-using-addr-messages), and I didn’t have a great answer then, nor do I have one now...

As Murch mentioned on the podcast, the idea is that having `GETADDR` responses including multiple networks makes it easier for smaller networks to see their addresses relayed, helping them find more peers more easily.

Related: there have been multiple occasions in the past where we had to slightly tweak address relay to ensure that smaller networks could still find peers reliably, see PRs [#22211](https://github.com/bitcoin/bitcoin/pull/22211), [#23077](https://github.com/bitcoin/bitcoin/pull/23077), and [#19728](https://github.com/bitcoin/bitcoin/pull/19728). Those changes relate to self-announcement relay rather than `GETADDR` responses directly, but the two mechanisms work together to propagate addresses throughout the network. As it stands today, we relay self-announcements to one or two peers even when the announcement originates from a network we cannot connect to.

Given the effort that was made into improving propagation, and the fact that we saw some propagation problems over time, I'm dubious of restricting the `GETADDR` responses to a single network... However, I would like to hear the opinion of someone that is more knowledgeable than me about this :slight_smile: 

EDIT: I realized that we only store in addrman the addresses of nodes that we can reach, and GETADDR responses come from our addrman, meaning: if we can't reach a certain network, we will relay addresses from it (as explained above), but we wouldn't store those addresses and we wouldn't include them in our GETADDR responses. So... I wonder if the "it helps with propagation" explanation is the correct one, or if addrman works like this because it always has and we never questioned it in the first place :) Need to think more about this!

-------------------------

naiyoma | 2026-05-15 10:01:36 UTC | #5



Yeah, I like this approach as well. I think it’s possible that self-announcements might be enough to update the timestamps. I haven’t gotten to measure this yet, but will do so soon.

-------------------------

naiyoma | 2026-05-15 10:50:40 UTC | #6

I think another side effect of filtering by network on the sender side is that we risk undershooting getaddr responses by a lot, and this creates a risk of filtering out addresses that could otherwise be stored in the receiver’s AddrMan.

-------------------------

