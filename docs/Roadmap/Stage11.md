---
title: "Stage 11: Index Creation and Deletion"
---

# Stage 11: Index Creation and Deletion (20 hours)

:::note Learning Objectives

- Implement the creation and insertion operations on a B+ tree on the XFS disk
- Implement the deletion of an index from the XFS disk

:::

:::tip PREREQUISITE READING

- [B+ Trees](../Misc/B%2B%20Trees.md)
- [Indexing in NITCbase](../Misc/Indexing.md)

:::

## Introduction

You must now already be familiar with the usage of indexes to speed up search operations. An index can be created on any attribute of a relation using the [CREATE INDEX](../User%20Interface%20Commands/ddl.md#create-index) command. Once an index is created on the attribute, all search operations involving that attribute will proceed through the index instead of a linear search through all the records. An index can be deleted with the [DROP INDEX](../User%20Interface%20Commands/ddl.md#drop-index) command. Note that NITCbase does not allow you to create/delete indexes for the relation catalog and the attribute catalog.

Thus far, index creation and deletion could only be done through the XFS Interface. In this stage, we will implement this functionality in NITCbase.

## Implementation

When an index is created for an attribute of a relation, the `RootBlock` field in the corresponding attribute catalog entry will have to be updated with the block number of the root block of the B+ tree. In practice, this value is updated in the attribute cache and then written back to the buffer (and subsequently the disk) when the relation is closed. Thus, we will need to implement attribute cache update and write back in the [Cache Layer](../Design/Cache%20Layer/intro.md).

In the [Buffer Layer](../Design/Buffer%20Layer/intro.md), we will need to update the [IndLeaf](../Design/Buffer%20Layer/IndBuffer.md#class-indleaf) and [IndInternal](../Design/Buffer%20Layer/IndBuffer.md#class-indinternal) classes to implement the `setEntry()` function we discussed in the previous stage. This function allows us to write to an index block.

A sequence diagram showing the call sequence involved in the implementation of index creation and deletion are shown below.

> **NOTE**: The functions are denoted with circles as follows.<br/>
> 🔵 -> methods that are already in their final state<br/>
> 🟢 -> methods that will attain their final state in this stage<br/>

```mermaid
 %%{init: { 'sequence': {'mirrorActors':false} } }%%
sequenceDiagram
    actor User
    participant Frontend User Interface
    participant Frontend Programming Interface
    participant Schema Layer
    participant B-Plus Tree Layer
    participant Cache Layer
    participant Buffer Layer
    User->>Frontend User Interface: CREATE INDEX
    activate Frontend User Interface
    Frontend User Interface->>Frontend Programming Interface: create_index()🟢
    activate Frontend Programming Interface
    Frontend Programming Interface->>Schema Layer: createIndex()🟢
    activate Schema Layer
    Schema Layer->>B-Plus Tree Layer: bPlusCreate()🟢
    activate B-Plus Tree Layer
    B-Plus Tree Layer->>Buffer Layer: IndLeaf()🔵
    Buffer Layer-->>B-Plus Tree Layer: blockNum (member field)
    loop for every record of the relation
      B-Plus Tree Layer->>Buffer Layer: getRecord()🔵
      activate Buffer Layer
      Buffer Layer-->>B-Plus Tree Layer: record
      deactivate Buffer Layer
      B-Plus Tree Layer->>B-Plus Tree Layer: bPlusInsert()🟢
      activate B-Plus Tree Layer
      B-Plus Tree Layer->>Buffer Layer: setEntry()🟢
      activate Buffer Layer
      Buffer Layer-->>B-Plus Tree Layer:operation status
      deactivate Buffer Layer
      deactivate B-Plus Tree Layer
    end
    B-Plus Tree Layer-->>Schema Layer: operation status
    deactivate B-Plus Tree Layer
    Schema Layer-->>User: operation status
    deactivate Schema Layer
    deactivate Frontend Programming Interface
    deactivate Frontend User Interface
```

- todo: discuss where cache updates

### Cache Update and Write-back

An index can only be created for an open relation. When an index is created for a relation on an attribute, the `RootBlock` field is set for the corresponding attribute catalog entry in the attribute cache entry of the relation. Similar to how we had implemented the updation of the relation cache in previous stages, this updated value will be written to the buffer when the relation is closed (or at system exit, when all open relations are closed.).

Thus, we will need to implement methods to write to the attribute cache. Additionally, we will also modify the `OpenRelTable::closeRel()` function to write-back any _dirty_ attribute cache entries on relation closing. Note that we do not need to update the destructor of the OpenRelTable class to handle write-back for the attribute cache entries of the relation catalog and the attribute catalog (why?).

A class diagram of the [Cache Layer](../Design/Cache%20Layer/intro.md) highlighting the methods relevant to this functionality is shown below.

```mermaid
classDiagram
direction BT
  RelCacheTable <|.. OpenRelTable : friend
  AttrCacheTable <|.. OpenRelTable : friend
  class RelCacheTable{
    -relCache[MAX_OPEN] : RelCacheEntry*
    -recordToRelCatEntry(union Attribute record[RELCAT_NO_ATTRS], RelCatEntry *relCatEntry)$ void🔵
    -relCatEntryToRecord(RelCatEntry *relCatEntry, union Attribute record[RELCAT_NO_ATTRS])$ void🔵
    +getRelCatEntry(int relId, RelCatEntry *relCatBuf)$ int🔵
    +setRelCatEntry(int relId, RelCatEntry *relCatBuf)$ int🔵
    +getSearchIndex(int relId, RecId *searchIndex)$ int🔵
    +setSearchIndex(int relId, RecId *searchIndex)$ int🔵
    +resetSearchIndex(int relId)$ int🔵
  }
  class AttrCacheTable{
    -attrCache[MAX_OPEN] : AttrCacheEntry*
    -recordToAttrCatEntry(union Attribute record[ATTRCAT_NO_ATTRS], AttrCatEntry *attrCatEntry)$ void🔵
    -attrCatEntryToRecord(AttrCatEntry *attrCatEntry, union Attribute record[ATTRCAT_NO_ATTRS])$ void🟢
    +getAttrCatEntry(int relId, int attrOffset, AttrCatEntry *attrCatBuf)$ int🔵
    +getAttrCatEntry(int relId, char attrName[ATTR_SIZE], AttrCatEntry *attrCatBuf)$ int🔵
    +setAttrCatEntry(int relId, char attrName[ATTR_SIZE], AttrCatEntry *attrCatBuf)$ int🟢
    +setAttrCatEntry(int relId, int attrOffset, AttrCatEntry *attrCatBuf)$ int🟢
    +getSearchIndex(int relId, char attrName[ATTR_SIZE], IndexId *searchIndex)$ int🔵
    +getSearchIndex(int relId, int attrOffset, IndexId *searchIndex)$ int🔵
    +setSearchIndex(int relId, char attrName[ATTR_SIZE], IndexId *searchIndex)$ int🔵
    +setSearchIndex(int relId, int attrOffset, IndexId *searchIndex)$ int🔵
    +resetSearchIndex(int relId, char attrName[ATTR_SIZE])$ int🔵
    +resetSearchIndex(int relId, int attrOffset)$ int🔵

  }
  class OpenRelTable{
    -tableMetaInfo[MAX_OPEN] : OpenRelTableMetaInfo
    +OpenRelTable(): 🔵
    +~OpenRelTable(): 🔵
    -getFreeOpenRelTableEntry()$ int🔵
    +getRelId(char relName[ATTR_SIZE])$ int🔵
    +openRel(char relName[ATTR_SIZE])$ int🔵
    +closeRel(int relId)$ int🟢
  }

```

<br/>

In class `AttrCacheTable`, we implement the method `setAttrCatEntry()` which is overloaded to find the entry in the attribute cache using the attribute's name or offset.

We also implement the `attrCatEntryToRecord()` function which converts from a [struct AttrCatEntry](../Design/Cache%20Layer/intro.md#attrcatentry) to a record of the attribute catalog (a [union Attribute](../Design/Buffer%20Layer/intro.md#attribute) array). This function will be useful when writing the dirty cache entry back to the buffer in record form.

<details>
<summary>Cache/AttrCacheTable.cpp</summary>

Implement the following functions looking at their respective design docs

- [`AttrCacheTable::setAttrCatEntry(relId, attrOffset, attrCatEntry)`](../Design/Cache%20Layer/AttrCacheTable.md#attrcachetable--setattrcatentry)
- [`AttrCacheTable::setAttrCatEntry(relId, attrName, attrCatEntry)`](../Design/Cache%20Layer/AttrCacheTable.md#attrcachetable--setattrcatentry)
- [`AttrCacheTable::attrCatEntryToRecord()`](../Design/Cache%20Layer/AttrCacheTable.md#attrcachetable--attrcatentrytorecord)

</details>

In class `OpenRelTable`, we update `closeRel()` to check for dirty attribute cache entries and write them back to the buffer using `attrCatEntryToRecord()` and the [Buffer Layer](../Design/Buffer%20Layer/intro.md) functions we are familiar with.

<details>
<summary>Cache/OpenRelTable.cpp</summary>

Implement the `OpenRelTable::closeRel()` function by looking at the [design docs](../Design/Cache%20Layer/OpenRelTable.md#openreltable--closerel).

</details>

### Writing to Index Blocks

In the previous stage, we had talked about the abstract class [IndBuffer](../Design/Buffer%20Layer/IndBuffer.md#class-indbuffer) and it's children [IndLeaf](../Design/Buffer%20Layer/IndBuffer.md#class-indleaf) and [IndInternal](../Design/Buffer%20Layer/IndBuffer.md#class-indinternal) representing leaf index blocks and internal index blocks respectively. We had implemented the `getEntry()` function in both the classes to read an entry from an index block. In this stage, we will implement the `setEntry()` function which allows us to write an entry to an index block.

A class diagram of the [Buffer Layer](../Design/Buffer%20Layer/intro.md) highlighting these functions is shown below.

```mermaid
classDiagram
    direction TB
    StaticBuffer <|.. BlockBuffer : friend
    BlockBuffer <|-- IndBuffer
    IndBuffer <|-- IndInternal
    IndBuffer <|-- IndLeaf
    class StaticBuffer{
        -blocks[BUFFER_CAPACITY][BLOCK_SIZE]: unsigned char
        -metainfo[BUFFER_CAPACITY]: struct BufferMetaInfo
        -blockAllocMap[DISK_BLOCKS]: unsigned char
        +StaticBuffer() 🔵
        +~StaticBuffer() 🔵
        -getFreeBuffer(int blockNum)$ int🔵
        -getBufferNum(int blockNum)$ int🔵
        +setDirtyBit(int blockNum)$ int🔵
        +getStaticBlockType(int blockNum)$ int🔵
    }
    class BlockBuffer{
        #blockNum: int
        +BlockBuffer(char blockType) 🔵
        +BlockBuffer(int blockNum) 🔵
        +getHeader(struct HeadInfo *head) int🔵
        +setHeader(struct HeadInfo *head) int🔵
        +releaseBlock() void🔵
        #setBlockType(int blockType) int🔵
        #getFreeBlock(int blockType) int🔵
        #loadBlockAndGetBufferPtr(unsigned char **buffPtr) int🔵
    }
    class IndBuffer{
        +IndBuffer(char blockType): 🔵
        +IndBuffer(int blockType): 🔵
        +getEntry(void *ptr, int indexNum)* int
        +setEntry(void *ptr, int indexNum)* int
    }
    class IndInternal{
        +IndInternal(): 🔵
        +IndInternal(int blockNum): 🔵
        +getEntry(void *ptr, int indexNum) int🔵
        +setEntry(void *ptr, int indexNum) int🟢
    }
    class IndLeaf{
        +IndLeaf(): 🔵
        +IndLeaf(int blockNum): 🔵
        +getEntry(void *ptr, int indexNum) int🔵
        +setEntry(void *ptr, int indexNum) int🟢
    }

```

<br/>

Implement these functions as shown in the links below.

<details>
<summary>Buffer/BlockBuffer.cpp</summary>

Implement the following functions looking at their respective design docs

- [`IndLeaf::setEntry()`](../Design/Buffer%20Layer/IndBuffer.md#indleaf--setentry)
- [`IndInternal::setEntry()`](../Design/Buffer%20Layer/IndBuffer.md#indinternal--setentry)

</details>

### Creating and Deleting B+ Trees

In the [B+ Tree Layer](../Design/B%2B%20Tree%20Layer.md), we implement methods to create a B+ tree, insert into a B+ tree and free all the index blocks associated with a B+ tree. Note that we do not need to implement the functionality to delete an individual entry from a B+ tree (why?).

A class diagram highlighting all the functions that will be modified to implement this functionality is shown below.

```mermaid
classDiagram
  class Schema{
    +openRel(char relName[ATTR_SIZE])$ int🔵
    +closeRel(char relName[ATTR_SIZE])$ int🔵
    +renameRel(char oldRelName[ATTR_SIZE], char newRelName[ATTR_SIZE])$ int🔵
    +renameAttr(char relName[ATTR_SIZE], char oldAttrName[ATTR_SIZE], char newAttrName[ATTR_SIZE])$ int🔵
    +createRel(char relName[ATTR_SIZE], int numOfAttributes, char attrNames[][ATTR_SIZE], int attrType[])$ int🔵
    +deleteRel(char relName[ATTR_SIZE])$ 🔵
    +createIndex(char relName[ATTR_SIZE], char attrName[ATTR_SIZE])$ int🟢
    +dropIndex(char relName[ATTR_SIZE], char attrName[ATTR_SIZE])$ int🟢
  }
```

```mermaid
classDiagram
  class BlockAccess{
    +linearSearch(int relId, char attrName[ATTR_SIZE], Attribute attrVal, int op)$ RecId🔵
    +renameRelation(char oldName[ATTR_SIZE], char newName[ATTR_SIZE])$ int🔵
    +renameAttribute(char relName[ATTR_SIZE], char oldName[ATTR_SIZE], char newName[ATTR_SIZE])$ int🔵
    +insert(int relId, union Attribute* record)$ int🟢
    +deleteRelation(char relName[ATTR_SIZE])$ int🟢
		+project(int relId, Attribute *record)$ int🔵
    +search(int relId, Attribute *record, char attrName[ATTR_SIZE], Attribute attrVal, int op)$ int🔵
  }
```

```mermaid
classDiagram
  class BPlusTree{
    +bPlusSearch(int relId, char attrName[ATTR_SIZE], Attribute attrVal, int op)$ int🔵
    +bPlusCreate(int relId, char attrName[ATTR_SIZE])$ int🟢
    +bPlusInsert(int relId, char attrName[ATTR_SIZE], Attribute attrVal, RecId recId)$ int🟢
    +bPlusDestroy(int rootBlockNum)$ int🟢
  }
```

<br/>

As shown in the sequence diagram above, the Frontend User Interface will parse the `CREATE INDEX` command and call the `Frontend::create_index()` function in the Frontend Programming Interface. This call is then transferred along to the [Schema Layer](../Design/Schema%20Layer.md). Hence, the implementation of the `Frontend::create_index()` function only involves a call to the `Schema::createIndex()` function. Similarly, the `DROP INDEX` command leads to the `Frontend::drop_index()` function which in turn transfers control to `Schema::dropIndex()`.

<details>
<summary>Frontend/Frontend.cpp</summary>

Implement the following functions looking at their respective design docs

- [`Frontend::create_index()`](../Design/Frontend.md#frontend--create_index)
- [`Frontend::drop_index()`](../Design/Frontend.md#frontend--drop_index)

</details>

The `Schema::createIndex()` function verifies that the relation is open and passes the rel-id and attribute name along to the `BPlusTree::bPlusCreate()` function to create an index. We will implement this function later in this stage.

The `Schema::deleteIndex()` function fetches the root block of the index on a specified attribute from the attribute cache and then calls the `BPlusTree::bPlusDestroy()` function to free the index blocks.

todo: is the cache updated

<details>
<summary>Schema/Schema.cpp</summary>

Implement the following functions looking at their respective design docs

- [`Schema::createIndex()`](../Design/Schema%20Layer.md#schema--createindex)
- [`Schema::dropIndex()`](../Design/Schema%20Layer.md#schema--dropindex)

</details>

In the [Block Access Layer](../Design/Block%20Access%20Layer.md), we update the `insert()` method to insert the new record into any existing indexes of the relation using `BPlusTree::bPlusInsert()`. The `deleteRelation()` method is updated to free up any indexes associated with the relation using `BPlusTree::bPlusDestroy()`.

<details>
<summary>BlockAccess/BlockAccess.cpp</summary>

Implement the following functions looking at their respective design docs

- [`BlockAccess::insert()`](../Design/Block%20Access%20Layer.md#blockaccess--insert)
- [`BlockAccess::deleteRelation()`](../Design/Block%20Access%20Layer.md#blockaccess--deleterelation)

</details>

Lastly, we implement the core functionality of this stage in the [B+ Tree Layer](../Design/B%2B%20Tree%20Layer.md).

<details>
<summary>BPlusTree/BPlusTree.cpp</summary>

Implement the following functions looking at their respective design docs

- [`BPlusTree::bPlusCreate()`](../Design/B%2B%20Tree%20Layer.md#bplustreebpluscreate)
- [`BPlusTree::bPlusInsert()`](../Design/B%2B%20Tree%20Layer.md#bplustreebplusinsert)
- [`BPlusTree::bPlusDestroy()`](../Design/B%2B%20Tree%20Layer.md#bplustreebplusdestroy)

</details>

And with that, your NITCbase now supports the creation and deletion of indexes! Verify your implementation with the exercises below.

## Exercises