# Haste v1.0

## **PRELIMINARY DRAFT**

## Overview

Haste is a Decred Proof-of-Work (PoW) mining protocol which is used primarily
for pool mining.  This protocol was created to allow expansion and
enhancements to the Proof-of-Stake (PoS) functionality of Decred.

In the Decred network, PoW miners submit solved blocks to the network and PoS
miners vote on them. 5 tickets are chosen at random from the ticket pool to
vote on the block. If at least 3 votes are ‘Yes’ then the block is validated.

The votes are compromised of vote bits.  Currently, the values used are either
1 (Yes) or 0 (No).  In the near future, vote bits will be expanded so agendas
can be voted on with different options.

## Background

Presently, Haste is essentially a renaming of the widely used stratum mining
protocol.  In the future, unused fields like Coinbase 2 will be used
to provide Proof-of-Voting functionality via merkle proofs.  The decision to base
Haste on the stratum protocol was made to ensure faster uptake by miners
and mining pool operators due to its familiarity.

## Introduction

The underlying protocol relies upon
[JSON-RPC 2.0](http://www.jsonrpc.org/specification) for communication.
A typical session packet flow is shown below.  The methods in the example below
are described in greater detail in the next section.

**_Client initiates a TCP connection to the server and sends a mining.subscribe packet_**

Client :left_right_arrow: **mining.subscribe** :arrow_right: Server

**_Server responds by sending a mining.notify packet_**

Client :arrow_left: **mining.notify** :left_right_arrow: Server

**_Client sends a mining.authorize to login_**

Client :left_right_arrow: **mining.authorize** :arrow_right: Server

**_Server sends a status response message in response to mining.authorize_**

Client :arrow_left: **mining.authorize response** :left_right_arrow: Server

**_Server informs the client what difficulty to use for mining work_**

Client :arrow_left: **mining.set_difficulty** :left_right_arrow: Server

**_Server sends initial work, mining can begin_**

Client :arrow_left: **mining.notify** :left_right_arrow: Server

**_Server sends updated work_**

Client :arrow_left: **mining.notify** :left_right_arrow: Server

**_Server sends updated work_**

Client :arrow_left: **mining.notify** :left_right_arrow: Server

**_A share has been found and is submitted_**

Client :left_right_arrow: **mining.submit** :arrow_right: Server

**_Server sends a status response to signal whether the share was accepted or rejected_**

Client :arrow_left: **mining.submit response** :left_right_arrow: Server

**_Server sends updated work_**

Client :arrow_left: **mining.notify** :left_right_arrow: Server

**_Server sends updated work_**

Client :arrow_left: **mining.notify** :left_right_arrow: Server

**_Mining continues indefinitely_**

## Session Startup

Stratum is a series of JSON messages with an incremented ID to pair
messages and replies.

### Client sends mining.subscribe message to the stratum server to initiate the session:

```JSON
{"id": 1, "method": "mining.subscribe", "params": ["gominer/0.2.0-decred"]}
```

### Server sends mining.notify combination message in response to mining.subscribe

```JSON
{"id":1,"result":[[["mining.set_difficulty","1"],["mining.notify","2bd595e34826a3b6271400920d4decb8"]],"0000000000000000e3014335",12],"error":null}
```

The fields are:

| Field Name               | Example                          | Notes                                                                      |
| ------------------------ | -------------------------------- | -------------------------------------------------------------------------- |
| Subscription Type ID     | 1                                | Never used in practice.                                                    |
| Stratum Session ID       | 2bd595e34826a3b6271400920d4decb8 | Never used in practice.                                                    |
| Extranonce1              | 0000000000000000e3014335         | Unique session string which will be used for coinbase serialization later. |
| Extranonce2 Length       | 12                               | Formatted length of Extra Nonce that the pool expects the miner to use.    |

### Client sends mining.authorize so work is correctly associated with an account/payment address

```JSON
{"id": 2, "method": "mining.authorize", "params": ["user.workerid", "password"]}
```

### Server sends true if authorization succeeded or false with error message if authorization failed

```JSON
{"id":2,"result":true,"error":null}
```

## Work Data And Target

After subscribing to server pushes and performing authorization,
the server will send mining.difficulty and mining.notify messages.
Typically this occurs for the first time in a session directly after
the mining.subscribe reply so the miner can start performing work
immediately.  Then the messages are sent on as-needed basis.  For
example, the server will send new work if a new block has appeared on the
network.

These messages inform the mining software what work to perform
and the target/difficulty of the work.  These messages do not contain
an ID field because no message is sent in response to them.

### mining.difficulty

Stratum servers can be either be fixed or auto-adjusting difficulty.
Fixed difficulty pools will only set the difficulty at the beginning of a
session. Auto-adjusting difficulty pools adapt to the speed of the miner.
The pool will increase or decrease the difficulty in an attempt to have the
miner to submit at a fixed interval (typically at least once per minute).
An example of such a message is below:

```JSON
{"id":null,"method":"mining.set_difficulty","params":[1]}
```

### mining.notify

This message contains everything necessary to derive work which will be passed
to a hashing device. An example message is below:

```JSON
{"id":null,"method":"mining.notify","params":["76df","7817c24aa99f3999a57dcfc8a7a834f92ebb442f8d519dbd000009e000000000","d52f367013ddc74d61a4f50c0d47c4b8e87b6d89a603a04447bcd2b110c508e31d37308253d38bbe0464508f4eb1f12b92e6431d41b01f01518d2d9b9b64fe93010098588e9b1c650500030052a90000b778171a134e691e01000000b1c70000ce0f0000d052a1570000000000000000","",[],"01000000","1a1778b7","57a152d0",false]}
```

The fields contained in params are:

| Field Name     | Purpose | Example |
| -------------- | --- | --- |
| JobID          | ID of the job. Used when submitting a solved shared to the server.                             | 76df |
| PrevHash       | Hash of the previous block.  Used when deriving work.                                          | 7817c24aa99f3999a57dcfc8a7a834f9<br>2ebb442f8d519dbd000009e000000000 |
| CoinBase1      | Other part of the block header.  Used when deriving work.                                      | d52f367013ddc74d61a4f50c0d47c4b8<br>e87b6d89a603a04447bcd2b110c508e3<br>1d37308253d38bbe0464508f4eb1f12b<br>92e6431d41b01f01518d2d9b9b64fe93<br>010098588e9b1c650500030052a90000<br>b778171a134e691e01000000b1c70000<br>ce0f0000d052a1570000000000000000 |
| CoinBase2      | Final part of the coinbase transaction.  Not used.                                             | Empty String |
| MerkleBranches | Array of merkle branches. Unused at this time.                                                 | Empty Array |
| BlockVersion   | Decred block version.  Already in CoinBase 1, not useful.                                      | 01000000 |
| Nbits          | Encoded current network difficulty.  Already in CoinBase 1, not useful.                        | 1a1778b7 |
| Ntime          | Server's time when the job was transmitted. Already in CoinBase 1, not useful.                 | 57a152d0 |
| CleanJobs      | When true, discard current work and re-calculate derived work.                                 | false |

## Deriving Work And Performing Hashing

### Converting Stratum Work To "getwork" Style Work

#### Constructing A Block Header Template

Field          | Description                                                                            | Size
---            | ---                                                                                    | ---
Version        | Decred Block header version                                                            | 4 bytes
PrevHash       | Hash of the previous block                                                             | 32 bytes
CoinBase1      | Merkle tree hash calculated using all transactions in the block                        | 108 bytes

### Calculate Midstate

The midstate is obtained by calling the BLAKE-256 hashing function on the first
two 64-byte units of the block header.

### Performing Hashing / Mining

The midstate is then passed to the hashing code that runs on the device.  Mining
is performed by incrementing the nonce and the device code currently returns
whenever a difficulty 1 share is found.  The returned value is then checked against
the mining target.

### Validating Work

The pool difficulty is converted to a standard mining difficulty by taking the
highest allowed proof-of-work value that a Decred block can have and dividing
that value by the pool difficulty.  The byte string is reversed and then
encoded as big endian numbers. The value for the main network is 2^224 - 1 and
the value for the test network is 2^232 - 1.

The block header template generated above is then copied and modified to
construct a finalized block header. This is done by overwriting the timestamp
using the Ntime from the current job and the nonce found from the device. A
hash of the finalized block header is then taken and compared to the difficulty
target.

If the hash of the finalized block header is below target then the share is
submitted to the stratum server for validation.  Otherwise the false positive
is discarded, work is possibly updated, and hashing resumes.

## Submitting A Share

### Client constructs and sends mining.submit message

```JSON
{"method": "mining.submit", "params": ["user.worker", "187", "0100000000188fece3014335", "5783c78e", "f6c4e01d"], "id":4}
```

The parameters included in the submission and their sources are as follows:

| Field Name               | Source                                         | Example                  |
| ------------------------ | ---------------------------------------------- | ------------------------ |
| Username/Payment Address | User specified                                 | user.worker              |
| JobID                    | Pool                                           | 187                      |
| Extra Nonce              | Bytes 144 to 156 from finalized block header   | 0100000000188fece3014335 |
| Ntime                    | Pool                                           | 5783c78e                 |
| Nonce                    | Device                                         | f6c4e01d                 |

### Server sends true if share accepted or false with error message if share rejected

```JSON
{"id":4,"result":true,"error":null}
```

The reject rate of submitted shares should be about 0.1%. The most typical error
message is "job not found" which happens when submitting an old share that was
solved using stale work.  This is typically a race condition which occurs when
a new block was just propagated across the network and the miner's work was not
updated prior to solving the share.  Some errors may also be caused by a bug in
the pool's software.

## Stratum Peculiarities and Unused Stratum Extensions

Stratum has some oddities that stem from being rushed into production use with
little planning and community feedback.  One example is:

* When solo mining, the timestamp of the block header is refers to the
approximate time when the block was mined.  When using a stratum server, the
timestamp of the last job (Ntime) is always used instead.  This shouldn't be
necessary since the JobID refers to the last job already.  Unfortunately,
this prevents generating new work by simply incrementing the timestamp.

Stratum has some extensions which are either unused by Decred pools or are not
in general use.  These remain unimplemented in the haste reference stack.

| Method                      | Purpose | Notes |
| --------------------------- | --- | --- |
| client.get_version          | Sends version string | Unnecessary / not widely used. |
| client.reconnect            | Requests that the client reconnect to a specified host/port. | Not widely used. |
| client.show_message         | Displays string. | Not widely used. |
| mining.ping                 | Server <-> Client Ping | Unnecessary / not widely used. |
| mining.extranonce.subscribe | Signals that the client supports the mining.set_extranonce method. | Used to replace initial Extranonce1 / Extranonce2 Length values. |

## Pseudo Implementation

The following serves as an overview to implementing a Haste client and
performing:

```
Step 01
=======
Open a TCP connection to the server.  Set JSON-RPC ID to 1 and send the
mining.subscribe message with the client version string as a parameter.

Step 02
=======
Parse mining.notify response and save the Extranonce1 and Extranonce2 Length
store them in the Haste data structure.

Step 03
=======
Increment JSON-RPC ID and send mining.authorize with login and password as
parameters.

Step 04
=======
Exit if mining.authorize response with matching JSON-RPC ID has result set
to false and display error message.

Step 05
=======
Parse the mining.set_difficulty message and store the difficulty in the Haste
data structure.

Step 06
=======
Parse the JobID, PrevHash, CoinBase1, BlockVersion, and NTime parameters from
the mining.notify message and store them in the Haste data structure.

Step 07
=======
Convert the data stored in the Haste data structure into getwork-style data

// Construct Extranonce which is a per-session identifier

ExtraNonce = ""
ExtraNonce1 = pool.ExtraNonce1
ExtraNonce2 = Sprintf("%0" + ExtraNonce2Length * 2 + "x", 0)

append(ExtraNonce, ExtraNonce1)
append(ExtraNonce, ExtraNonce2)

BlockHeaderTemplate = ""

Append(BlockHeaderTemplate, pool.BlockVersion)
Append(BlockHeaderTemplate, pool.PrevHash)
Append(BlockHeaderTemplate, pool.CoinBase1)
Append(BlockHeaderTemplate, 4 random bytes)

Replace(BlockHeaderTemplate[Bytes 140-144], pool.Ntime)
Replace(BlockHeaderTemplate[Bytes 144-148], ExtraNonce)

// Calculate Midstate

MidState = Blake256(BlockHeaderTemplate[Bytes 0 to 128])

// Adapt Stratum Difficulty To Network Target

Network = main | test

mainPowLimit = 2^224 - 1
testPowLimit = 2^232 - 1

WorkTarget = "net"+"PowLimit" / pool.Target

Step 08
=======
Pass the calculated midstate to the device and wait for it to return from its
hashing loop.  Update the device work from the Haste work if necessary and
restart the hashing on the device.

Step 09
=======
If a valid solution is returned from the device, finalize the header and hash
and check the finalized hash against the difficulty target.

// Validate Work

FinalBlockHeader = BlockHeaderTemplate

Replace(FinalBlockHeader[Bytes 140-144], pool.Ntime)
Replace(FinalBlockHeader[Bytes 144-148], ExtraNonce)

BlockHash = Blake256(FinalBlockHeader)

if (BlockHash < WorkTarget) {
    SubmitShare()
}

Step 10
=======
When a valid share is found, increment JSON-RPC ID and send mining.submit
with login, JobID, Extranonce (Bytes 144 to 156 from the finalized block
header), Ntime, and Nonce (from the Device).

Step 11
=======
Parse the response with matching JSON-RPC ID.  If result is true, increment
accepted shares counter, otherwise increment rejected shares counter and
display the error message to the user.
```

## Go Programming Language Example

[gominer](https://github.com/decred/gominer) is the reference implementation
and simplified example code will be broken out to better serve as an example
at a later date.

## See Also

The following stratum resources may also assist in implementing Haste:

* [Stratum Author's Documentation](https://slushpool.com/help/#!/manual/stratum-protocol)
* [Stratum Network Protocol Documentation](https://docs.google.com/document/d/17zHy1SUlhgtCMbypO8cHgpWH73V5iUQKk_0rWvMqSNs/)
* [Bitcoin Wiki Stratum Protocol Article](https://en.bitcoin.it/wiki/Stratum_mining_protocol)
