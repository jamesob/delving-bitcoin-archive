# CTV, APO, CAT activity on signet

ajtowns | 2024-11-14 17:34:11 UTC | #1

BIP 118 (`SIGHASH_ANYPREVOUT`, APO) and BIP 119 (`OP_CHECKTEMPLATEVERIFY`, CTV) have been enabled on signet for about two years now, since late 2022. More recently, BIP 347 (`OP_CAT`) was also activated -- it's been available for about six months now.

Here's a brief investigation into how they've been used:

## APO

 * Most of the transactions were spends of the coinbase payout, and can be observed as spends of address [tb1pwzv7fv35yl7ypwj8w7al2t8apd6yf4568cs772qjwper74xqc99sk8x7tk](https://mempool.space/signet/address/tb1pwzv7fv35yl7ypwj8w7al2t8apd6yf4568cs772qjwper74xqc99sk8x7tk). All those spends reuse the same APOAS signature, spending multiple block rewards back to faucet addresses, generally with very large amounts lost to fees.
 * There are a handful of spends with the script `1 OP_CHECKSIG`, ie signed by the internal public key:
    * [Tue, 27 Sep 2022 06:29:14 +0000](https://mempool.space/signet/tx/ee6f6eda93a3d80f4f65f2e1000334132c9a014b3ed3dec888fdcc1f3441f52c)
    * [Tue, 27 Sep 2022 06:38:49 +0000](https://mempool.space/signet/tx/2cbcc4857e6ee8510d9479c01fbf133a9a2cde3f5c61ccf9439c69b7b83334ba)
    * [Wed, 28 Sep 2022 19:52:25 +0000](https://mempool.space/signet/tx/53a9747546e378956072795351e8436cf704da72d235c8ac7560787b554a4d3f)
    * [Mon, 12 Dec 2022 12:05:38 +0000](https://mempool.space/signet/tx/97297668d465c8104e3bbb91357e1282ff07dba2d5246779825a946e593caf4f)
  * There are substantially more spends using APO as an overridable CTV in the second quarter of 2023, with a script of the form `<sig> 1 OP_CHECKSIG` -- spending via the script path means that you have to satisfy the fixed signature, so are limited in how your tx is structured, however since the IPK was used to generate that signature, it could equally be used to generate a key path signature with no restrictions. They conclude around the time that [bips PR#1472](https://github.com/bitcoin/bips/pull/1472) was filed.
  
    * [Sat, 04 Mar 2023 18:41:03 +0000](https://mempool.space/signet/tx/a4cd8ddad062c0418bf60b3c542982cbbff04216d9347be6c0e762c1770e6d4d)
    * [Sun, 05 Mar 2023 15:44:51 +0000](https://mempool.space/signet/tx/932f3e537d810107f66d6bd9ee6d79758e89ff3da8c698492447d307eb9fa5b3)
    * [Thu, 09 Mar 2023 00:14:56 +0000](https://mempool.space/signet/tx/34dde3f7b8a47260cebeb896b1830a78d487c85cec21cdee94e442e1aa8bef09)
    * [Thu, 09 Mar 2023 00:36:27 +0000](https://mempool.space/signet/tx/72a9d0a0f4fe1334b8d89561eedfa41b7b1bbd41bef764e04aed917dfcb64a31)
    * [Wed, 05 Apr 2023 13:32:50 +0000](https://mempool.space/signet/tx/d65d98068d1bdc5b22fa7b3eb81ee6d3c1d8c9309d157c376ba3e65a10925f61)
    * [Wed, 05 Apr 2023 19:54:51 +0000](https://mempool.space/signet/tx/bcd2f5fd8c843574d2faad985dd3c572f203ee25fd5540952a866da659893e87)
    * [Thu, 06 Apr 2023 00:07:20 +0000](https://mempool.space/signet/tx/37ab7bddbd7ffff9f62ccc5c5d06b4d0a0e8577b416f3510ef94eaf48508cfe8)
    * [Mon, 10 Apr 2023 01:23:03 +0000](https://mempool.space/signet/tx/b82ba38911c59f5cd7874a7858751464dc0dbb362e188ecf54f243e80a7e6cd2)
    * [Sat, 22 Apr 2023 22:09:05 +0000](https://mempool.space/signet/tx/aab9904584db7e88e02b5dea0bc757bf10a181df37558a8f1ffdf77063e74f7c)
    * [Sat, 22 Apr 2023 22:58:28 +0000](https://mempool.space/signet/tx/03bc4207e91cac0a5663c6e77faecfcec92d910ceb9133f1ee1da26d9302a496)
    * [Sat, 22 Apr 2023 23:14:46 +0000](https://mempool.space/signet/tx/6a1612e6c590eb452af7b24c6cc58cbf1f2bf4b6b7203b25f7d3fcbf27e0d00d)
    * [Sun, 23 Apr 2023 00:57:13 +0000](https://mempool.space/signet/tx/8fa60b6f6cd4d1fdd17d4d05b44675d6652298cf4294118c841d4c565ad61f78)
    * [Sun, 23 Apr 2023 07:04:50 +0000](https://mempool.space/signet/tx/de57960c7da04eb2dd8658db01791dc6b37ccd19683b6aaecd262ecebdd584d7)
    * [Sun, 23 Apr 2023 07:37:48 +0000](https://mempool.space/signet/tx/febe39138e341ab4426dcdb99b1e067e198f8649f8066afdd6afd38f23cbb03e)
    * [Sun, 23 Apr 2023 11:58:23 +0000](https://mempool.space/signet/tx/7ef8d3c9b5935f31c8769ff4fa7732e9404aee35d5f55513d244dd00c0a1da28)
    * [Sun, 23 Apr 2023 12:37:50 +0000](https://mempool.space/signet/tx/64bb8d85de7dcc6884df0a1b73f71f8938c567261255bc1c860dc676a8a91327)
    * [Sun, 23 Apr 2023 13:29:34 +0000](https://mempool.space/signet/tx/36f535870082af1e6e465e73d928c637efb0c3ec3ac56793c605d60df70fd1e5)
    * [Sun, 23 Apr 2023 15:28:57 +0000](https://mempool.space/signet/tx/be708374f2ef03e58a59a5c238c9afbfd2ec22d7afde1f66652cd7933bb004b1)
    * [Sun, 23 Apr 2023 15:55:32 +0000](https://mempool.space/signet/tx/ab3e910d010597dfaaa2bd326652cabb140d7612d911f8a4cb30f399dcda7e92)
    * [Sun, 23 Apr 2023 19:46:44 +0000](https://mempool.space/signet/tx/f14c28489c25f5212cf96eaf974f75688b90fb31c4ccf9baa6f371376103d9da)
    * [Sun, 23 Apr 2023 20:20:17 +0000](https://mempool.space/signet/tx/9e9d4c84237b5322161dc101ec1006da0e422fbbaaf3ec8252aa0ab7b547992d)
    * [Sun, 23 Apr 2023 23:39:36 +0000](https://mempool.space/signet/tx/360b1a0a4bf9bc8e19ebd94726858eefb6fe79f80565c81a92ae882faa48fac1)
    * [Mon, 24 Apr 2023 11:51:02 +0000](https://mempool.space/signet/tx/88c5340d458753a7e1b948cef0f5f0fb06fc6a726aa1f928de07c70ae956a145)
    * [Mon, 24 Apr 2023 12:27:22 +0000](https://mempool.space/signet/tx/89dd842d730bef5706c070a750560984f588059e1631920212e0756b7b99321c)
    * [Mon, 24 Apr 2023 20:11:49 +0000](https://mempool.space/signet/tx/647ec3ba10dcacb82f30df4c48a046b1205fae08d9a04d5ca7d0e5c1c3ab8383)
    * [Mon, 24 Apr 2023 20:23:54 +0000](https://mempool.space/signet/tx/de3d7016a6f4f5d735f30937814421df50a84321fc0362a56823942a15852533)
    * [Mon, 24 Apr 2023 20:34:18 +0000](https://mempool.space/signet/tx/609cab42c9c6a02ec9f0968ba6c66113bc2950a045a2a11f6cd8b0281e785383)
    * [Mon, 24 Apr 2023 21:01:51 +0000](https://mempool.space/signet/tx/54a995d9ba8e1cc4c8dc4c21292ec6dde0bf45390bbcada361456b9fe6349a5e)
    * [Mon, 24 Apr 2023 21:12:31 +0000](https://mempool.space/signet/tx/ce8e3ee501546690e6cf06ea5258fdc188fe79f9ff56b4b013dd367d8154bc00)
    * [Tue, 25 Apr 2023 03:58:02 +0000](https://mempool.space/signet/tx/497423974a051f87c6e9f6ec20162221f35c8c546c7cd84e8a473d19e0e0a69a)
    * [Tue, 25 Apr 2023 04:16:09 +0000](https://mempool.space/signet/tx/07f69511ffe4b11113b053b8fe1df87de598b6e5424231b7412497f4ebd066db)
    * [Wed, 26 Apr 2023 13:53:41 +0000](https://mempool.space/signet/tx/37f49975fc271f651832b991126bff09e10c56c322dac54186d44f2dae3aeaa8)
    * [Thu, 27 Apr 2023 14:47:01 +0000](https://mempool.space/signet/tx/042aad16f33f8830337884367559a5cc861af2640cf1838834d89151ce2e07a6)
    * [Thu, 27 Apr 2023 15:50:18 +0000](https://mempool.space/signet/tx/f54ed58f51f945e37d33849c810b4a18452971ca46ffbcaff5b4a1a6383a3a84)
    * [Fri, 28 Apr 2023 05:49:20 +0000](https://mempool.space/signet/tx/5e15e6d94403b1a12304ae03523d77379ff03848e20f5c3c4cc52835293b068c)
    * [Fri, 28 Apr 2023 17:13:04 +0000](https://mempool.space/signet/tx/ce2473df26455e2977201daddafaa56b3601e2645f9cd6180d362062935aa4c4)
    * [Sat, 29 Apr 2023 21:49:12 +0000](https://mempool.space/signet/tx/6a8c2b1ca1e7de8ed9c708aa7193e234ed5a4572f018efd5bbe0deceb048c3df)
    * [Fri, 05 May 2023 03:39:18 +0000](https://mempool.space/signet/tx/afd7d735f4f1fe2679234e298bbfd329bfdee31667ed9bc0145f6d8d75a318d0)
    * [Fri, 12 May 2023 17:31:59 +0000](https://mempool.space/signet/tx/c8513db745068bd12752cf8451721d13d31c5c9d8fc9459c337d3ddce830ba42)
    * [Fri, 12 May 2023 19:45:54 +0000](https://mempool.space/signet/tx/49f3af895f7457ef74ed67b3adc85b089a36356f8939c1a97a6f331ca43513cf)
    * [Wed, 17 May 2023 23:39:08 +0000](https://mempool.space/signet/tx/d96d8447276813bf95d4ddb398819fed27b63ecc5fa9abfe36dceb82cb012ecf)
    * [Thu, 18 May 2023 00:08:57 +0000](https://mempool.space/signet/tx/0a07cd737a5dbe757ca31af1acd9866dcc45b81defd0da9cd320dd7bb6a4fe42)
    * [Tue, 30 May 2023 22:09:58 +0000](https://mempool.space/signet/tx/e133127d6e87b1c07d99efdce2ab07b8b0dfd3ef5882785378df77c55dad5336)
    * [Wed, 31 May 2023 19:30:50 +0000](https://mempool.space/signet/tx/86890780f5d32882900e1fa0833e4135e984f7c790fc089dc36ef79e2273a1be)
    * [Wed, 31 May 2023 19:37:34 +0000](https://mempool.space/signet/tx/dd621a79c35e6bf55c7550d4d1efd63e7a97c33477fd0086277cbf176a79b61f)
    * [Wed, 31 May 2023 19:53:48 +0000](https://mempool.space/signet/tx/9f045b098840814e8599c69b2de6f5561bb360d1bfd18b13f3ac8395ac1d9a15)
    * [Tue, 06 Jun 2023 23:40:17 +0000](https://mempool.space/signet/tx/55db4cf8332cbde72d6ff871f1ac256efdaa85c5b580bb54be74b0175859b244)
    * [Wed, 07 Jun 2023 00:23:01 +0000](https://mempool.space/signet/tx/d33d1a57f2d5613cd6327559d2ea3672b053092b863ba97dd7fdb3ed40c3f767)
    * [Wed, 07 Jun 2023 17:55:42 +0000](https://mempool.space/signet/tx/e48b52f1d1056b101358a06e5d0f46e7d7c69eb95c1c1850e197d4de87de581a)
  * A couple of tests of ln-symmetry were conducted, which use two APO-based script templates; one is a simple APO-based spend using the IPK, ie `1 CHECKSIG`, that also includes an annex commitment that provides enough information to reconstruct the following tx script:
    * [Tue, 18 Apr 2023 13:20:37 +0000](https://mempool.space/signet/tx/e45d10d59a3f09774aea0615398944226c6aae5df6a77151153ed585ccaaa7cf)
    * [Thu, 11 Jan 2024 16:33:58 +0000](https://mempool.space/signet/tx/babe14326bf9ca2a180247e0e46970095403451c0b3b2b4c684aa353b419e9b7)
    * [Wed, 24 Jan 2024 02:32:52 +0000](https://mempool.space/signet/tx/f5bab4b4a5a6dd603b9fc9ed028f5b7a5d9fba3135d87e2c55f2ae22e17eb5bb)
  * The other is a CTV-like construction `<sig> <01;G> CHECKSIG` which uses the curve generator for the signature, as the IPK is a musig point, and isn't available for signature generation at the time the tx is constructed:
    * [Tue, 18 Apr 2023 14:48:56 +0000](https://mempool.space/signet/tx/9d63375a596ffd2be55624049580267ae07bf464e4d2fddc4043c7c3be79944c)
    * [Thu, 11 Jan 2024 17:34:56 +0000](https://mempool.space/signet/tx/21f854714debb0db46adc9d446a242e9a1d502dfe9761ac490ae8862fc0254b1)
    * [Wed, 24 Jan 2024 03:30:34 +0000](https://mempool.space/signet/tx/a20d2bf9e42402e5eac1a381bafd83961f9756a9b0d998a8a6bcaad605dd84e5)

That's the extent of APO-based usage on signet to date to the best of my knowledge. It's possible that further ln-symmetry tests will be possible soon, with all the progress in tx relay that's been made recently.

## CTV

 * The first pair of CTV txs have a "a committed set of outputs immediately, or anything after a timeout pattern", also requiring a signature for the specific path chosen: "`IF <height> CLTV DROP <key> CHECKSIG ELSE <hash> CTV DROP <key> CHECKSIG ENDIF`". These scripts are similar to the [simple-ctv-vault](https://github.com/jamesob/simple-ctv-vault/blob/7dd6c4ca25debb2140cdefb79b302c65d1b24937/main.py) unvault script, though that uses CSV rather than CLTV and does not require a signature on the CTV path.

    * [Wed, 25 Oct 2023 16:48:34 +0000](https://mempool.space/signet/tx/17959099cd0f3dbf47e9e233ce2eee60cda7815e1735c837b588c5753bdc23c1)
    * [Tue, 31 Oct 2023 23:26:32 +0000](https://mempool.space/signet/tx/d7fdb4f2a9f8a7a2e9b9c0b4858e47e32d984c72e6c53125dd3c748d82d8ddc5)

 * This was followed up by another pair that restricted the CTV path by also requiring a preimage reveal as well as a signature: "`IF <height> CLTV DROP <key> CHECKSIG> ELSE SHA256 <hash> EQUALVERIFY <hash> CTV DROP CHECKSIG ENDIF`"

    * [Wed, 01 Nov 2023 16:41:56 +0000](https://mempool.space/signet/tx/dc51a10ac54faa681253c9ecd853981ba56aefae232195ec18e79e1f6d850431)
    * [Wed, 01 Nov 2023 16:41:56 +0000](https://mempool.space/signet/tx/b68dc9f78a5a8c5c52704afdc4930636a9670297603bac5deaf3cfc421404fcb)

 * There are a couple of examples of bare ctv hash, ie the scriptPubKey was "`<hash> CTV`", with not scriptSig or witness required when spending. 

    * [Thu, 25 Jan 2024 14:14:50 +0000](https://mempool.space/signet/tx/85e1db10c47d222b83ed0b540acbe2568e65ad34f25968725d06d7e7a8c02b1b)
    * [Fri, 26 Jan 2024 18:19:27 +0000](https://mempool.space/signet/tx/12385e74c6ce476c8b3af56472234565454e2462c5d724508d7c8465f510b4f8) - "hello world"

 * There are a handful of p2wsh spends where the witness script was a simple "`<hash> CTV`". These are pretty generic, so could be anything

    * [Fri, 09 Feb 2024 03:00:51 +0000](https://mempool.space/signet/tx/224c5ba06e6fd577dcd2c712a3e737377ec73a1a82dc50d988dbed02acc56e73)
    * [Fri, 09 Feb 2024 14:37:40 +0000](https://mempool.space/signet/tx/5fc5cd19a341a946dc3fa1f37bd303d865e08d3e82fa89f75624d1b869a55287)
    * [Thu, 08 Aug 2024 00:52:10 +0000](https://mempool.space/signet/tx/931ac79f471e6c31eee598f00ad730ea60ac2ccab55121aaa1fc0e742a8ab008)
    * [Thu, 08 Aug 2024 01:22:10 +0000](https://mempool.space/signet/tx/98e5af9f34bf89bce0931a487619d97b2ce0d5ef22f06ac9ad5c355b3cfa41e8)

 * Also p2wsh "`<hash> CTV`", but with important messages in OP_RETURN outputs:
    * [Thu, 15 Feb 2024 13:49:10 +0000](https://mempool.space/signet/tx/d8e8be5565173a6d67928dd2ac70607e18d381507db3d2f1e1c2d04cc5e93bd6)
    * [Thu, 15 Feb 2024 14:00:01 +0000](https://mempool.space/signet/tx/c38efa32a48a4e8bd975807a6d8e42bbaeb668add88a693e476213404c9c4234)
    * [Thu, 15 Feb 2024 14:08:56 +0000](https://mempool.space/signet/tx/efa103bfe5fc929435f699f8c661c7ebdc3e1660522ed43c46e14b08d406043f)
    * [Thu, 15 Feb 2024 14:15:39 +0000](https://mempool.space/signet/tx/dc504a93ac5c4efe73e38ad1427a9f55559ac9d775e57602c83269aff4314a30)
    * [Thu, 15 Feb 2024 14:31:16 +0000](https://mempool.space/signet/tx/7bb3dbe59ad6e21a841dcc58741bca10dd9f15506fbf274018c845ac5074b1bc)
    * [Thu, 15 Feb 2024 14:55:23 +0000](https://mempool.space/signet/tx/c85dad8bbd7399f10412ef1e35a75be103a65a5111a0069988d55bf17474f0b3)

And that's it. I didn't see any indication of exploration of [kanzure's CTV-vaults](https://github.com/kanzure/python-vaults/blob/0d01750de642f325be4e84027a8f5a1a47b2c533/vaults/bip119_ctv.py) which uses OP_ROLL with OP_CTV, or the [simple-ctv-spacechain](https://github.com/nbd-wtf/simple-ctv-spacechain/blob/3c3c664c56aaf8069cd33bb3ba7b86b4d7e0e137/main.py), which uses bare CTV and creates a chain of CTV's ending in an OP_RETURN.

## CAT

There are substantially more transactions on chain (74k) that use it than either APO (1k) or CTV (16) due to the PoW faucet, which adds three spends per block, so it's a bit harder to analyse. Linking to addresses rather than txids, I think they sum up as:

 * PoW faucet:
   * [tb1pffsyra2t3nut94yvdae9evz3feg7tel843pfcv76vt5cwewavtesl3gsph](https://mempool.space/signet/address/tb1pffsyra2t3nut94yvdae9evz3feg7tel843pfcv76vt5cwewavtesl3gsph)
   * [tb1pf2v25yk7m8mv203pvjusmk2a8r6tu8p59nhvwux86ck3s3pp0nkqt30dvt](https://mempool.space/signet/address/tb1pf2v25yk7m8mv203pvjusmk2a8r6tu8p59nhvwux86ck3s3pp0nkqt30dvt)
   * [tb1pffsyra2t3nut94yvdae9evz3feg7tel843pfcv76vt5cwewavtesl3gsph](https://mempool.space/signet/address/tb1pffsyra2t3nut94yvdae9evz3feg7tel843pfcv76vt5cwewavtesl3gsph)
   * (there's some earlier addresses as well)
 * STARK verification:
   *  [tb1pnpxhs2syr62zkxgry0xv44zn84jg9dwg8jhp4gjefv9gh3ysmmssjxlyqy](https://mempool.space/signet/address/tb1pnpxhs2syr62zkxgry0xv44zn84jg9dwg8jhp4gjefv9gh3ysmmssjxlyqy) -- google points at this article: https://l2ivresearch.substack.com/p/recent-progress-on-bitcoin-stark
   * [tb1pn7zprl7ufprqht03k7erk2vtp78rqdj9vw4xs95qcee7v7ge0uks2f3u48](https://mempool.space/signet/address/tb1pn7zprl7ufprqht03k7erk2vtp78rqdj9vw4xs95qcee7v7ge0uks2f3u48) -- google links to [a tweet instead](https://x.com/StarkWareLtd/status/1843977366860865859)
   * [tb1p2jczsavv377s46epv9ry6uydy67fqew0ghdhxtz2xp5f56ghj5wqlexrvn](https://mempool.space/signet/address/tb1p2jczsavv377s46epv9ry6uydy67fqew0ghdhxtz2xp5f56ghj5wqlexrvn) gives an [earlier tweet](https://x.com/StarkWareLtd/status/1813619696538939455)
 * looks to be a CAT/CHECKSIG-based introspection, perhaps for vault or covenant behaviour?
   * [tb1p499mmrdadtyh47rw2220t9f2mteyszd352f22ajtx3mp5sdwqj3sqwdm0n](https://mempool.space/signet/address/tb1p499mmrdadtyh47rw2220t9f2mteyszd352f22ajtx3mp5sdwqj3sqwdm0n) 
   * [tb1pccmgyhanq2fpr2ru4pnm9xyknlmel9h6a8dlgaaz6gq5w4vw096qmj6xtw](https://mempool.space/signet/address/tb1pccmgyhanq2fpr2ru4pnm9xyknlmel9h6a8dlgaaz6gq5w4vw096qmj6xtw)
    * [tb1pcjzr4fq7gzcxmtnfdg8c293ej2mhqf2s0svj9eltjx3ze0f4v9pq00x3y4](https://mempool.space/signet/address/tb1pcjzr4fq7gzcxmtnfdg8c293ej2mhqf2s0svj9eltjx3ze0f4v9pq00x3y4)
    * [tb1pn8ekwyu4gw9lqa373e9tmq6xm9pn7gf7pgyr6v75rty5j7frz5ysjde8sn](https://mempool.space/signet/address/tb1pn8ekwyu4gw9lqa373e9tmq6xm9pn7gf7pgyr6v75rty5j7frz5ysjde8sn)
    * [tb1pdz3pakln88567wetz3tjjc3vgghhxqwdlaya2xwweuspt63eveusgm9rg9](https://mempool.space/signet/address/tb1pdz3pakln88567wetz3tjjc3vgghhxqwdlaya2xwweuspt63eveusgm9rg9)
    * [tb1p2mrsstfsz6vc8kraaf3nn0wdg3lmj60e6wq7sacfj07a0uxkpuqqwz2czy](https://mempool.space/signet/address/tb1p2mrsstfsz6vc8kraaf3nn0wdg3lmj60e6wq7sacfj07a0uxkpuqqwz2czy)
    * [tb1pzlhwsacu5ucf04wnws27elj2pyt0nthcv9c2njwu35jl6k0cxp9ssyuje3](https://mempool.space/signet/address/tb1pzlhwsacu5ucf04wnws27elj2pyt0nthcv9c2njwu35jl6k0cxp9ssyuje3)
    * [tb1pjhwjqw9gvtv0jvne8x7y85vzrp0yueu2xsfkyak222l5k564lg7qrqf0re](https://mempool.space/signet/address/tb1pjhwjqw9gvtv0jvne8x7y85vzrp0yueu2xsfkyak222l5k564lg7qrqf0re)
    * [tb1ppfdtz8eufft30gswdzfa39lq0adc6dg937n09krrwxa3cnlclfjqhlmsaa](https://mempool.space/signet/address/tb1ppfdtz8eufft30gswdzfa39lq0adc6dg937n09krrwxa3cnlclfjqhlmsaa)
    * [tb1pt3u4mgyg3xze6js2unn298tase5xmj5n45e9gexfmm4mecjq98lq90h2jt](https://mempool.space/signet/address/tb1pt3u4mgyg3xze6js2unn298tase5xmj5n45e9gexfmm4mecjq98lq90h2jt)
    * [tb1prcx703vp59q7sa7y0759a99vl7fjfmhuarnqww45u04s9vpmx8dsxm44jl](https://mempool.space/signet/address/tb1prcx703vp59q7sa7y0759a99vl7fjfmhuarnqww45u04s9vpmx8dsxm44jl)
    * [tb1pzz7xpljawrl6tg0h0w0shn98vfga5v4txn9qwy85a6rvvpx7datq7l4sp7](https://mempool.space/signet/address/tb1pzz7xpljawrl6tg0h0w0shn98vfga5v4txn9qwy85a6rvvpx7datq7l4sp7)
    * [tb1p8gqmhv6scsnxy40yxqxp8zjxshhm2mc5sgtkmyxae3yrsjkxd24qqjxf25](https://mempool.space/signet/address/tb1p8gqmhv6scsnxy40yxqxp8zjxshhm2mc5sgtkmyxae3yrsjkxd24qqjxf25)
    * [tb1pqkuq43vm5qe8naptpehe6qph7kh3mgtl4wr9mnk46pet0gk3sl4qjltsxl](https://mempool.space/signet/address/tb1pqkuq43vm5qe8naptpehe6qph7kh3mgtl4wr9mnk46pet0gk3sl4qjltsxl)

### Addendum 

I tracked these by hacking up a signet node that tries to validate block transactions with all of CTV/APO/CAT discouraged, and on failure retries with CTV and APO individually enabled, logging a message hinting at what features txs use. Could be interesting to turn that into an index that could track txs that use particular features a bit more reliably.

[details="patch"]
```diff
--- a/src/validation.cpp
+++ b/src/validation.cpp
@@ -2663,13 +2663,22 @@ bool Chainstate::ConnectBlock(const CBlock& block, BlockValidationState& state,
             std::vector<CScriptCheck> vChecks;
             bool fCacheResults = fJustCheck; /* Don't cache results if we're actually connecting blocks (still consult the cache, though) */
             TxValidationState tx_state;
-            if (fScriptChecks && !CheckInputScripts(tx, tx_state, view, flags, fCacheResults, fCacheResults, txsdata[i], m_chainman.m_validation_cache, parallel_script_checks ? &vChecks : nullptr)) {
-                // Any transaction validation failure in ConnectBlock is a block consensus failure
-                state.Invalid(BlockValidationResult::BLOCK_CONSENSUS,
-                              tx_state.GetRejectReason(), tx_state.GetDebugMessage());
-                LogError("ConnectBlock(): CheckInputScripts on %s failed with %s\n",
-                    tx.GetHash().ToString(), state.ToString());
-                return false;
+            auto xflags = flags | SCRIPT_VERIFY_DISCOURAGE_CHECK_TEMPLATE_VERIFY_HASH | SCRIPT_VERIFY_DISCOURAGE_ANYPREVOUT | SCRIPT_VERIFY_DISCOURAGE_OP_CAT;
+            if (fScriptChecks && !CheckInputScripts(tx, tx_state, view, xflags, false, false, txsdata[i], m_chainman.m_validation_cache, nullptr)) {
+                if (CheckInputScripts(tx, tx_state, view, (xflags ^ SCRIPT_VERIFY_DISCOURAGE_CHECK_TEMPLATE_VERIFY_HASH), false, false, txsdata[i], m_chainman.m_validation_cache, nullptr)) {
+                    LogInfo("CTV using transaction %s\n", tx.GetHash().ToString());
+                } else if (CheckInputScripts(tx, tx_state, view, (xflags ^ SCRIPT_VERIFY_DISCOURAGE_ANYPREVOUT), false, false, txsdata[i], m_chainman.m_validation_cache, nullptr)) {
+                    LogInfo("APO using transaction %s\n", tx.GetHash().ToString());
+                } else if (CheckInputScripts(tx, tx_state, view, flags, fCacheResults, fCacheResults, txsdata[i], m_chainman.m_validation_cache, nullptr)) {
+                    LogInfo("CTV/APO/CAT using transaction %s\n", tx.GetHash().ToString());
+                } else {
+                    // Any transaction validation failure in ConnectBlock is a block consensus failure
+                    state.Invalid(BlockValidationResult::BLOCK_CONSENSUS,
+                                  tx_state.GetRejectReason(), tx_state.GetDebugMessage());
+                    LogError("ConnectBlock(): CheckInputScripts on %s failed with %s\n",
+                        tx.GetHash().ToString(), state.ToString());
+                    return false;
+                }
             }
             control.Add(std::move(vChecks));
         }
```
[/details]

-------------------------

1440000bytes | 2024-11-15 21:28:39 UTC | #2

I can understand the importance of use cases and proof of concepts. Looking at number of transactions and adoption of prototypes on signet for a soft fork is beyond me. 

Especially when I see such things on signet:

![image|690x98](upload://w4tiivqn0jxFRBJx9wAgXA0FqIa.png)

[quote="ajtowns, post:1, topic:1257"]
or the [simple-ctv-spacechain ](https://github.com/nbd-wtf/simple-ctv-spacechain/blob/3c3c664c56aaf8069cd33bb3ba7b86b4d7e0e137/main.py), which uses bare CTV and creates a chain of CTV’s ending in an OP_RETURN.
[/quote]

It was tested on CTV signet and transaction is shared in the repository. I am sure some developers prefer to use regtest as well.

I have seen zero interest in average bitcoin users to try anything built using OP_CAT. Although some of them have tried using [CTV playground](https://ctv.ursus.camp/).
[quote="ajtowns, post:1, topic:1257"]
* [Thu, 08 Aug 2024 00:52:10 +0000](https://mempool.space/signet/tx/931ac79f471e6c31eee598f00ad730ea60ac2ccab55121aaa1fc0e742a8ab008)
* [Thu, 08 Aug 2024 01:22:10 +0000](https://mempool.space/signet/tx/98e5af9f34bf89bce0931a487619d97b2ce0d5ef22f06ac9ad5c355b3cfa41e8)
[/quote]

These 2 transactions are mine and were used to demo [joinpool](https://gist.github.com/harding/a30864d0315a0cebd7de3732f5bd88f0).

-------------------------

AdamISZ | 2024-11-16 23:13:48 UTC | #3

The CTV examples in Oct/Nov 2023 were me and a friend demo testing [pathcoin](https://github.com/AdamISZ/pathcoin-poc/tree/master) (see fidelitybonds.py for the scripts).

(at least .. I'm 99% sure .. I certainly did it around then just before the San Salvador conference but didn't keep a record, and the scripts match exactly)

Perhaps not the most practically realistic use-case but, interesting experiment :) thanks for setting it up to be usable on signet!

-------------------------

vostrnad | 2024-11-21 20:04:01 UTC | #4

Your investigation inspired me to create an explorer for Bitcoin Inquisition transactions. It should hopefully make similar analysis easier in the future, let me know if there's something I can improve.

https://inquisition.observer

-------------------------

JeremyRubin | 2024-11-27 21:07:42 UTC | #8

I don't really think these are valid metrics, given they don't include ctv-signet which predates inquisition.

I ran a count of CTV activity on ctv-signet. I did this by scanning for NOP4.

- 1558 segwit v0 spends
- 52 legacy spends
- 14 taproot spends

 This was last used in 2022, though i keep it running still.

There were only 52 spends of legacy, although there were around 24 additional creations not counted in the above figure. This suggests some amount of other taproot or segwit outputs would also be CTV users. Especially taproot -- the whole point being you can't tell if it's just a key!



Overall I think signet is just... not that useful, in general. There's no track record of these vanity metrics having anything to do with consensus. Most devs just do stuff on regtest at the end of the day.

-------------------------

JeremyRubin | 2024-11-27 21:11:41 UTC | #9

Your methodology also AFAIU won't catch IF blah ELSE <H> CTV ENDIF scripts, if they execute the blah.

-------------------------

ajtowns | 2024-12-09 19:28:51 UTC | #10

[quote="JeremyRubin, post:8, topic:1257"]
I don’t really think these are valid metrics, given they don’t include ctv-signet which predates inquisition.
[/quote]

As you know, I already did a similar [review of transactions](https://gnusha.org/pi/bitcoindev/20220420023107.GA6061@erisian.com.au/) on ctv-signet in 2022. There wasn't anything there that made it seem like it would be worth the time to do another review.

[quote="JeremyRubin, post:8, topic:1257"]
Overall I think signet is just… not that useful, in general. There’s no track record of these vanity metrics having anything to do with consensus.
[/quote]

The point isn't to find another instance of "number go up", it's to see if there's been any testing of the code, or any experimentation to see if the hypothesised use cases are implementable in the real world.

For things that are interesting to people, you do see real experimentation on signet. For example, there's about 60k [inscriptions](https://signet.ordinals.com/inscription/e10d3076bc1af7cbb194053ca744830d3ceeca1e71505ab71324c2ba4d2c7843i0), about 1000 [runes](https://signet.ordinals.com/rune/THESE%E2%80%A2WILL%E2%80%A2BE%E2%80%A2WORTHLESS), and babylon's test runs ["filled" quite a few blocks](https://mempool.space/signet/block/000000167ac3b24d5ea8c911901cf4eab87c345240257dfc94528897a3e9d596), leading to an apparently successful (and comparatively less disruptive) [mainnet launch](https://x.com/mononautical/status/1826604180251050388) (and, TIL, [followup](https://x.com/babylonlabs_io/status/1845961016158704132)).

[quote="JeremyRubin, post:9, topic:1257, full:true"]
Your methodology also AFAIU won’t catch IF blah ELSE CTV ENDIF scripts, if they execute the blah.
[/quote]

It similarly won't catch CTV invocations that aren't executed because they're in unrevealed taproot branches, or in presigned child transactions that are never broadcast. The point of having a test environment is that you can test the rare cases to make sure they behave correctly just in case the rare case happens in real life some day; and if you do that, there will be example transactions where those paths are exercised. Nobody's saying you have to do that testing on signet, of course.

-------------------------

fiatjaf | 2024-12-10 02:18:08 UTC | #11

[quote="ajtowns, post:1, topic:1257"]
APO as an overridable CTV in the second quarter of 2023, with a script of the form `<sig> 1 OP_CHECKSIG`
[/quote]

These were testing the https://github.com/nbd-wtf/soma, a spacechain toy implementation for transferring NFTs (the simplest example of blockchain I could come up with). These transactions correspond to blind merged-mining events (paid out of band with signet lightning) that would each yield a different spacechain block, as you can see they have an OP_RETURN with the hash of corresponding spacechain block.

The broken key path free spending part was a stupid vulnerability due to an overlooked detail. Please ignore, it was just a demo.

-------------------------

ajtowns | 2024-12-11 11:49:09 UTC | #12

Oh wow; up until now I've only seen spacechain demos that didn't actually implement the spacechain part. Is the spacechain data archived somewhere?

Not using a NUMS point probably makes sense for an experiment -- lets you use a key path spend to hardfork the spacechain when updating the code, while potentially retaining history. Of course, that assumes you're using a somewhat secure key; looks like you used G so the funds could be stolen at any point (edit: or more importantly, the spacechain could be stopped dead) by anyone who can figure out what signature the latest scriptPubKey contains.

(I was expecting a non-well-known pubkey and a published list of signatures, which would have been secure-ish and also allowed permissionless mining)

If I'm reading the code right, you limited the spacechain to only doing 50 blocks (Publish.scala, `bmm.precompute(0, 50)`), which looks like it about matches the number of txs listed above, which presumably means the only script path spend you can do now is the final tx, paying to a 0sat "fim" OP_RETURN and reclaiming the funds (or using them for fees, or using them to launch a brand new spacechain)?

-------------------------

fiatjaf | 2024-12-11 21:56:21 UTC | #13

This was a long time ago so I don't remember exactly, but it looks like 50 was just the number of precomputed transactions that was stored initially by the miner (and presumably it would run the computation again and store more transactions when it was restarted). In fact the spacechain was precomputed to last 100 years (`BMM.scala`, line 11).

So I wasn't really expecting to get to the last block of this, I was probably planning on expanding the maximum period to 1000 years in a "production" scenario, but it didn't make sense to calculate so much given that this was just a demo.

I ran the demo miners and spacechain nodes for months and tried to get people to use it, but didn't get much attention, maybe ~10 people played with it at the time. Then I got disillusioned and decided to shut it down, I thought about keeping a record of the blockchain data, but it was too depressing and the blockchain didn't really have any value, it probably makes more sense to start a new spacechain now and test again if anyone is interested.

All the blockchain did was to generate NFTs (without any metadata whatsoever except for a serial number) and transferred them between bip340 pubkey accounts. It had two types of transactions, `mint` and `transfer`.

The mining process was interesting: to publish a transaction on the spacechain a user would contact any of the (or all) miners and send them the transaction, the miner would make a Lightning invoice and the user would pay. Miners would hold the Lightning payment in-flight while they gathered more transactions from other spacechain users and then try to use the funds they got via Lightning to pay for an onchain BMM transaction, always performing an RBF when a new user tx arrived -- once they succeeded they would resolve the Lightning payments and release the spacechain block to other spacechain nodes. If they didn't succeed in 10 blocks or if they saw one of the transactions they were holding mined in another spacechain block then they would cancel that transaction specifically and fail the Lightning payment. The user "wallet" was a web interface for keeping track of user assets, pending transactions and it also contacted miners directly via the CLN websocket "commando" interface. It's weird, I should at least have recorded a screencast of the process, but I can't find any.

For the BMM covenant I should have used a provably invalid key (is that what a NUMS mean?), but I think I got too distracted while coding the script path and all the APO stuff (I didn't have any prior experience with programming Bitcoin script and much less with Taproot and APO and PSBT) and didn't pay attention to that obvious key path flaw.

-------------------------

fiatjaf | 2024-12-11 22:00:37 UTC | #14

Oh, by the way, the BMM transactions should all be linked in a chain, every new one always spending the "canonical" `1234sat` output from the previous -- when you see that chain broken that is probably because I was testing myself, then abandoning the chain and starting a new one, before releasing it to the public.

-------------------------

ajtowns | 2024-12-12 05:57:47 UTC | #15

[quote="fiatjaf, post:13, topic:1257"]
t looks like 50 was just the number of precomputed transactions that was stored initially by the miner (and presumably it would run the computation again and store more transactions when it was restarted).
[/quote]

Ah, that makes more sense. Could cache every 4000th hash, and the next 4000 hashes, so you don't have to recalculate all 5M hashes everytime you run out of cached hashes. Would only be ~20k hashes (640kB) to cache even for the 1000 year case then, and you could save them to disk or precalculate at compile-time.

[quote="fiatjaf, post:13, topic:1257"]
I ran the demo miners and spacechain nodes for months and tried to get people to use it, but didn’t get much attention, maybe ~10 people played with it at the time.
[/quote]

Yeah, gotta have a marketing plan to build up lots of drama/excitement if you want attention in the NFT/memecoin space. As far as I'm aware, it never even got mentioned in any of the tech spaces I follow (eg bitcoin-dev, optech). OTOH, I expect I would have ignored anything I did see though, since that was around the time of the [inv-to-send issue](https://bitcoincore.org/en/2024/10/08/disclose-large-inv-to-send/). Shame.

Two things that would probably be interesting, but also perhaps much harder than what you did: a spacechain that includes actual new programming features (simplicity? EVM like RSK? confidential txs? multiple fungible assets? utreexo commitments?), and/or a spacechain with a currency that's pegged against sBTC in some way (even a boring trusted third-party way like RKS's or liquid's multisig federation or whatever WBTC does), so the chain could theoretically help with "overflow txs" during fee spikes. But those are probably hard to do, and it's better to be simple but implemented than fancy but imaginary.

Maybe utreexo commitments could be interesting for NFTs -- you could have short proofs that you "own" your profile picture NFT; though having the proof potentially change every block might be too awkward in practice... I guess you'd need some way to link the things you mint to some other data though; atm I think soma just lets you create a token that's tradeable but has no link to anything else? [Reverse commitments](https://gnusha.org/pi/bitcoindev/Y9t%2FNcgRzv1w%2F%2FFx@erisian.com.au/), where the data links to the token (instead of the token containing/committing to the data) might work fine with soma as-is though, though then you'd also need to add an `O(n_transfers)`-size proof that your utxo matches the token... a spacechain with a chia-esque coin-set model instead of a utxo model could let you avoid that, though...

How reliable spacechains are in adversarial environments is a big question to me; will people actually fully validate spacechains, or is there a risk that you can just pay to mine lots of "invalid" spacechain blocks and that will be accepted anyway? How easy/disruptive are reorgs -- if you can reorg some blocks, doublespend one transaction, but nobody else loses out (because there's no coinbase fees), will anyone try to prevent the reorg, or is the person being doublespent on their own? Could be fun to explore that sort of thing with a real implementation.

[quote="fiatjaf, post:13, topic:1257"]
I ran the demo miners and spacechain nodes for months and tried to get people to use it, but didn’t get much attention, maybe ~10 people played with it at the time.
[/quote]

That's probably similar to the actual usage of the powcoins script, fwiw. Also comparable to bitcoin itself, really; ignoring coinbase txs, there were only 93 txs in its first 3 months (10570 blocks). Sometimes you just have to give these things time.

[quote="fiatjaf, post:13, topic:1257"]
For the BMM covenant I should have used a provably invalid key (is that what a NUMS mean?),
[/quote]

Yeah, "Nothing Up My Sleeve" -- ie, "it's completely/cryptographically infeasible for me to possibly have a private key/preimage/etc corresponding to this".

[quote="fiatjaf, post:13, topic:1257"]
Miners would hold the Lightning payment in-flight while they gathered more transactions from other spacechain users and then try to use the funds they got via Lightning to pay for an onchain BMM transaction,
[/quote]

Sounds a bit complicated for an experiment tbh; though I see you setup a thing to auto pay people's invoices for them anyway. I think I can see how you could plausibly build up a reputation as a trustworthy miner for people to be willing to go through that process with non-fake coins. Neat.

[quote="fiatjaf, post:13, topic:1257"]
I thought about keeping a record of the blockchain data, but it was too depressing and the blockchain didn’t really have any value, it probably makes more sense to start a new spacechain now and test again if anyone is interested.
[/quote]

If you do, I'd suggest trying to make something that you can just leave running unattended for ages, and ideally that automatically includes some spacechain content occassionally, even if it's trivial. Automatically dumping the block data into a github repo once a week might be workable? If the explorer could make static html pages that could also go into the github repo that would maybe be nice, as a low overhead way of making it still explorable even if you don't want to keep actually maintaining/running the servers?

-------------------------

1440000bytes | 2024-12-14 22:17:24 UTC | #16

It doesn't even detect transactions that create bare CTV unspent outputs with OP_NOP4.

Example: https://mempool.space/signet/tx/ab953773f8f8306567bf48357d8ca5a179909c12b2bc64ae2ef80f0850708920

-------------------------

ajtowns | 2024-12-16 11:15:42 UTC | #17

As I already replied to Jeremy, it detects spends, not output creation.

-------------------------

1440000bytes | 2024-12-16 12:42:36 UTC | #18

Fair enough. This looks misleading though.

![image|690x179](upload://faVUI8oLrExmYI3gfFSlUMbClsI.png)

Cc: @vostrnad

-------------------------

vostrnad | 2024-12-16 13:03:52 UTC | #19

The website also only detects spends. I guess I could argue that if you never spend an output you didn't actually "use" the opcode. Nevertheless, you're more than welcome to suggest a different wording in a pull request.

-------------------------

1440000bytes | 2024-12-16 17:04:12 UTC | #20

Pull request: https://github.com/vostrnad/inquisition.observer/pull/1

However, it would be great if transactions that create bare CTV outputs were included in these stats. It is possible that someone was unable to spend a CTV output after creation during testing.

-------------------------

1440000bytes | 2025-01-03 00:47:17 UTC | #21

It's unclear how much these stats have helped developers who are genuinely testing soft forks. However, they seem to have convinced a number of OP_CAT supporters that signet bots can be used to justify activation on mainnet. Here are some examples from rationales on the Bitcoin Wiki:

> Despite being activated just six months ago, OP_CAT has already outpaced other proposals like BIP 118 (SIGHASH_ANYPREVOUT, APO) and BIP 119 (OP_CHECKTEMPLATEVERIFY, CTV), which have been on signet for nearly two years. With 10 times more transactions on signet, OP_CAT’s usage dwarfs APO’s and CTV’s, showcasing its exponential adoption (source).

[https://gist.github.com/catOnStack/3e9aea351346cd316e6e0dcaa195da2d](https://gist.github.com/catOnStack/3e9aea351346cd316e6e0dcaa195da2d)

> While BIP 118 (SIGHASH_ANYPREVOUT, APO) and BIP 119 (OP_CHECKTEMPLATEVERIFY, CTV) have been active on signet for nearly two years, OP_CAT, activated six months ago, has already seen significantly more usage. With 95101 transactions compared to APO’s 1093 and CTV’s 261, OP_CAT’s adoption is exponentially higher

[https://gist.github.com/rousjiamo/309d1ddfcfa89e061bbc397838480bbb](https://gist.github.com/rousjiamo/309d1ddfcfa89e061bbc397838480bbb)

> BIP 118 (SIGHASH_ANYPREVOUT, APO) and BIP 119 (OP_CHECKTEMPLATEVERIFY, CTV) have been enabled on signet for nearly two years, since late 2022. More recently, BIP 347 (OP_CAT) was activated, having been available for six months. Despite of being active for only 1/4 of the time, there are significantly more on-chain transactions utilizing OP_CAT (74K) compared to APO (1K) or CTV (16), translating to ***300X*** and ***18500X*** more usage in a given period.

[https://scryptplatform.medium.com/evaluating-bitcoin-upgrade-proposals-2df82bfe48df](https://scryptplatform.medium.com/evaluating-bitcoin-upgrade-proposals-2df82bfe48df)

-------------------------

ajtowns | 2025-01-03 07:34:05 UTC | #22

Not sure what good you think will come of promoting rationales you think are bad. It's already easy to do a sybil attack against a more-or-less permissionless wiki page; if you're going to do anything with it, it's better to promote whatever signal there is to be found there rather than the noise.

-------------------------

1440000bytes | 2025-01-03 08:27:55 UTC | #23

I think its far easier to create fake stats on signet than fake evaluations on wiki. 

https://x.com/stevenroose3/status/1862231147851243760

-------------------------

ajtowns | 2025-07-03 08:10:02 UTC | #24

[quote="ajtowns, post:1, topic:1257"]
the [simple-ctv-spacechain](https://github.com/nbd-wtf/simple-ctv-spacechain/blob/3c3c664c56aaf8069cd33bb3ba7b86b4d7e0e137/main.py), which uses bare CTV and creates a chain of CTV’s ending in an OP_RETURN.
[/quote]

There were a few of these in Nov/Dec, with shortish chains starting at [[1]](https://mempool.space/signet/tx/4436231f5003fe7943c4394d5c04a2b1d51fea76bb8f63f2b611bd834c0573ed) [[2]](https://mempool.space/signet/tx/f4e627b1d97264fad4905270f9c5b9962c0f9b1aebb818fb058043d49786caa6) [[3]](https://mempool.space/signet/tx/dff73d8afd0c9abdc6b7fd00e4e70371f16c295f9ca1ccd62d038d19fb9f2b1f), and some slightly longer chains at [[4]](https://mempool.space/signet/tx/68f45bca9e52c76497060cbc33a8c9f33c74f578de4c40eb3eea8396e1efffb2) [[5*]](https://mempool.space/signet/tx/d541ccbce23f958cb475d1878a7333300cddb5d92da7b1103b0d9439da588b04) [[6*]](https://mempool.space/signet/tx/141d852317d3c45e34f9454dd343636f4e1b9a3ad7a6f1a1c8881d2d065b830e) [[7]](https://mempool.space/signet/tx/e2d5f9852e9fce69bbaafeb3df1a9e6975f36f4b9eb25f13bb0c2ac2ad4d3d36) [[8]](https://mempool.space/signet/tx/c1d41002d36321c381fcc0124160f93ddca0525cbb4680a1f673a2c76a3a6ce2) [[9]](https://mempool.space/signet/tx/54e38782931bc7cf2539099b892313dae078ec64953ac2edc12146dc840a44b5). A couple of those chains weren't completed, but presumably could be by reusing information from previous spends of the same scriptPubKey.

There were a few txs in Dec (eg [[10]](https://mempool.space/signet/tx/d6c0a1a6ef05566c20bc17377e03dda555aed21414b363110b1a1d069c38938e)) using the pattern `<p> CHECKSIGVERIFY <t> CTV DROP`, and redundantly including the CTV hash in the witness as the final "true" value for the stack.

April saw a single tx ([[11]](https://mempool.space/signet/tx/8dba98e0773d783620311c0ae4d636ccd4f95dbdddef57fa93758337f4a5d741)) using the pattern `IF 1 CSV DROP <t1> CTV ELSE <t2> CTV ENDIF`.

There were also a variety of plain `<t> CTV` or `<t> CTV DROP` txs between November and April that didn't appear to include any additional logic, eg [[12]](https://mempool.space/signet/tx/17a4dfa811e0a44ab7370b0e4c10127816d4d65eae78b36b0102bc6d80895be0) [[13]](https://mempool.space/signet/tx/8b0c09b92387ddbbea7ae2a8bca24e48a28551be4025c50ab74844ac8001077c) [[14]](https://mempool.space/signet/tx/5fa5342f5be158b5a7190ec57f8f1b53c0dac6cd03a4fc7e0f778686832f420a) [[15]](https://mempool.space/signet/tx/e95e03fca749db418f97aa56f22d737822fd57e0c20d3573a684e0f606ed5421) [[16]](https://mempool.space/signet/tx/d508d1f6873abe6031a2ff620c7830d72e103a1593353b88ee15896f81eef827).

-------------------------

