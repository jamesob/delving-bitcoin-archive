# Stealth addresses using nostr

1440000bytes | 2025-07-09 17:13:25 UTC | #1

Nostr uses the same [cryptography](https://github.com/nostr-protocol/nips/blob/477e3dfd4d672a830d157f32c41795b7a143b9d7/01.md) to generate keys as bitcoin. So, users can generate new keys from npub and receive bitcoin payments.

**Motivation**

* BIP 47 v1 uses OP_RETURN for notifications
* Silent payments scanning affects UX

**Protocol**

Alice would generate a new key for Bob and share the notification as encrypted message using NIP-17. Bob's wallet will save the details to receive payments from Alice in future. Alice will get a new address for Bob by incrementing counter.

A BIP or NIP could be written to describe the specifications and below is a proof of concept:

**Proof of Concept**

![image|690x260](upload://7g6XHh5s41VKyx4lEGGtRtdYaKP.png)

![image|690x248](upload://qRmBNxJucVxqcmn4l63T3F3tV2B.png)

```python
#!/usr/bin/env python3

import hashlib
import hmac
import secrets
import json
import os
from typing import Tuple, Optional

from nostr.key import PrivateKey, PublicKey
import coincurve


class NostrKeyGenerator:
    
    def __init__(self, private_key_hex: str):
        if len(private_key_hex) != 64:
            raise ValueError("Private key must be 64 hex characters")
        
        try:
            private_key_bytes = bytes.fromhex(private_key_hex)
            self.private_key = PrivateKey(private_key_bytes)
            self.public_key = self.private_key.public_key
        except ValueError:
            raise ValueError("Invalid hex string for private key")
    
    def get_private_key_hex(self) -> str:
        return self.private_key.hex()
    
    def get_public_key_hex(self) -> str:
        return self.public_key.hex()
    
    def compute_shared_secret(self, other_public_key_hex: str) -> bytes:
        shared_secret = self.private_key.compute_shared_secret(other_public_key_hex)
        if isinstance(shared_secret, bytes):
            return shared_secret
        else:
            return bytes.fromhex(shared_secret)
    
    def generate_stealth_public_key(self, recipient_public_key_hex: str, counter: int = 0) -> str:
        shared_secret = self.compute_shared_secret(recipient_public_key_hex)
        counter_bytes = counter.to_bytes(4, 'big')
        
        key_factor = hmac.new(shared_secret, counter_bytes + b"stealth", hashlib.sha256).digest()
        
        recipient_key_bytes = bytes.fromhex('02' + recipient_public_key_hex)
        recipient_point = coincurve.PublicKey(recipient_key_bytes)
        
        factor_private_key = coincurve.PrivateKey(key_factor)
        factor_point = factor_private_key.public_key
        
        combined_point = recipient_point.combine([factor_point])
        
        compressed_pubkey = combined_point.format(compressed=True)
        return compressed_pubkey[1:].hex()
    
    def derive_stealth_private_key(self, sender_public_key_hex: str, counter: int = 0) -> Tuple[str, str]:
        shared_secret = self.compute_shared_secret(sender_public_key_hex)
        counter_bytes = counter.to_bytes(4, 'big')
        
        key_factor = hmac.new(shared_secret, counter_bytes + b"stealth", hashlib.sha256).digest()
        
        factor_int = int.from_bytes(key_factor, 'big')
        my_private_int = int(self.private_key.hex(), 16)
        
        secp256k1_order = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141
        new_private_int = (my_private_int + factor_int) % secp256k1_order
        
        new_private_bytes = new_private_int.to_bytes(32, 'big')
        new_private_key = PrivateKey(new_private_bytes)
        
        new_public_key = new_private_key.public_key
        
        coincurve_private = coincurve.PrivateKey(new_private_bytes)
        coincurve_public = coincurve_private.public_key
        compressed_pubkey = coincurve_public.format(compressed=True)
        
        return (new_private_bytes.hex(), compressed_pubkey[1:].hex())


def validate_hex_key(key_hex: str, key_type: str, expected_length: int) -> bool:
    if len(key_hex) != expected_length:
        print(f"Error: {key_type} must be {expected_length} hex characters")
        return False
    
    try:
        bytes.fromhex(key_hex)
        return True
    except ValueError:
        print(f"Error: {key_type} must be valid hex")
        return False


def generate_stealth_public_key():
    print("\n\033[34mYou are the SENDER. Generate a stealth public key for the recipient.\033[0m")
    print()
    
    sender_private = input("Enter your private key (64 hex chars): ").strip()
    if not validate_hex_key(sender_private, "Private key", 64):
        return
    
    recipient_public = input("Enter recipient's public key (64 hex chars): ").strip()
    if not validate_hex_key(recipient_public, "Public key", 64):
        return
    
    try:
        counter = int(input("Enter counter value (0-999, default 0): ").strip() or "0")
        if counter < 0 or counter > 999:
            print("Counter must be between 0 and 999")
            return
    except ValueError:
        print("Counter must be a number")
        return
    
    try:
        keygen = NostrKeyGenerator(sender_private)
        stealth_pubkey = keygen.generate_stealth_public_key(recipient_public, counter)
        
        print(f"\n\033[32mGenerated Stealth Public Key: {stealth_pubkey}\033[0m")
        print()
        print("\033[33mShare these details with the recipient:\033[0m")
        print(f"\033[33m   - Your public key: {keygen.get_public_key_hex()}\033[0m")
        print(f"\033[33m   - Counter used: {counter}\033[0m")
        
    except Exception as e:
        print(f"Error generating stealth public key: {e}")


def derive_stealth_private_key():
    print("\n\033[34mYou are the RECIPIENT. Derive the private key for a stealth public key.\033[0m")
    print()
    
    recipient_private = input("Enter your private key (64 hex chars): ").strip()
    if not validate_hex_key(recipient_private, "Private key", 64):
        return
    
    sender_public = input("Enter sender's public key (64 hex chars): ").strip()
    if not validate_hex_key(sender_public, "Public key", 64):
        return
    
    try:
        counter = int(input("Enter counter value used by sender: ").strip())
        if counter < 0 or counter > 999:
            print("Counter must be between 0 and 999")
            return
    except ValueError:
        print("Counter must be a number")
        return
    
    try:
        keygen = NostrKeyGenerator(recipient_private)
        stealth_private, stealth_public = keygen.derive_stealth_private_key(sender_public, counter)
        
        print(f"\n\033[95mDerived Stealth Private Key: {stealth_private}\033[0m")
        print(f"\033[32mDerived Stealth Public Key: {stealth_public}\033[0m")
        print()
        
    except Exception as e:
        print(f"Error deriving private key: {e}")


def generate_random_keypair():
    
    private_key_bytes = secrets.token_bytes(32)
    private_key = PrivateKey(private_key_bytes)
    
    print(f"\nPrivate Key: {private_key.hex()}")
    print(f"Public Key:  {private_key.public_key.hex()}")
    print()


def main():
    while True:
        print("\n" + "="*70)
        print("                    Nostr Stealth Key Generator")
        print("="*70)
        print("1. Generate random keypair")
        print("2. Get stealth public key")
        print("3. Get stealth private key")
        print("4. Exit")
        print("="*70)
        
        choice = input("Enter your choice (1-4): ").strip()
        
        if choice == "1":
            generate_random_keypair()
        elif choice == "2":
            generate_stealth_public_key()
        elif choice == "3":
            derive_stealth_private_key()
        elif choice == "4":
            break
        else:
            print("Invalid choice, please try again")


if __name__ == "__main__":
    main()
```

-------------------------

