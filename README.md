
# delvingbitcoin.org archive

This repository contains the archived contents of https://delvingbitcoin.org. It uses
[discourse-archive](https://github.com/jamesob/discourse-archive).

## Contents

Rendered topic threads are in [`archive/rendered-topics`](./archive/rendered-topics).
Raw post JSON, which might be useful for search indexing, can be found in
[`archive/posts`](archive/posts).

## Running it yourself

```
$ pip install discourse-archive
$ discourse-archive -u https://delvingbitcoin.org -t archive
```
