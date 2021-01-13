# RFC Title

- Status: `In-Progress`
- Feature Name: Transaction buffer design
- Propose Date: 2021-1-13
- RFC PR: [streamnative/community#0001](https://github.com/streamnative/community/pull/3)
- Project Issue: [apache/pulsar#0001](https://github.com/apache/pulsar/pull/9195)
- Authors: congbo

# Motivation

Currently, we produce transaction message in the topic ManagedLedger. When we commit or abort this transaction, we will write a marker into this managedLedger. We store transaction have sent messages in memory, TC will send commit or abort command and then we can know witch messages we need to commit or abort.We handle transaction message after we read the mark. Because, we already store the transaction messageIds in the mark metadata, we can deserialize the metadata and get the messageIds witch is belong to this transaction.This will produce some problem.

- When one transaction commit or abort, client also can produce message into ManagedLedger.
- We can&#39;t store the transaction&#39;s messageIds into marker, because it may bigger than 5MB.
- We don&#39;t store the messageIds in marker,  we should store the messageIds in sub dispatcher, it will take up a lot memory.
- If we dispatch message after we read a commit mark, it will not ensure the order of messages.
- If we cumulative ack may lost the message which have not read the mark.

# Condition

What kind of transaction buffer should we design?

1. Delete the unless transaction message in ManagedLedger.
1. Don&#39; store the messageIds in mark metadata.
1. No messageIds corresponding to the transaction in memory.
1. If we cumulative ack will not lost the message witch have not read mark and dispatch to consumer.
1. Ensure the order of messages when dispatch.

# Background

[https://docs.google.com/document/d/145VYp09JKTw9jAT-7yNyFU255FptB2_B2Fye100ZXDI](https://docs.google.com/document/d/145VYp09JKTw9jAT-7yNyFU255FptB2_B2Fye100ZXDI)

# Design

## Stable position

We need to ensure the order of messages. When we ensure the order of message, we are already ensure the cumulative ack will not lost message. So we should know witch positions before this position we can dispatch.  We call the position is **stable position**.

### Working principle

![](https://static.slab.com/prod/uploads/cam7h8fn/posts/images/o93HAuBlO9ghXLOFkdymMm5b.png)

You will have questions.Why stable position is always -1？When we have not read any transaction mark, we will not  change the stable position. It means that we will not dispatch any message to consumer

![](https://static.slab.com/prod/uploads/cam7h8fn/posts/images/2Dp81-qMqW-Ny7tRlzIWqtX8.png)

When we read **Tnx 1** mark, this means that all messages witch belong to **Tnx 1** can dispatch, but before **Ps 2** the **Ps 1** is belong to **Txn 2**, so we should ensure the order of messages we only can dispatch the **Ps 0** to consumer and the **SP = 0**. It means that we can read from **PS -1** to **PS 0** second time and dispatch these positions.

![](https://static.slab.com/prod/uploads/cam7h8fn/posts/images/Nq3AqMooboCUJ7nLM14dCUI2.png)

The same when we read position to 7, we can read from **PS 0** to **PS 7**. The result as picture shows.

### Implement



![](https://static.slab.com/prod/uploads/cam7h8fn/posts/images/ST9WEcZMjognSKBaHvj5Z5sZ.png)

As the picture show, we only need the first position of the transaction. When we read a mark we can remove the first position of the transaction in the queue, it means all this transaction messages can dispatch. It saves memory and don&#39;t need additional data structure.

### Choose

Obviously we choose the easy plan.



![](https://static.slab.com/prod/uploads/cam7h8fn/posts/images/JXXEU53bTkuZiECpMJ-9B_dN.png)

### Reject alternatives

As the picture show, it is easy to think of we should to maintain an ordered queue. When we read mark, we can remove the position witch is belong to this transaction of this mark. But it have some problem :

- When we have not read a transaction mark, we have to record the all the position in memory witch is belong to this transaction.
- Position queue is orderly, so we have to maintain a map with the corresponding relationship between position and transaction. 

### Abort mark

The above are all plans based on commit mark, how we handle the abort mark?

We need read transaction message twice in broker, so we can record the abort mark in memory. When we read from perv stable position to current stable position, we have already know witch transaction have been aborted.

![](https://static.slab.com/prod/uploads/cam7h8fn/posts/images/TDMLhUt_tobCdQ6TxEY9Pg3x.png)

When we read **Txn 1** mark, we have already know that the **Txn 1** have been aborted. So when we read from **SP -1** to **SP 0** again, the **SP 0** is **Txn 1** and it have been aborted,  we can ack this message directly and don&#39;t dispatch this message to consumer.



![](https://static.slab.com/prod/uploads/cam7h8fn/posts/images/f8huhs-fmKVlFIqTP08ECoSe.png)

What time we can delete the abort mark from memory?  After we read **PS 7**, we can know that change the **SP 0** to **SP 7.** When we find the **SP** changed, we can read **SP 0** to **SP 7**. The **Txn 1** abort mark position is smaller than **SP**, so we can delete then **Txn 1** abort mark from memory. The same is true for **Txn3** abort mark.

## LowWaterMark

Now, Transaction buffer don&#39;t know the TxnId is repeated, Tc can&#39;t  write abort mark by time out mechanism to control the Illegal transaction message append to ManagedLedger. We mast find a good way to solve this question.

![](https://static.slab.com/prod/uploads/cam7h8fn/posts/images/L9yBsER1tyZdP9ulwK6J4fKw.png)

As shown in the figure above, Txn1 m4 can&#39;t be cleared. Because after TC time out and then the TC will never write any mark to TB with Txn1.

### Transaction buffer handle

We can&#39;t handle the messages witch send after have written commit or abort mark. So when we handle the commit or abort command protobuf with the least txnId, we can judge witch txnId has been invalid. We should maintain an orderly data structure to handle which txnId is smaller than the least txnIn in TC.

![](https://static.slab.com/prod/uploads/cam7h8fn/posts/images/HcPnQR896tGccF7_XAAd2bPG.png)



![](https://static.slab.com/prod/uploads/cam7h8fn/posts/images/YFAwZ43btyP1OIT7i2NKaIx0.png)

Transaction buffer can maintain a map `Map<TCid, Queue<Long>>` , when the queue store sequenceId is smaller than the least sequenceId in TC. We will write an abort marker in this topic&#39;s ManagedLedger again. This can clear the redundant data of transaction buffer completely!

### Advantage

We write the Invalid transaction abort mark in ManagedLedger. Dispatcher only need to handle this by a normal abort mark. Dispatcher logical will become very easy.

### Disadvantage

Transaction buffer need to replay the ManagedLedger when every time ManagedLedger close. When it replay, the topic can&#39;t write the transaction message into ManagedLedger. It is difficult to store the start position when it replay. If we use cursor individual ack the position, it will take up a lot of memory and disk. We also need to maintain the ongoing transaction in the memory, it also will take up a lot of memory.

### Dispatcher handle

Dispatcher don&#39;t know the LowWaterMark of this TC, but we can store the LowWaterMark when we write the abort or commit mark. Now when dispatcher read mark, it also know this time of the TC LowWaterMark.



![](https://static.slab.com/prod/uploads/cam7h8fn/posts/images/-kimVRsJgYCjrvV53De1O3fz.png)

As the picture show, we can&#39;t read any mark with **Txn 1**. This picture lose the LowWaterMark. Next picture we will understand the LowWaterMark how work.



![](https://static.slab.com/prod/uploads/cam7h8fn/posts/images/WxFCL-MZW-1Pd-AegbHPXPy0.png)

As the picture show, we can delete **Txn 1** beacause **Txn 1** is smaller than **LowWaterMark 2**. We can add the **Txn 1** to abort Map. When dispatch read **PS 5** we will know the **Tnx 1** have been aborted.



![](https://static.slab.com/prod/uploads/cam7h8fn/posts/images/Cn23UI0p5TTKgptFdAS5RzqD.png)

When we read **PS 6,** we can remove the **T1, P6** from abort map.

### Advantage

We don&#39;t need add any structure to maintain the LowWaterMark, transaction buffer don&#39;t need to replay, He is only responsible for writing abort and commit mark.

### Disadvantage

When we write a mark, we have to add a field, the field type is Long. So every time we write mark, we will store a Long more.

### Choose

Although dispatcher handle store a Long more when write mark, but transaction buffer should replay and maintain an ordered structure for ongoing transaction.  So, we only write a Long more can solve the unless transaction and don&#39;t add any structure and replay ManageLedger, we choose dispatcher to handle the ongoing transaction.

## Final plan

The above plan have some problem:

- Every dispatcher have to store the every transaction stable position and abort transaction.
- Dispatcher should read twice when stable position change.

How to solve these problem:

Transaction buffer know every transaction state. Maintain the transaction stable position and abort transaction in transaction buffer, when the once read position we can obtain the stable position from transaction buffer and filter the abort transaction.

One problem, how we replay transaction stable position and abort transaction quickly? Store snapshot in mark and use cursor to replay.  Also we don&#39;t need to store the lowWaterMark.

### Advantage

- Don&#39;t need to read twice when dispatch.
- Don&#39;t need to maintain transaction stable position and abort transaction in every dispatcher.
- Don&#39;t need to store lowWaterMark.

### Disadvantage

- When topic init transaction buffer need to replay the transaction stable position and abort transaction , in this time dispatcher can&#39;t read entries.
- Transaction buffer need to store snapshot.
- When the message have been deleted the abort transaction can delete from the memory.

## Process

### Transaction buffer

- Add one protobuf format with least txnId in TC `CommandEndTxnOnPartition`

```
	message CommandEndTxnOnPartition {
    required uint64 request_id = 1;
    optional uint64 txnid_least_bits = 2 [default = 0];
    optional uint64 txnid_most_bits = 3 [default = 0];
    optional string topic = 4;
    optional TxnAction txn_action = 5;
    optional uint64 txnid_least_bits_of_low_watermark = 6;
}

```

Maintain the transaction stable position map and abort map.

### Dispatcher

- Can&#39;t read position more than transaction stable position.
- Filter the abort transaction message.

## In summery

### Meet the conditions？

Look up to the [Condition](https://streamnative.slab.com/posts/snip-4-transaction-buffer-data-clear-and-dispatch-transaction-message-bs296tyc#condition), Is our design satisfied?

- LowWaterMark implement meet [Condition 1](https://streamnative.slab.com/posts/snip-4-transaction-buffer-data-clear-and-dispatch-transaction-message-bs296tyc#condition) 
- We don&#39;t store the messageId in metadata meet [Condition 2](https://streamnative.slab.com/posts/snip-4-transaction-buffer-data-clear-and-dispatch-transaction-message-bs296tyc#condition)
- Stable position implement meet [Condition 4, Condition 5](https://streamnative.slab.com/posts/snip-4-transaction-buffer-data-clear-and-dispatch-transaction-message-bs296tyc#condition)
- We don&#39;t store the messageId corresponding the transaction in memory meet [Condition 3](https://streamnative.slab.com/posts/snip-4-transaction-buffer-data-clear-and-dispatch-transaction-message-bs296tyc#condition)

### Disadvantage

- LowWaterMark is affected by the entire TC, unstable situations will occur. But you use transaction correctly, it will not happen.
- Now we use stable position,  after a transaction is committed, the message sent for this transaction will be affected by other uncommitted transactions.
- We ensure that consume messages are in the order in which they are actually sent, but there is no guarantee that transactions are committed first and consumed first.

### When you read over this disadvantage. I hope can get some suggestions for these disadvantages. Will it cause defects in use?  In other words, can we really tolerate these disadvantages?
