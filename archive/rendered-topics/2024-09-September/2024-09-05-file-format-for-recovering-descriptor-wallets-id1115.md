# File Format for Recovering Descriptor Wallets

rustaceanrob | 2024-09-05 18:13:42 UTC | #1

Hi folks. I am posting a draft proposal to attempt to improve the wallet recovery experience for users or their heirs.

TLDR: I am worried an heir will have to copy-paste descriptors from a `txt` or `json` into future wallet software. Ideally, heirs may simply drag-and-drop a file into future wallet software and be assured all relevant data that encapsulates a wallet is present. This file should also store metadata, like where signers may be located, or a height to start searching for outputs.

I know that reading a file format specification may not be very exciting, but I am hoping to gather feedback and ultimately improve recovery for the community. The specification is outlined below, and I also have a [reference implementation in Rust](https://github.com/rustaceanrob/wdef) to experiment with. Any feedback is greatly appreciated!

### Motivation

The purpose of a standarized file format is to decrease the chance of loss of funds in the case of wallet recovery or inheritance. BIP32 seed phrases are not sufficient information to recover funds if multiple signers are used, a non-standard derivation path is used, or if the recoverer does not know what to do with such seed phrase. Moreover, seed phrases must be entered into a device to recover wallet public keys. Output descriptors alleviate these issues, however the process of storing, exporting, and sharing descriptors is currently determined by users. A unified file format allows for "one-click" backups and recoveries for descriptor-based bitcoin wallets, transferring the responsibilities of handling descriptors from users to software.

Output descriptors are often exported, shared, and stored in `.txt` files, but other formats such as JSON are also used. This discrepancy causes developer burden, and varying formats cause uncertainty of support for parsing such data in future wallet software. If descriptor parsing from a `txt` or `json` file fails, it is up to the heir or recoverer to resolve the problem. A unifed file format for importing and exporting descriptors not only decreases developer burden, but allows for a higher degree of certainty that an heir will not have to manually "copy and paste" descriptors into future software. Files are portable, duplicable, and easily parsed by most devices. Files may also be encrypted and safely shared with attorneys, business partners, or other semi-trusted entities.

Futher still, metadata about the descriptors, such as the name of the wallet, a description of its uses, where to start scanning for ouputs, and additional information to recover signers are either encoded as a comment in the `.txt`, introducing difficulty for software to parse reliably, or fields in a JSON, introducing forward-compatiblity concerns for future wallet software. These fields are easily encoded and decoded with a standard format.

### Definitions

_descriptor_, shorthand for "output descriptor."

`||` denotes the concatenation of two elements.

`[]bytes` represents a variable array of bytes.

`[N]bytes` represents an array of `N` bytes.

`Record` is an entry in an import.

`Import` an entry in the file that describes metadata and associated descriptors.


### Specification

#### `Record` 

Information is stored in a WDEF file in the form of `Record`s. A `Record` is a tuple of a `Type`, `Length`, and `Value`, followed by a 4 byte checksum. Every WDEF file is prefixed with a single byte representing the number of records in the file. A record type is represented as a byte, with possible types listed below:

| Record Type (`Type`) | Value (`u8`) | Description                        |
| ------------------- | ---------- | ---------------------------------- |
| Name | 0x00 | The name of the wallet in the file |
| Description | 0x01 | Summary of this wallet's use(s) |
| Info | 0x02 | Any additional information to recover funds |
| RecoveryHeight | 0x03 | Height in the blockchain this wallet began to receive and send payments |
| ExternalDescriptor | 0x04 | A descriptor that is used to receive bitcoins. Encodes public keys and cannot spend bitcoins |
| InternalDescriptor | 0x05 | A descriptor that is used to generate change outputs when spending bitcoins. Encodes public keys and cannot spend bitcoins |
| MultipathDescriptor | 0x06 | A descriptor that encodes multiple paths to unique descriptors. |

A `Length` is a 16-bit number represented as bytes in _little endian_. The length represents the number of bytes in the value encoding that follows.

`Name`, `Description`, `Info`, `ExternalDescriptor`, `InternalDescriptor`, and `MultipathDescriptor` are all represented as strings and encoded as the UTF-8 byte array
for such a string representation. `RecoveryHeight`s are represented as a 4 byte _little endian_ array representation.

The checksum for a `Record` is calculated by `SHA256( Type || Value )` and taking the first four bytes of the resulting hash.

A `Record` is completely defined as:
- `Type`: `[1]bytes` ID
- `Length`: `[2]bytes` _little endian_ value representing the length of the next field
- `Value`: `[]bytes` variable length contents representing the record type, often a UTF-8 encoded string
- `Checksum`: `[4]bytes` a commitment to the record type and record value

#### `Import`

An `Import` is simply a structured list of `Record`s. An import contains a single byte that represents the count of `Record`s in the file, followed by the `Record`s. All `Import`s must adhere to these requirements, or the file is invalid:

- A single `Name` is required.
- Any `Description` must be unique, but is not required.
- Any number of `Info` may be included.
- A `RecoveryHeight` may be provided, but is not required. The `RecoveryHeight` must be unique.
- Any number of `ExternalDescriptor`, `InternalDescriptor`, or `MultipathDescriptor` may be provided.
- At least one `ExternalDescriptor` or `MultipathDescriptor` must exist. 

An `Import` is encoded as:
- `Length`: `[1]bytes` _little endian_ integer representing the `Record` count
- `Record`s: `[]bytes` the entries of the file.

#### File Prefix

Every WDEF file is prefixed with seven bytes of data that identifies the file: `0x00, 0x00, 0x00, 0x57, 0x44, 0x45, 0x46`. This data is followed by the protocol version, which is set to `0x00` for version one.

#### Complete Structure

- File identifier: `0x00, 0x00, 0x00, 0x57, 0x44, 0x45, 0x46`
- `Version`: `[1]bytes`
- `Length`: `[1]bytes`
- `Record`s: `[]bytes`

#### Encoding Files

First, the seven bytes of the file identifier and single verison byte are written to the file.

Next, the number of records to be recorded in the file should be determined, and the byte representing the length is added to the serialization buffer. If the number of records cannot fit into an 8-bit unsigned integer, encoding fails.

Next, each record is composed as `Type || Length || Value || Checksum`, and each array of bytes are concatinated together. Encoding fails if the `Length` cannot be represented as a 16-bit unsigned integer.

If any of the `Import` requirements are violated, encoding fails.

#### Decoding Files

Read the first seven bytes of the file, failing the decoding if the file identifier does not match. Read the next byte and interpret it as the version. For versions greater than those supported by the software, decoding fails. All WDEF decoders must implement `0x00`.

The next byte is read from the file and interpreted as the number of records. For each record in the record count:

1. Read and interpret the first byte as the `Type`. Decoding fails for unrecognized `Type` values.

2. Read the next two bytes and interpret them as an 16-bit unsigned integer, denoted `L`

3. Read the next `L` bytes and parse into the desired representation. Decoding fails if the bytes cannot be parsed.

4. Calculate the checksum using the `Value` and `Type`

5. Read the next 4 bytes and fail if the calculated and presented checksum do not match.

6. For descriptors, parse the computed string and attempt to cast it to a descriptor that encodes public keys. Decoding fails for if the provided string cannot be cast to a public key descriptor.

After all the `Record`s have been collected, evalutate if the collection follows the rules for an `Import`. Decoding fails if a rule is violated.

Files adhering to this standard should be postfixed with the `.wdef` extension.

### Rationale

Descriptor `Value`s are represented strings for the following reasons:

1. Bitcoin Core supports parsing strings as descriptors within `bitcoin-cli`.

2. UTF-8 encodings are an open standard with forwards compatability for new `Type`s.

3. Most encoding logic may be shared for each `Type`.

The checksum is added to ensure that invalid `Record` encodings are discovered during decoding and file integrity is guaranteed when successfully parsing a WDEF file.

-------------------------

valuedmammal | 2024-09-24 15:36:57 UTC | #2

Maybe have a record type for "last known derivation index"?

-------------------------

