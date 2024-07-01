# B’SST: Bitcoin-like Script Symbolic Tracer v0.1.3 released

dgpv | 2024-06-27 07:58:11 UTC | #1

Improved display of execution paths, and a bunch of bugfixes

URL: https://github.com/dgpv/bsst/

Release notes: https://github.com/dgpv/bsst/blob/9e1f2e39a5c0475598c33bbdc8cd0b690f5cc84f/release-notes.md

The descriptions of execution paths now show actual conditions that designate the paths, instead of opcodes with args:

    Instead of

    ```
    IF wit0 @ 0:L1 : True
    NOTIF wit1 @ 1:L1 : False
    IFDUP wit2 @ 2:L1 : True
    -------------------------
    ```

    Paths will be shown as


    ```
    When wit0 :: [IF @ 0:L1]
     And not wit1 :: [NOTIF @ 1:L1]
     And BOOL(wit2) :: [IFDUP @ 2:L1]
    ---------------------------------
    ```

-------------------------

