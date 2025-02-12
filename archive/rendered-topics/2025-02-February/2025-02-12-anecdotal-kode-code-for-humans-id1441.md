# Anecdotal kode (code) for humans

jsarenik | 2025-02-12 14:55:49 UTC | #1

Following is the latest version of best block hash shortening with a practical application for naming time-sensitive entries in the end.

# Motivation

To write less and still grasp it all.

# Demo

See the following block text-only block with its height and hash including the "shortened kode" `sk` and the new "anecdotal kode" `ak`. Available live at 
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

