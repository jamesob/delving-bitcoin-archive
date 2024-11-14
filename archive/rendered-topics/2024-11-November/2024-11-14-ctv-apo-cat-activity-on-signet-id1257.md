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
