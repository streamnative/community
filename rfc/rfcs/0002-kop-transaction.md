# RFC-0002: KoP Transaction

- Status: `Proposal`
- Feature Name: KoP Transaction
- Propose Date: 2020-12-07
- RFC PR: [streamnative/community#2](https://github.com/streamnative/community/pull/2)
- Project Issue: [streamnative/kop#39](https://github.com/streamnative/kop/issues/39)
- Authors: Ran Gao

## Motivation

Make KoP support Kafka transaction feature leverage Pulsar transaction components.

## Compare With Pulsar Transaction

The ideal solution for supporting the KoP transaction is to leverage Pulsar transaction mode, but this is hard to achieve because there are some differences between Pulsar Transaction mode and Kafka Transaction mode.

For Kafka, users must specify a unique `transactional.id` for the client, and there is only one ongoing transaction at the same time. In a Pulsar client, multiple transactions could exist at the same time.

### Transaction Coordinator

#### Status

Kafka TC has the status as below, but the Pulsar doesn&#39;t maintain TC status, Pulsar maintains status for every transaction.

1. Empty: transaction has not existed yet.
1. Ongoing
1. PrepareCommit
1. PrepareAbort
1. CompleteCommit
1. CompleteAbort
1. Dead: transactional id has expired and is about to be removed from the transaction side.
1. PrepareEpochFence: in the middle of bumping the epoch and fencing out older producers.

There are some statuses (`Empty`, `Dead`, and `PrepareEpochFence`) that can&#39;t map with Pulsar transaction status, so it&#39;s hard to see the Kafka TC as a special Pulsar transaction.

#### Metadata

Kafka TC needs to maintain the map of the &lt;transactional.id, PID, epoch&gt; and the offset commit group id, this is quite different from Pulsar.

#### PID &amp; Epoch

Kafka TC needs to initialize a Producer id for the _transactional.id_ and create a monotone increasing epoch for PID.

If a new producer initialized with the same _transactional.id_, the same PID will return to the producer. If the transaction status of this _transactional.id_ is prepareCommit, it will be committed completely, otherwise, it will be aborted.

Kafka TC will check the epoch for every transaction request, if the epoch along with the request smaller than the epoch in the cache, the request will be rejected.

### Acknowledgment

Kafka uses a consumer coordinator to maintains all partition offsets and save the offset commit data to a special topic, so we couldn&#39;t reuse the Pulsar pending-ack mechanism and we couldn&#39;t use the Pulsar TC to complete a transaction ack.

The above differences make creating a new TC for KoP is more reasonable.

## Approach

Because there are many differences between Pulsar and Kafka transaction mode, we decide to create a new transaction coordinator for KoP.

### TransactionMetadata

### TransactionCoordinator

The KoP transaction coordinator also backed on the Pulsar TC topic `pulsar/system/transaction_coordinator_assign` , compute the hash code of the _transactional.id_ and mod _tc_assign_ topic partition count to find the owner broker of the  _tc_assign_ topic partition.

The KoP transaction coordinator used to handle transaction requests by producers. Each KoP TC is responsible for a set of producers, producers with _transactional.id_ assigned to their corresponding coordinators.

Each transactional id only maintains an ongoing transaction. The KoP TC could use a map to manage the transactional id and its ongoing transaction.

```
public class TransactionCoordinator {

  public void handleInitProducerId(String transactionalId,
                                   CompletableFuture<AbstractResponse> response) {

  }

  public void handleAddPartitionsToTransaction(String transactionalId,
                                               long producerId,
                                               short epoch,
                                               List<TopicPartition> partitionList,
                                               CompletableFuture<AbstractResponse> response) {
  }

  public void handleEndTransaction(String transactionalId,
                                   long producerId,
                                   short epoch,
                                   CompletableFuture<AbstractResponse> response) {

  }

}
```

TransactionState

```
public enum TransactionState {
  EMPTY,
  ONGOING,
  PREPARE_COMMIT,
  PREPARE_ABORT,
  COMPLETE_COMMIT,
  COMPLETE_ABORT,
  DEAD,
  PREPARE_EPOCH_FENCE;
}
```

### TransactioinLog

Save transaction log in the managed ledger `public/default/__transaction_state-partition-x` .

KoP read the kop transaction log topic and recover all metadata to the cache.

### Transaction Marker Handler

When ending a transaction, the Kafka client will send an _EndTxnRequest_ to the KoP transaction coordinator, the KoP TC will change the transaction status to _prepareCommit_ or _prepareAbort_ and send _WriteTxnMarkersRequest_ to leader brokers of related topic partitions.

We need to add a new class _TransactionMarkerChannelHandler_ to handle the transaction marker process and manage network requests and responses.

### ProducerIdManager



### Transaction Messages Read

Currently, the KoP needs to read messages by the Pulsar _ManagedCursor_ and is lack transaction messages handle, if the `isolation.level` is _read-committed_ only return the committed transaction messages, if the `isolation.level` is _read-uncommitted_ all messages should be returned also the aborted transaction messages.

## DataFlow

![](https://static.slab.com/prod/uploads/cam7h8fn/posts/images/0uvZiYbajlQCuXWRWrT4K9v5.png)

#### 1. Find Coordinator

Command: FIND_COORDINATOR

Request: FindCoordinatorRequest

Kafka client sends transactional.id to KoP to find its coordinator, KoP compute hash code by transactional.id to get the TC topic partition and find the topic partition owner Broker, use the owner Broker as the coordinator.

hash(transactional.id ) mod tc_count -&gt; TC backed topic partition owner Broker as the KoP transaction coordinator

#### 2. Transaction Initialization

Command: INIT_PRODUCER_ID

Request: InitProducerIdRequest

Client Method:

`void initTransactions();`

Kafka client registers transactional.id to KoP, get the PID and epoch from KoP.

KoP needs to do:

1. generate the cluster unique PID leverage ZK, PID should greater than the TC backed topic partition count.
1. generate a monotonically increasing epoch for this transactional.id.
1. save &lt;transactionalId, PID, epoch&gt; to the KoP transaction log topic `${topic-partition}-txn-metadata`.
1. recover any transaction left incomplete by the previous instance of the producer.

Producer Fencing

If the epoch of the PID smaller than KoP current epoch, the producer fencing should be triggered, if the old transaction is prepareCommit then wait for the old transaction commit completely, otherwise, abort the old transaction.

#### 3. Transaction Begin

Client Method:

`void beginTransaction() throws ProducerFencedException;`

Kafka client begins the transaction, change Kafka client internal transaction status, this step doesn&#39;t need to communicate with KoP.

#### 4.1 Add Partitions To Txn

Command: ADD_PARTITIONS_TO_TXN

Request: _AddPartitionsToTxnRequest_

Kafka client send _AddPartitionsToTxnRequest_ to coordinator.

KoP coordinator needs to do:

1. save partition to the transaction log.
1. if the partitions of the _transactional.id_ are empty, change the transaction status to ongoing.

#### 4.2 Send Transaction Messages

Command: PRODUCE

Request: ProduceRequest

Client method:

```
Future<RecordMetadata> send(ProducerRecord<K, V> record);
Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback);
```

Kafka client sends _ProduceRequest_ to KoP, KoP could get the _PID_ and _epoch_.

KoP needs to do:

1. messages deduplication
1. send pulsar transaction messages, TxnID &lt;PID, epoch&gt;

#### 4.3 Add Offsets To Txn

Command: ADD_OFFSETS_TO_TXN

Request: AddOffsetsToTxnRequest

KoP needs to do:

1. call the method _TransactionMetadataStoreServiceaddAckedPartitionToTxn_ to add the groupId to the transaction metadata, use the groupId as the topic name and the partition is -1, TxnID &lt;PID, epoch&gt;

#### 4.4 Txn Offset Commit

Command: TXN_OFFSET_COMMIT

Request: TxnOffsetCommitRequest

Client Method:

```
void sendOffsetsToTransaction(Map<TopicPartition, OffsetAndMetadata> offsets,
                                  String consumerGroupId) throws ProducerFencedException;
void sendOffsetsToTransaction(Map<TopicPartition, OffsetAndMetadata> offsets,
                                  ConsumerGroupMetadata groupMetadata) throws ProducerFencedException;
```

// KoP could get the topic partition and it&#39;s ack offset

similar as the offset commit operation

KoP needs to do:

1. publish txn offset commit messages to topic partition `__consumer_offsets`.
1. update group coordinator partitions offset, save in _GroupMetadata#pendingTransactionalOffsetCommits._

#### 5. End Txn

Command: END_TXN

Request: EndTxnRequest

Client Method:

```
void commitTransaction() throws ProducerFencedException;
void abortTransaction() throws ProducerFencedException;
```

// KoP add topic partitions to the transaction and commit the transaction

KoP needs to do:

1. save _prepareCommit_(_prepareAbort_) operation in the transaction log.
1. call the topic partition leader broker to save the control message.
1. call the consumer coordinator of the offset commit groupId, write control message to offset topic, if commit updates the offset cache if abort ignored.

#### 5.2 Append Control Messages

Command: WRITE_TXN_MARKERS

Request: WriteTxnMarkersRequest

KoP needs to do:

1. send _WriteTxnMarkersRequest_ to the related normal partition leaders and the related consumer coordinator.

### PID

#### PID Generate

Reuse the Kafka PID generate strategy

1. Add ZK node `/latest_producer_id_block` called PID_NODE.

#### PID Expiration (refer to KIP)

It would be undesirable to let the PID-sequence map grew indefinitely, so we need a mechanism for **PID expiration**. We expire producerId’s when the age of the last message with that producerId exceeds the transactionalId expiration time or the topic’s retention time, whichever happens sooner. This rule applies even for non-transactional producers.

If the transactionalId expiration time is less than the topic’s retention time, then the producerId will be ‘logically’ expired. In particular, its information will not be materialized in the producerId-&gt;sequence mapping, but the messages with that producerId would remain in the log until they are eventually removed.

### Coordinator TransactionalId Expiration (refer to KIP)

Ideally, we would like to keep TransactionalId entries in the mapping forever, but for practical purposes we want to evict the ones that are not used any longer to avoid having the mapping growing without bounds. Consequently, we need a mechanism to detect inactivity and evict the corresponding identifiers. In order to do so, the transaction coordinator will periodically trigger the following procedure:

Scan the TransactionalId map in memory. For each TransactionalId -&gt; PID entry, if it does NOT have a current ongoing transaction in the transaction status map, AND the age of the last completed transaction is greater than the [TransactionalId expiration config](https://docs.google.com/document/d/11Jqy_GjUGtdXJK94XGsEIK7CP1SnQGdp2eF0wSw9ra8/edit#bookmark=id.nfan5rg6kjgk), remove the entry from the map. We will write the tombstone for the TransactionalId, but do not care if it fails, since in the worst case the TransactionalId will persist for a little longer (ie. the transactional.id.expration.ms duration).

### KoP Transaction Coordinator

#### Recover

The KoP transaction coordinator could leverage Pulsar TC recovery, after the Pulsar TC recovery KoP could get all transactions metadata.

KoP read the kop transaction log topic and recover all transaction extra metadata (Transactional Id, PID, epoch) to the cache, and KoP could get the topic partitions and offset commit group id by method _TransactionMetadataStoreService#getTxnMeta_.

### Consumer Coordinator

#### Recover

1. For each consumer offset message read from the offset topic, check if PID and Epoch fields are specified, if yes hold it from putting into the cache.
1. For each control message read from the offset topic, if it is a COMMIT transaction marker then put the previously kept offset entry into the cache; if it is an ABORT transaction maker then forget the previously kept offset entry.

### Commands

#### FIND_COORDINATOR

```
{ "name": "Key", "type": "string", "versions": "0+",
  "about": "The coordinator key." },
{ "name": "KeyType", "type": "int8", "versions": "1+", "default": "0", "ignorable": false,
  "about": "The coordinator key type.  (Group, transaction, etc.)" }
```

#### INIT_PRODUCER_ID

```
{ "name": "TransactionalId", "type": "string", "versions": "0+", "nullableVersions": "0+", "entityType": "transactionalId",
  "about": "The transactional id, or null if the producer is not transactional." },
{ "name": "TransactionTimeoutMs", "type": "int32", "versions": "0+",
  "about": "The time in ms to wait before aborting idle transactions sent by this producer. This is only relevant if a TransactionalId has been defined." },
{ "name": "ProducerId", "type": "int64", "versions": "3+", "default": "-1",
  "about": "The producer id. This is used to disambiguate requests if a transactional id is reused following its expiration." },
{ "name": "ProducerEpoch", "type": "int16", "versions": "3+", "default": "-1",
  "about": "The producer's current epoch. This will be checked against the producer epoch on the broker, and the request will return an error if they do not match." }
```

#### PRODUCE

```
{ "name": "TransactionalId", "type": "string", "versions": "3+", "nullableVersions": "3+", "default": "null", "entityType": "transactionalId",
    "about": "The transactional ID, or null if the producer is not transactional." },
  { "name": "Acks", "type": "int16", "versions": "0+",
    "about": "The number of acknowledgments the producer requires the leader to have received before considering a request complete. Allowed values: 0 for no acknowledgments, 1 for only the leader and -1 for the full ISR." },
  { "name": "TimeoutMs", "type": "int32", "versions": "0+",
    "about": "The timeout to await a response in miliseconds." },
  { "name": "TopicData", "type": "[]TopicProduceData", "versions": "0+",
    "about": "Each topic to produce to.", "fields": [
    { "name": "Name", "type": "string", "versions": "0+", "entityType": "topicName", "mapKey": true,
      "about": "The topic name." },
    { "name": "PartitionData", "type": "[]PartitionProduceData", "versions": "0+",
      "about": "Each partition to produce to.", "fields": [
      { "name": "Index", "type": "int32", "versions": "0+",
        "about": "The partition index." },
      { "name": "Records", "type": "records", "versions": "0+", "nullableVersions": "0+",
        "about": "The record data to be produced." }
    ]}
  ]}
]
```

#### TXN_OFFSET_COMMIT

```
{ "name": "TransactionalId", "type": "string", "versions": "0+",
  "about": "The ID of the transaction." },
{ "name": "GroupId", "type": "string", "versions": "0+", "entityType": "groupId",
  "about": "The ID of the group." },
{ "name": "ProducerId", "type": "int64", "versions": "0+", "entityType": "producerId",
  "about": "The current producer ID in use by the transactional ID." },
{ "name": "ProducerEpoch", "type": "int16", "versions": "0+",
  "about": "The current epoch associated with the producer ID." },
{ "name": "GenerationId", "type": "int32", "versions": "3+", "default": "-1",
  "about": "The generation of the consumer." },
{ "name": "MemberId", "type": "string", "versions": "3+", "default": "",
  "about": "The member ID assigned by the group coordinator." },
{ "name": "GroupInstanceId", "type": "string", "versions": "3+",
  "nullableVersions": "3+", "default": "null",
  "about": "The unique identifier of the consumer instance provided by end user." },
{ "name": "Topics", "type" : "[]TxnOffsetCommitRequestTopic", "versions": "0+",
  "about": "Each topic that we want to commit offsets for.", "fields": [
  { "name": "Name", "type": "string", "versions": "0+", "entityType": "topicName",
    "about": "The topic name." },
  { "name": "Partitions", "type": "[]TxnOffsetCommitRequestPartition", "versions": "0+",
    "about": "The partitions inside the topic that we want to committ offsets for.", "fields": [
    { "name": "PartitionIndex", "type": "int32", "versions": "0+",
      "about": "The index of the partition within the topic." },
    { "name": "CommittedOffset", "type": "int64", "versions": "0+",
      "about": "The message offset to be committed." },
    { "name": "CommittedLeaderEpoch", "type": "int32", "versions": "2+", "default": "-1", "ignorable": true,
      "about": "The leader epoch of the last consumed record." },
    { "name": "CommittedMetadata", "type": "string", "versions": "0+", "nullableVersions": "0+",
      "about": "Any associated metadata the client wants to keep." }
  ]}
]}
```

#### ADD_PARTITIONS_TO_TXN

```
{ "name": "TransactionalId", "type": "string", "versions": "0+", "entityType": "transactionalId",
  "about": "The transactional id corresponding to the transaction."},
{ "name": "ProducerId", "type": "int64", "versions": "0+", "entityType": "producerId",
  "about": "Current producer id in use by the transactional id." },
{ "name": "ProducerEpoch", "type": "int16", "versions": "0+",
  "about": "Current epoch associated with the producer id." },
{ "name": "Topics", "type": "[]AddPartitionsToTxnTopic", "versions": "0+",
  "about": "The partitions to add to the transaction.", "fields": [
  { "name": "Name", "type": "string", "versions": "0+", "mapKey": true, "entityType": "topicName",
    "about": "The name of the topic." },
  { "name": "Partitions", "type": "[]int32", "versions": "0+",
    "about": "The partition indexes to add to the transaction" }
]}
```

#### ADD_OFFSETS_TO_TXN

```
{ "name": "TransactionalId", "type": "string", "versions": "0+", "entityType": "transactionalId",
  "about": "The transactional id corresponding to the transaction."},
{ "name": "ProducerId", "type": "int64", "versions": "0+", "entityType": "producerId",
  "about": "Current producer id in use by the transactional id." },
{ "name": "ProducerEpoch", "type": "int16", "versions": "0+",
  "about": "Current epoch associated with the producer id." },
{ "name": "GroupId", "type": "string", "versions": "0+", "entityType": "groupId",
  "about": "The unique group identifier." }
```

#### END_TXN

```
{ "name": "TransactionalId", "type": "string", "versions": "0+", "entityType": "transactionalId",
  "about": "The ID of the transaction to end." },
{ "name": "ProducerId", "type": "int64", "versions": "0+", "entityType": "producerId",
  "about": "The producer ID." },
{ "name": "ProducerEpoch", "type": "int16", "versions": "0+",
  "about": "The current epoch associated with the producer." },
{ "name": "Committed", "type": "bool", "versions": "0+",
  "about": "True if the transaction was committed, false if it was aborted." }
```

## Changes

### Pulsar Changes

1. Add interface `newTransaction(TxnID txnId)` in `TransactionMetadataStore`

### API Changes

### Protocol Changes

### Configuration Changes

## Compatibility

## Test Plan

## Future Work

1. support read-uncommitted messages for Pulsar and KoP.

## Refer To

1. KIP Transaction [https://docs.google.com/document/d/11Jqy_GjUGtdXJK94XGsEIK7CP1SnQGdp2eF0wSw9ra8/edit#heading=h.mcphg8e8gg24](https://docs.google.com/document/d/11Jqy_GjUGtdXJK94XGsEIK7CP1SnQGdp2eF0wSw9ra8/edit#heading=h.mcphg8e8gg24)
1. Kafka Transaction blog [https://www.confluent.io/blog/transactions-apache-kafka/](https://www.confluent.io/blog/transactions-apache-kafka/)
1. Offset Related [https://juejin.cn/post/6844904035368058894](https://juejin.cn/post/6844904035368058894)