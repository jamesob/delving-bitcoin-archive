# Pools without covenants

1440000bytes | 2024-05-04 03:09:40 UTC | #1

<h1>Problem</h1>

Coinjoin process requires lot of interaction between peers

<h1>Research</h1>

- [`SIGHASH_ALL | SIGHASH_ANYONECANPAY`](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-May/021696.html) reduces some steps 
- Nostr encrypted channels can be used so that users don't need to be always online
- Payment Pools allow [rolling coinjoin](https://rubin.io/bitcoin/2021/12/10/advent-13/#fnref:greg) but not possible without covenants :warning:

<h1>Solution</h1>

Using `SIGHASH_ALL | SIGHASH_ANYONECANPAY` and nostr encrypted channels we can create pools in which users can enter any time and exit together. Unilateral exits are also possible which requires lot of pre-signed transactions, multisig and some interaction.

I have used this approach in [electrum plugin for joinstr](https://gitlab.com/1440000bytes/joinstr/-/tree/main/plugin) which is still a work in progress. There are different ways to create nostr channels and I have used basic method using NIP 4. See other NIPs in [nostr repository](https://github.com/nostr-protocol/nips) that can be used to implement better encrypted channels.

![GG7Tn0cbIAAJzdB|690x441](upload://b8vWzjfdnYoGRp5NsV5xbDX72c9.png)

Paid relay is used to avoid denial of service attacks and below are 2 functions used for creating and joining pools:

```
    def create_channel(self, denomination, peers, timeout, address, wallet, tab_widget):

        letters = string.ascii_lowercase
        pool_id = ''.join(random.choice(letters) for i in range(10)) + str(int(time.time()))

        private_key = PrivateKey()
        pub_key=private_key.public_key.hex()
        channel = {"type": "new_pool","id": pool_id, "public_key": pub_key,"denomination": float(denomination), "peers": int(peers), "timeout": int(time.time())+timeout, "relay": self.config.get('nostr_relay', 0)}
        channel_creds = channel
        channel_creds["private_key"] = private_key.hex()     

        pool_path = os.path.expanduser(os.path.join('~', '.electrum', "pools.json"))
    
        if os.path.exists(pool_path) and os.stat(pool_path).st_size > 0:
            with open(pool_path, 'r') as file:
                pools = json.load(file)
        else:
            pools = []

        pools.append(channel_creds)

        with open(pool_path, 'w') as file:
            json.dump(pools, file, indent=2)

        self.publish_event(channel, 999999)

        self.register_output(wallet,address, tab_widget)

        thread = threading.Thread(target=self.run_checkevents, args=('pool_msg','join_request', None, None, None, None, pub_key, peers))
        thread.daemon = True
        thread.start()
```

```
    def join_channel(self, request, pubkey, relay):
        event = self.publish_event(request, 4, pubkey, relay=relay, qr=True)
        
        if not hasattr(self, 'dialog'):
            self.dialog = QDialog()
            self.dialog.setWindowTitle("Pool Request")
            self.dialog_layout = QVBoxLayout()
            self.dialog.setLayout(self.dialog_layout)

            self.message_label = QLabel("Waiting for pool credentials...")
            self.message_label.setAlignment(Qt.AlignCenter)
            self.dialog_layout.addWidget(self.message_label)

            self.progress_bar = QProgressBar()
            self.progress_bar.setTextVisible(False)
            self.dialog_layout.addWidget(self.progress_bar)

            self.dialog.setFixedWidth(400)

            self.dialog.show()

        self.timer = QTimer()
        self.timer.timeout.connect(self.update_progress)
        self.timer.start(100)

        thread = threading.Thread(target=self.run_checkevents, args=('own_msg', None, None, None, None, event.public_key))
        thread.daemon = True
        thread.start()
```

---

There was some [discussion](https://github.com/stevenroose/covenants.info/pull/16#issuecomment-1891021885) about the terms so I have just called them "pools". Such pools are not limited to coinjoin and could be used for other things. Example: Discreet Log Contracts

-------------------------

