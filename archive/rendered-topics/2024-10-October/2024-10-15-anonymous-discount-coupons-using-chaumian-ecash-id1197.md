# Anonymous discount coupons using chaumian ecash

1440000bytes | 2024-10-15 09:50:03 UTC | #1

![image|651x383, 100%](upload://mooFQBgZrIszLihZUQz0EVP7iU5.jpeg)

There were 2 talks in recent bitcoin++ conference about use of ecash protocol for other stuff and not money. Blind signatures have been used in coinjoin by Wabisabi for coordination and I think they can be useful for discount coupons as well.

> More complicated implementations are possible where even the server doesn't learn the mapping.

> E.g. Using chaum blind signatures: The users connect and provide inputs (and change addresses) and a cryptographically-blinded version of the address they want their private coins to go to; the server signs the tokens and returns them. The users anonymously reconnect, unblind their output addresses, and return them to the server. The server can see that all the outputs were signed by it and so all the outputs had to come from valid participants. Later people reconnect and sign.

https://bitcointalk.org/?topic=279249

In open source projects, incentivizing users and contributors with discount coupons could be useful. Example: Paid [nostr relays](https://legacy.nostr.watch/relays/find) 


**Proof of Concept**

```
import os
import hashlib
from dataclasses import dataclass
from typing import Dict, Tuple

from ecdsa import SECP256k1, SigningKey, VerifyingKey
from ecdsa.ellipticcurve import Point

DOMAIN_SEPARATOR = b"Secp256k1_HashToCurve_Relay_"

def int_to_bytes(x: int) -> bytes:
    return x.to_bytes((x.bit_length() + 7) // 8, byteorder='big')

def int_from_bytes(xbytes: bytes) -> int:
    return int.from_bytes(xbytes, byteorder='big')

def hash_to_curve(x: bytes) -> Point:
    msg_hash = hashlib.sha256(DOMAIN_SEPARATOR + x).digest()
    x = int_from_bytes(msg_hash)
    
    curve = SECP256k1.curve
    p = curve.p()
    
    while True:
        x = x % p
        y_squared = (pow(x, 3, p) + 7) % p
        y = pow(y_squared, (p + 1) // 4, p)
        
        if pow(y, 2, p) == y_squared:
            return Point(curve, x, y)
        
        x += 1

@dataclass
class BlindedMessage:
    amount: int
    id: str
    B_: str

@dataclass
class BlindSignature:
    amount: int
    id: str
    C_: str

@dataclass
class Proof:
    amount: int
    id: str
    secret: str
    C: str

class Relay:
    def __init__(self):
        self.private_keys = {}
        self.public_keys = {}
        self.spent_secrets = set()

    def generate_keypair(self, amount: int):
        private_key = SigningKey.generate(curve=SECP256k1)
        public_key = private_key.get_verifying_key()
        key_id = hashlib.sha256(public_key.to_string()).hexdigest()[:8]
        self.private_keys[amount] = private_key
        self.public_keys[amount] = public_key
        return key_id, public_key.to_string().hex()

    def sign(self, blinded_message: BlindedMessage) -> BlindSignature:
        private_key = self.private_keys[blinded_message.amount]
        B_ = VerifyingKey.from_string(bytes.fromhex(blinded_message.B_), curve=SECP256k1).pubkey.point
        C_ = private_key.privkey.secret_multiplier * B_
        return BlindSignature(
            amount=blinded_message.amount,
            id=blinded_message.id,
            C_=C_.to_bytes().hex()
        )

    def verify(self, proof: Proof) -> bool:
        if proof.secret in self.spent_secrets:
            return False
        public_key = self.public_keys[proof.amount]
        Y = hash_to_curve(proof.secret.encode())
        C = VerifyingKey.from_string(bytes.fromhex(proof.C), curve=SECP256k1).pubkey.point
        
        private_key = self.private_keys[proof.amount]
        expected_C = private_key.privkey.secret_multiplier * Y
        
        return C.x() == expected_C.x() and C.y() == expected_C.y()

    def redeem(self, proof: Proof):
        if self.verify(proof):
            self.spent_secrets.add(proof.secret)
            return True
        return False

class User:
    def __init__(self):
        self.blinding_factors: Dict[str, Tuple[int, str]] = {}

    def blind(self, amount: int, key_id: str, public_key_hex: str) -> BlindedMessage:
        x = os.urandom(32).hex()
        Y = hash_to_curve(x.encode())
        r = int.from_bytes(os.urandom(32), byteorder='big') % SECP256k1.order
        B_ = Y + SECP256k1.generator * r
        B_hex = B_.to_bytes().hex()
        self.blinding_factors[B_hex] = (r, x, public_key_hex)
        return BlindedMessage(amount=amount, id=key_id, B_=B_hex)

    def unblind(self, blind_signature: BlindSignature, B_: str) -> Proof:
        r, x, public_key_hex = self.blinding_factors.pop(B_)
        C_ = VerifyingKey.from_string(bytes.fromhex(blind_signature.C_), curve=SECP256k1).pubkey.point
        K = VerifyingKey.from_string(bytes.fromhex(public_key_hex), curve=SECP256k1).pubkey.point
        
        C_ = Point(C_.curve(), C_.x(), C_.y())
        K = Point(K.curve(), K.x(), K.y())
        
        C = C_ + (SECP256k1.order - r) * K
        return Proof(
            amount=blind_signature.amount,
            id=blind_signature.id,
            secret=x,
            C=C.to_bytes().hex()
        )

def main():
    relay = Relay()
    user = User()
    amount = 100
    key_id, public_key_hex = relay.generate_keypair(amount)
    print(f"Relay generated keypair for amount {amount} with ID {key_id}")

    blinded_message = None
    blind_signature = None
    proof = None

    while True:
        print("\n1. Blind")
        print("2. Sign")
        print("3. Unblind")
        print("4. Verify")
        print("5. Redeem")
        print("6. Exit")
        choice = input("Enter your choice: ")

        if choice == '1':
            blinded_message = user.blind(amount, key_id, public_key_hex)
            print(f"Blinded message: {blinded_message}")
        elif choice == '2':
            if blinded_message is None:
                print("Please blind a message first.")
            else:
                blind_signature = relay.sign(blinded_message)
                print(f"Blind signature: {blind_signature}")
        elif choice == '3':
            if blind_signature is None:
                print("Please get a blind signature first.")
            else:
                proof = user.unblind(blind_signature, blinded_message.B_)
                print(f"Proof: {proof}")
        elif choice == '4':
            if proof is None:
                print("Please unblind the signature first.")
            else:
                is_valid = relay.verify(proof)
                print(f"Proof is valid: {is_valid}")
        elif choice == '5':
            if proof is None:
                print("Please unblind the signature first.")
            else:
                is_spent = relay.redeem(proof)
                print(f"Token redeemed successfully: {is_spent}")
        elif choice == '6':
            break
        else:
            print("Invalid choice. Please try again.")

if __name__ == "__main__":
    main()
```

This does not involve any custodians and users do not depsosit bitcoin for minting ecash. It can be gifted to other users. I plan to use it for incentivizing users and [market makers](https://uncensoredtech.substack.com/p/market-makers-in-joinstr) in joinstr can also offer discounts for regular users. 

**Alternative**

Normal discount and promo codes can be used to achieve the same thing but ecash protocol improves privacy.

-------------------------

1440000bytes | 2024-10-29 09:36:00 UTC | #2

Known issues with the code: https://xcancel.com/not_nothingmuch/status/1846660499460837742

-------------------------

