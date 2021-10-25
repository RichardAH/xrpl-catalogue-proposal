# XRPL Catalogue Proposal

## Introduction
The XRP ledger produces a large amount of immutable historical data which currently must be maintained across a network of [Full History](https://xrpl.org/configure-full-history.html) nodes or it will be forever lost.

Note that some optimisation of history sharing and storage has already been achieved through [History Sharding](https://xrpl.org/history-sharding.html). However this proposal will make the case that the robustness and searchability of network history would be vastly improved by a new canonical mega-ledger format: _Catalogue_.

An XRPL _Catalogue_ is a file. Each Cataglogue contains the complete data of `1048575` sequential validated ledgers. The file format is organised such that the ledger data is both well compressed and fast to search without an external index, or preparation time.

The file format is a binary format. It begins with a header followed by several fixed length indicies, finally followed by several variable length sections.

## Overview
 Sections start at file offset `0`:
|Size| Type / Info | Section | Notes |
|--|--|--|--|
| 48 b | _Catalogue Header_  | Header| Contains file offsets to sections |
| 1 Kib | uint64_t[0x80] | StateIndex | Relative to StateOffset, location of each State Entry |
| 8 Mib | uint64_t[0x100000] | LedgerIndex | Relative to LedgerOffset |
| 128 Mib | uint64_t[0x1000000] | TransactionIndex | Relative to TransactionOffset |
| 128 Mib | uint64_t[0x1000000] | AccountIndex | Relative to AccountOffset |
| variable | [len txn_hash blob meta]* | Transaction | Sorted by txn_hash |
| variable | [len acc_id txn_offset* ]* | Account | Sorted by acc_id |
| variable | [len ledger_header] | Ledger | Sorted by seq_num | 
| variable | [len 7zblob]{0x80} | State | Sorted by sequence number |

## Sections

### Header
The header is 48 bytes containing the following fields.
|Size| Type | Entry | Description |
|--|--|--|--|
| 4b | uint32_t | Magic Number | `0xCA7A` |
| 4b | uint32_t | Version | `0` |
| 8b | uint64_t | FirstLedgerIndex | The first ledger number in this Catalogue |
| 8b | uint64_t | TransactionOffset| The file offset the _Transaction_ section starts at|
| 8b | uint64_t | AccountOffset| The file offset the _Account_ section starts at|
| 8b | uint64_t | LedgerOffset| The file offset the _Ledger_ section starts at|
| 8b | uint64_t | StateOffset| The file offset the _State_ section starts at|


### LedgerIndex

### StateIndex

### TransactionIndex

### AccountIndex

### Transaction

### Account

### Ledger
Ledger headers are stored uncompressed in this section.

Each Ledger header is placed into canonical XRPL serialized binary format then uint32_t length prefixed. 

	    LedgerEntry ::= Len(uint32_t) SLedgerHeader(variable length)

These LedgerEntries are concatenated in sequential order to make up the section:

	    LedgerSection ::= LedgerEntry{1048576}

 

### State
Each consecutive set of `8192` ledgers, in canonical XRPL serialized binary format, are solidly compressed using 7z compression into an archive where the filename is the ledger number, as below:

		7ZipArchive ::= 
		    ** example **
		    /
		      |- 67256320
		      |- 67256321
		      |- 67256322
		      .
		      .
		      .
		      |- 67264511
		      
The 7ZipArchive is prefixed with a uint32_t length. This is called a StateEntry:

		StateEntry ::= Len(uint32_t) 7ZipArchive(variable length)

Each of `128` State Entries are then sequentially written from the start of the State section to the end of the file.

		StateSection ::= StateEntry{128}

