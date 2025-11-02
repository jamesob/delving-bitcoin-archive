# Public Disclosure: Denial of Service using HTLC in Cashu

1440000bytes | 2025-11-02 06:27:53 UTC | #1

## Vulnerability

The size of preimage was not validated in nutshell versions < 0.18.0. This allows the attacker to fill the mint's database and disk with arbitrary data. 

```python
    def _verify_htlc_spending_conditions(
        self,
        proof: Proof,
        secret: HTLCSecret,
        message_to_sign: Optional[str] = None,
    ) -> bool:

        htlc_secret = secret
        # hash lock
        if not proof.htlcpreimage:
            raise TransactionError("no HTLC preimage provided")
        # verify correct preimage (the hashlock) if the locktime hasn't passed
        now = time.time()
        if not htlc_secret.locktime or htlc_secret.locktime > now:
            if not hashlib.sha256(
                bytes.fromhex(proof.htlcpreimage)
            ).digest() == bytes.fromhex(htlc_secret.data):
                raise TransactionError("HTLC preimage does not match.")
        return True
```

NUT-14 can be used to create cashu tokens with a preimage and NUT-07 to see the preimage stored by the mint.

## Proof of Concept

https://uncensoredtech.substack.com/p/denial-of-service-using-htlc-in-cashu

## Fix

It was fixed by *lollerfirst* in https://github.com/cashubtc/nutshell/pull/803

## Timeline

19 October 2025: I reported the vulnerabillity to cashu-dev@pm.me  
19 October 2025: Cashu dev team acknowledged it as a serious issue and [opencash](https://opencash.dev/) rewarded with 100k sats  
21 October 2025: It was fixed in https://github.com/cashubtc/nutshell/commit/f84028ca3f8f0b476f7be8c29b58666f075be2c2  
28 October 2025: v0.18.0 was released with the fix  
29-31 October 2025: I reached out to several mints and requested to update nutshell  
2 November 2025: Public Disclosure

## Advisory for mints and users

Some mints have still not updated nutshell. They should backup the database and run the mint with v0.18.0. Reach out to Cashu dev team if you experience any issues while updating the mint.

Users should check the version of the mint with `https://<mint_url>/v1/info`. Stop using the mint if it supports NUT-14 and running nutshell version older than 0.18.0.

-------------------------

