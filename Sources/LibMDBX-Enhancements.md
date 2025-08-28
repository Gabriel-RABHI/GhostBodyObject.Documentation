# Proposed Enhancements for MDBX
## Proposed Features
There are three key features that need to be added to LibMDBX, either as a layer built on top of the library or as a direct modification to its core:

1. A faster **single-find** `Get()` function to retrieve a `DbValue` from a `DbKey`. This would likely require a new MDBX feature to track all data changes to maintain a fast, auxiliary lookup structure.
    
2. A **concurrent write transaction** model, with support for both optimistic and pessimistic conflict checking. This could be implemented as a new transaction class that encapsulates MDBX.
    
3. A **live rebuild** of the entire database to compact storage and ensure optimal performance for cursor operations.
    

## 1. Faster Single-Find `Get()`
### The Need
My application manages "objects" that have relationships with each other. Objects are stored in separate DBIs according to their type, using a unique ID as the key and a serialized object representation as the value.

These relationships are of two main types:

- **1-to-N**: An object has a collection of IDs pointing to other objects. These relationships are stored in a dedicated index (a separate DBI). Each key in this index is a composite key: `<src_key_id><direction><relation_type><dst_key_id>`. To enumerate all objects in the collection, I use a cursor to scan these keys and then perform a `Get()` call with the `<dst_key_id>` to retrieve each object's data.
    
- **N-to-1 / 1-to-1**: An object holds a direct pointer (the ID) to another object within its data. Retrieving the target object requires only a single `Get()` call.
    
In both scenarios, performance relies heavily on fast, direct `Get()` calls. With MDBX, a single retrieval is always a B+Tree traversal with **O(log n) complexity**. This is significantly slower than a hash table lookup, which can be **5, 10, or even 20+ times faster**. A hash table also has a much smaller memory footprint per lookup, and its performance can be further optimized based on the properties of my ID format.

The proposed solution is to **maintain a real-time `HashTable`** that acts as a cache. A lookup would first check this hash table; if the key is not found, it would fall back to a standard MDBX `Get()` call and add it to the lookup. This mechanism could be applied globally or, more efficiently, on a per-DBI basis for specific "object" tables.

### A Generational Hash Table
To handle MDBX's MVCC (Multi-Version Concurrency Control) model, the hash table would be generational. For each ID, it would store a list of pointers corresponding to different "generations," where a generation is identified by the transaction ID that created it.

Here's how it would work:

- **When a Read Transaction starts:**
    
    - The user count for the transaction's generation is incremented.
        
    - When retrieving a value, the hash table is checked first:
        
        - If there's no entry, a `Get()` is performed on the MDBX transaction. If the key is found, a new entry is created in the hash table, associating the `DbValue` pointer with the current transaction's generation.
            
        - If an entry exists, we use the pointer associated with the transaction's generation or the most recent previous generation. Pointers belonging to the current _write_ transaction's generation are ignored.
            
    - Once the last read transaction for a given generation closes, the pointers associated with that generation can be marked for cleanup. The oldest generation that must be preserved is determined by the oldest active read transaction.
        
- **When a Write Transaction starts:**
    
    - Data retrieval follows the same logic as in a read transaction.
        
    - When a key/value pair is updated (`Put()`), the new `DbValue` pointer is added to the hash table and associated with the current write transaction's generation ID.
        

This approach should result in a limited number of generations being stored per entry, as long as the number of concurrent read transactions remains reasonably low.

### The Problem

To reliably maintain this hash table, it's necessary to track **all** movements of key/value pairs. Tracking changes made explicitly through `Put()` calls is straightforward. However, this is insufficient because MDBX can implicitly relocate key/value pairs for internal reasons, such as:

- B+Tree page operations (splits, merges, or rebalancing).
    
- Database compaction.
    
### The Need for a Change Notification
To solve this, MDBX needs a notification mechanism - essentially a **callback** that is invoked during a write transaction whenever a key/value pair is implicitly moved.

This callback should ideally provide the DBI, the new `DbKey`, and the new `DbValue` pointer for each affected item. A notification covering a range of keys (e.g., a start and end key) could also be effective. For example, if a leaf page is modified, a notification for all key/value pairs in its parent node might be a sufficient approximation.

When this callback is invoked, it would update the corresponding pointers in the hash table.

The callback does not need to be perfectly precise; false positives (notifying of a change that didn't happen) are acceptable. The primary goal is to **avoid missing any changes** while also avoiding notifications for unnecessarily large sets of keys when only a few have been relocated.

### The Expected Result
With a such cache, the average Get operation delay may be **one order of magnitude faster** :
- From 1.5 M Get() to 15 M Get() per second.

The Write transactions may be up to **2  time slower** in average (not all induce massive splits, merges or rebalance). The memory consumption may be the given : 16 bytes for Key, 4 byte for gen, 16 bytes for DbValue. Approximately 48 bytes per entry. For 1M DB, 50MB of memory.

| Key/Value pairs | Avr. Size (1000 bytes per kvp) | Lookup Memory |
| --------------- | ------------------------------ | ------------- |
| 1 M             | 1 GB                           | 50 MB         |
| 10 M            | 10 GB                          | 500 MB        |
| 100 M           | 100 GB                         | 5 GB          |
| 1 B             | 1 To                           | 50 GB         |
This looks correct.

## 2. Concurrent Write Transaction
_(To be detailed later)_

## 3. Complete Database Live Rebuild
_(To be detailed later)_