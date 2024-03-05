# Property-based testing for Bitcoin Core

bruno | 2024-03-05 14:28:23 UTC | #1

I just want to share some initial experiments I've done regarding property-based testing for Core.

Property-based tests are **designed to test the aspects of a property that should always be true** . They allow for a range of inputs to be programmed and tested within a single test rather than writing a different test for every value you want to test. 

First, let's check a Bitcoin Core's functional test that tests resource exhaustion. See:

```py
def test_resource_exhaustion(self):
        self.log.info("Test node stays up despite many large junk messages")
        conn = self.nodes[0].add_p2p_connection(P2PDataStore())
        conn2 = self.nodes[0].add_p2p_connection(P2PDataStore())
        msg_at_size = msg_unrecognized(str_data="b" * VALID_DATA_LIMIT)
        assert len(msg_at_size.serialize()) == MAX_PROTOCOL_MESSAGE_LENGTH

        self.log.info("(a) Send 80 messages, each of maximum valid data size (4MB)")
        for _ in range(80):
            conn.send_message(msg_at_size)

        # Check that, even though the node is being hammered by nonsense from one
        # connection, it can still service other peers in a timely way.
        self.log.info("(b) Check node still services peers in a timely way")
        for _ in range(20):
            conn2.sync_with_ping(timeout=2)

        self.log.info("(c) Wait for node to drop junk messages, while remaining connected")
        conn.sync_with_ping(timeout=400)

        # Despite being served up a bunch of nonsense, the peers should still be connected.
        assert conn.is_connected
        assert conn2.is_connected
        self.nodes[0].disconnect_p2ps()
```

By looking at this function, some questions were raised on my mind:

* What if the node had more than two connections? 
* What if connections happen at different times?
* What if one of the connections sent more or less than 80 messages?
* What if other connections/disconnections happen during the path?

Well, there are a lot of probabilities, but it seems a perfect scenario for a property-based test. Since Bitcoin Core has its test framework and it is written in Python, I used it to draft a property-based test using [TSTL](https://github.com/agroce/tstl).

> TSTL is a domain-specific language (DSL) and set of tools to support automated generation of tests for software. This implementation targets Python. You define (in Python) a set of components used to build up a test, and any properties you want to hold for the tested system, and TSTL generates tests for your system. TSTL supports test replay, test reduction, and code coverage analysis, and includes push-button support for some sophisticated test-generation methods. In other words, TSTL is a *property-based testing* tool.

I wrote a TSTL file to replicate the idea of the resource exhaustion test from `p2p_invalid_messages.py`. See:
```
...
init: TestShell().setup(num_nodes=1, setup_clean_chain=True)
finally: TestShell().shutdown()
pool: <connection> 100
pool: <node> 1
pool: <int> 1

<int> := <[80..200]>
<node> := TestShell().nodes[0]
<connection> := <node>.add_p2p_connection(P2PDataStore())

for _ in range(<int>): conn = <connection>; conn.send_message(msg_at_size); conn.sync_with_ping(timeout=400)
<connection>.sync_with_ping(timeout=2)

property: <connection>.is_connected
```

It has a pool of 100 possible connections, one node, and one integer value (from 80 to 200). TSTL will create different scenarios with different amounts of messages being sent, different orders of connections, synchronization, and other stuff. Regardless of all this, we must ensure that any disconnection will happen.

TSTL is able to create scenarios with more than 1000 actions from this simple script.

-------------------------

