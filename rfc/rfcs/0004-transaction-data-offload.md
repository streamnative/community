- **Status: Discussing**
- **Authors: Xiangying Meng**
- **Mailing List discussion**: null
- **tiered storage：**[https://github.com/apache/pulsar/wiki/PIP-17:-Tiered-storage-for-Pulsar-topics](https://github.com/apache/pulsar/wiki/PIP-17:-Tiered-storage-for-Pulsar-topics)
- **Transaction Buffer:** [SNIP 4: Transaction buffer data clear and dispatch transaction message](https://streamnative.slab.com/posts/bs296tyc)

# Motivation

Pulsar stores topics in bookkeeper ledgers. Bookkeeper ledger are normally replicated to three nodes. In order to save costs, when the topics need to be stored for a long time, pulsar may offload topics to an object store. In this case, we default that all the entries in topics is necessary.

But when we implement transaction in pulsar, some auxiliary Mark and aborted messages will be filtered out in bookkeeper:

- Transaction Mark: Identify the messages in a transaction how to deal
    - Commit mark: It means the messages in this transaction can be exposed to consumers.
    - Abort mark: It means the messages in this transaction have been abandoned.
- Aborted messages: the messages in a aborted transaction.

So what we need to do is filter out the unnecessary entries in the ledger that will to be offloaded.

**The remainder of the document covers how we will do these:**

- Filter out transaction Mark and aborted messages
- Only offload the ledger before Stable Position

# Background 

## TransactionBuffer

### Stable Position

![](https://static.slab.com/prod/uploads/cam7h8fn/posts/images/-68qKpKaOWnxsCVcC-oYTg8Q.png)

Let&#39;s first understand the implementation of transactionBuffer. There is a more important concept in the implementation of transactionBuffer: Stable Position. In order to explain this concept clearly, I introduced another variable ReadPosition in the above figure, which means the position we read. If when we read position 3, we know that transaction 1 is commit, so the message of transaction 1 can be read, but we don’t know that the message status of transaction 2 is commit or abort, so we can’t read the message of transaction 2 yet. Which means that we can only read the data of position 0 in the above figure. We use a variable Stable Position to record this position. When Read Position&lt;3, we don&#39;t know the status of the above-mentioned transactions, so these messages cannot be exposed to consumers. At this time, Stable Position = -1. Similarly, when Read Position = 3, Stable Position = 0; when Read Position &lt;6, Stable Position = 0; when Read Position =6 Stable Position = 4; when Read Position =7 Stable Position = 7;

**Documentation**

There is a detailed description of TransactionBuffer in this document

[SNIP 4: Transaction buffer data clear and dispatch transaction message](https://streamnative.slab.com/posts/bs296tyc)

###  SnapShot

TransactionBuffer will persist its own data according to certain conditions.

- No need to take snapshot before using transaction
- Regular take snapshot
- Take snapshot when the transaction messages exceeds a certain value
- When a producer build by a client which turn on transaction, TopicTransactionBuffer will take a snapshot first.And the State of TransactionBuffer will change to Ready from NoSnapShot



## Offload

When we create a ManagerLedger and initialize it, the initializeBookKeeper method will be used. In this method, a task (updateLastLedgerCreatedTimeAndScheduleRolloverTask) will be executed to periodically offload the ledger.The offload method of the LedgerOffloader interface is called at the end of the call chain.

Now LedgerOffload mainly has two implementations:

- FileSystemManagedLedgerOffloader
- BlobStoreManagedLedgerOffloader

They receive a ReadHandle while can read the entries of a ledger store in bookkeeper, and read the all entries , then offload them to an object store.

## ReadHandle

When we read data from tiered storage, we will call the readsync method of ReadHandle, which has three implementation classes.

ReadHandle, a class to read entry information in ledger.When it reads the entries of ledger, it traverses and reads one by one. If the entry read now is not the expected entry, it will find the entry with the index before the expected entry.This will cause an endless loop problem. The solution to it is discussed below.

# Todo

- Design a filter class
    - There needs to be a way to judge whether the transactionBuffer is NoSnapshot or Initializing.
    - Need to have a method to judge whether the current entry needs offload
    - Need to determine whether Position of the  current ledger is less than or equal to MaxReadPosition
    - add a class  OffloadFilterDisable.
- When a timing task triggers offload or the client calls offload,   filter out  the ledger that cannot  be offloaded currently in  the  list of ledgers
- When processing syncRead in ReadHandle, some of the entries may be filtered out, which may cause logic problems in the previous code.We need to optimize these code.
- If the message is read from offload,  It does not need to be subject to the read limit of MaxReadPosition.

Discussion (2021-8-26)

1. Append transaction buffer snapshot when receiving the first transaction message.
1. Check whether the ledger could be offloaded based upon the transaction buffer snapshot position.
1. Record transaction buffer snapshot in timing even if there is no transaction messages.  
