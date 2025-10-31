# Anecdotal kode (code) for humans

jsarenik | 2025-05-12 07:06:45 UTC | #1

Following is the latest version of best block hash shortening with a practical application for naming time-sensitive entries in the end.

# Motivation

To write less and still grasp it all.

# Demo

See the following block text-only representation with its height and hash including the "shortened kode" `sk` and the new "anecdotal kode" `ak`. Available live at 
https://anyone.eu.org/niceblack.txt although from now only with `ak` code.

Both codes have the same octet in the end. Explained in https://anyone.eu.org/skm.txt

An example which includes both follows:

```
     _|_|      _|_|    _|_|_|    
   _|    _|  _|    _|        _|  
     _|_|      _|_|      _|_|    
   _|    _|  _|    _|        _|  
     _|_|      _|_|    _|_|_|    

     _|  _|      _|  _|_|_|_|  
     _|  _|    _|_|  _|        
     _|_|_|_|    _|  _|_|_|    
         _|      _|        _|  
         _|      _|  _|_|_|    

  ,---   .123 4567 89ab cdef   ---,
  | ..   .... .... .... ....   .f |
  | 1.   ...1 64ea 5622 1a9b   1f |
  | 2.   6512 2cd3 dcd9 75fa   2f |
  | 3.   aac. 1.47 3ea4 c883   3f |
  '===   ==== ==== ==== ====   ==='
   sk:   c883 94
   ak:   841. 94
```

# What happens

All of the "doublets" (double octets) shown above are XORed to get `ak`.

See https://anyone.eu.org/bitcoin.txt and look for "shortform" there. It was the first thought of mine, some years ago.

# Practical use

To describe SHA256 sum of any data one would make an `ak` of the hash, XOR it with `ak` of a Bitcoin block and publish the result widely including the block height. The string is much shorter than the original 32 octets but allows checking with full 32-byte hashes (one of the block and the other of input data) whenever needed and promotes running Bitcoin nodes in order to locally query for the full block hash. 

## Old example using `sk`

See http://anyone.eu.org/ws and https://anyone.eu.org/ws/883417-c1a4a4.txt for a current example.

## New example using `ak`

See https://anyone.eu.org/as/ and https://anyone.eu.org/as/883438-53c655.txt

# fin

I will be happy for feedback since I have never heard any complaint other than `16777215` (`0xFFFFFF`) is a too small number.

-------------------------

jsarenik | 2025-05-13 15:10:29 UTC | #2

In extension to this anecdotal thought I would like to say that a witness of any SHA256 hash consisting of two octets (two bytes, 0xFFFF maximum) can be easily put into the locktime of a transaction. This does not influence the raw transaction size.

Example:

```
BITCOIN BITCOIN BITCOIN BITCOIN

896366 ak: e..5 84
Any other plaintext follows

BITCOIN BITCOIN BITCOIN BITCOIN
```

The plaintext could be easily signified - see https://man.openbsd.org/signify.1

The header and footer lines ensure easy optical check that the plaintext is not a result of a collision.

The SHA256 of above preformatted plaintext obtained on the command line with `sha256sum` is 16a49ef7d7f3fab3276dc1f46205baa805ef57d6e0e8069aebd65eedaf554517 which humanized and anectotalized is:

```
         #       #       #
        # #     # #     # #
       #   #   #   #   #   #
      #     # #     # #     #
      ####### ####### #######
      #     # #     # #     #

  ,----- .123 4567 89ab cdef -----,
  |                               |
  | ..   16a4 9ef7 d7f3 fab3   .f |
  | 1.   276d c1f4 62.5 baa8   1f |
  | 2.   .5ef 57d6 e.e8 .69a   2f |
  | 3.   ebd6 5eed af55 4517   3f |
  '===   ==== ==== ==== ====   ==='
   ak:   7.15 c.
```

Here is a Testnet4 example transaction (note its Locktime; 0x7015=28693)

https://mempool.space/testnet4/tx/6a955b1e60ab64473f38b6dbc8355e7eb42c32bcf0c45e8d4c14f6a490167bab?mode=details

-------------------------

