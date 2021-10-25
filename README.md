# XRPL Catalogue Proposal

NOTE: README IS WIP

## Introduction
The XRP ledger produces a large amount of immutable historical data which currently must be maintained across a network of [Full History](https://xrpl.org/configure-full-history.html) nodes or it will be forever lost.

Note that some optimisation of history sharing and storage has already been achieved through [History Sharding](https://xrpl.org/history-sharding.html). However this proposal will make the case that the robustness and searchability of network history would be vastly improved by a new canonical mega-ledger format: _Catalogue_.

An XRPL _Catalogue_ is a file. Each Cataglogue contains the complete data of `1048575` sequential validated ledgers. The file format is organised such that the ledger data is both well compressed and fast to search without an external index, or preparation time.

The file format is a binary format. It begins with a header followed by several fixed length indicies, finally followed by several variable length sections.

### Constants and terms
| Term | Description |
|--|--|
|`NOT_FOUND`|Constant: 0xFFFFFFFFFFFFFFFFULL.|
|`::=`|_defined as_|
|`*`|0 or more instances of the preceeding|
|`(x)`|size or type information about preceeding field|
|`{x}`|exactly x instance of the preceeding|
|`[x]`|grouping of other elements such that another operator can act on the group|
|`varlen` |variable length|

## File Structure
Each section is explained in detail after this table.

|Size| Type / Info | Section | Notes |
|--|--|--|--|
| 48 b | [ magic version first_ledger_index offset{4} ] | Header| Contains file offsets to sections |
| 1 Kib | uint64{0x80} | StateIndex | Relative to StateOffset, location of each State Entry |
| 8 Mib | uint64{0x100000} | LedgerIndex | Relative to LedgerOffset |
| 128 Mib | uint64{0x1000000} | TransactionIndex | Relative to TransactionOffset |
| 128 Mib | uint64{0x1000000} | AccountIndex | Relative to AccountOffset |
| variable | [ len txn_hash blob meta ]* | Transaction | Sorted by txn_hash |
| variable | [ len acc_id txn_offset* ]* | Account | Sorted by acc_id |
| variable | [ len ledger_header ] | Ledger | Sorted by seq_num | 
| variable | [ len 7zblob ]{0x80} | State | Sorted by sequence number |

Grammatically:

        Catalogue ::= Header StateIndex LedgerIndex TransactionIndex AccountIndex Transaction Account Ledger State

## Sections

### Header
The header is 48 bytes containing the following fields.
|Size| Type | Entry | Description |
|--|--|--|--|
| 4b | uint32 | Magic Number | `0xCA7A` |
| 4b | uint32 | Version | `0` |
| 8b | uint64 | FirstLedgerIndex | The first ledger number in this Catalogue |
| 8b | uint64 | TransactionOffset| The file offset the _Transaction_ section starts at|
| 8b | uint64 | AccountOffset| The file offset the _Account_ section starts at|
| 8b | uint64 | LedgerOffset| The file offset the _Ledger_ section starts at|
| 8b | uint64 | StateOffset| The file offset the _State_ section starts at|

Grammatically:

        Header ::= MagicNumber(uint32) Version(uint32) FileOffset(uint64){4}

### LedgerIndex
An array of 1048576 offets relative to LedgerOffset allowing the direct indexing of a particular ledger header given a ledger sequence number. The array index is the ledger sequence number less the FirstLedgerIndex.

        LedgerIndex ::= FileOffset(uint64){1048576}

### StateIndex
An array of 128 offsets relative to StateOffset allowing the direct indexing of a particular StateEntry.

        StateIndex ::= FileOffset(uint64){128}

### TransactionIndex
An array of 16777216 offsets relative to TransactionOffset. The index of this array is the first 3 bytes of the transaction hash. The offset is the first TransactionEntry that begins with those bytes or `NOT_FOUND` if no such transaction exists across the whole span of 1048576 ledgers.

        TransactionIndex ::= FileOffset(uint64){16777216}

### AccountIndex
An array of 16777216 offsets relative to the AccountOffset. The index of this array is the first 3 bytes of the Account ID under query. The offset is the beginning of the set of transactions that affected any account with this prefix. The end of that set is the next non `NOT_FOUND` entry in the array or the beginning of the next section if there are no further `NOT_FOUND` entries in the AccountIndex array.

        AccountIndex ::= FileOffset(uint64){16777216}

### Transaction
An ordered list of transactions with their metadata that occured across the whole 1048576 ledgers.

        TransactionEntry ::= Len(uint32) TransactionHash(uint256) TxnBlob(varlen) Meta(varlen)

        TransactionSection ::= TransactionEntry*

### Account
A list sorted first by account then by transaction hash of transactions that affected those aforesaid accounts. Transactions are referenced via an offset relative to the Transaction Section. Transactions may be referenced more than once.
    
        AccountEntry ::= Len(uint32) AccountID(uint160) FileOffset(uint64)

        AccountSection ::= AccountEntry*


### Ledger
Ledger headers are stored uncompressed in this section.

Each Ledger header is placed into canonical XRPL serialized binary format then uint32 length prefixed. 

        LedgerEntry ::= Len(uint32) SLedgerHeader(varlen)

These LedgerEntries are concatenated in sequential order to make up the section:

        LedgerSection ::= LedgerEntry{1048576}

 

### State
Each consecutive set of `8192` ledgers, in canonical XRPL serialized binary format, are solidly compressed using 7z compression into an archive where the filename is the ledger number, as below:
        
        7ZipArchive ::= 
        7z compressed file structure
        [
            /
              |- 67256320           ** example **
              |- 67256321
              |- 67256322
              .
              .
              .
              |- 67264511
         ]
         
The 7ZipArchive is prefixed with a uint32 length. This is called a StateEntry:

        StateEntry ::= Len(uint32) 7ZipArchive(varlen)

Each of `128` State Entries are then sequentially written from the start of the State section to the end of the file.

        StateSection ::= StateEntry{128}

