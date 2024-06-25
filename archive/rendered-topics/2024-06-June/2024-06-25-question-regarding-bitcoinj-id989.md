# Question Regarding BitcoinJ

myles | 2024-06-25 20:18:46 UTC | #1

I've recently been developing software for a personal, offline key management service to hold the keys of my friends and family. I had the idea to hold each of their funds in deterministic wallets where I hold the seed and they hold the pass phrase and whenever they want to move funds, we get together.

About me: I am a relatively new programmer (~2 years through my CS degree) and am looking for feedback on my design.

Currently, I want to make it so my friend can create a "PartiallySignedTransaction" object of which I can interact with through the "signWithSeed" method which I hope can return a serialized canonical Bitcoin transaction. However, I am completely stumped as to how to implement this even after scowering the BitcoinJ documentation and watching videos for 3 days.

Any help would be appreciated!

```
import org.bitcoinj.core.Transaction;
import org.bitcoinj.crypto.DeterministicKey;

public class PartiallySignedTransaction {
  private final String passPhrase;
  private final Transaction transaction;

  public PartiallySignedTransaction(Transaction transaction, String passPhrase) {
    this.passPhrase = passPhrase;
    this.transaction = transaction;
  }

  public byte[] signWithSeed(byte[] seed) {
    //Generate the masterKey using the seed and passPhrase.
    //Within transaction, get inputs,
    //Sign each input with private key equal to path of input's private key (derived from masterKey).
    return null;
  }

  private DeterministicKey getMasterKeyFromSeedAndPassPhrase(byte[] seed, String passPhrase) {
    return null;
  }
}
```

-------------------------

