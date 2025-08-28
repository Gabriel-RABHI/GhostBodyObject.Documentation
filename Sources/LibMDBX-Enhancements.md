# Evolutions of MDBX
## Missing features
There is 3 features that needs to be added to LibMDBX, as a layer or as a modification of the library, directly:
1) A faster **single find** Get() function, to get a DbValue from a DbKey : it may need a new feature in MDBX to track all changes to maintains an fast lookup.
2) A **concurrent Write Transaction** (with optimistic and pessimistic conflict check) : it can be done "over" MDBX as an encapsulated new transaction class.
3) A **live rebuild** of the entire database to compact the database and get a optimal key cursor operations.
## Single Find
### The need
I have to manage "objects" that have relations. Objects are stored in DBIs per type, with Id as Key, and the object representation as Value. Relations can be of two types :
- **1 -> N** : an object have a collection of Id to another objects. In this case, the relations are recorder as an index, a specific DBI where each key is composed of few fields :  `<src_key_id><direction><relation_type><dst_key_id>`. To enumerate all the objects in the internal collection I have to use a cursor to enumerate all keys, and retrieve the DbValue for each `<dst_key_id>` using a Get() call.
- **N -> 1 / 1 -> 1** : an object have a single pointer to another object. In this case, the object have ID of the target directly in his data. To retrieve the pointed object, a single Get() is necessary.
In both cases, a single, direct hit Get() is done, for each collection entry for internal object collections, and a single time for 1 <-> 1 relations.

Using MDBX, a single retrieve is all time a B+Tree traversal in OLog(n). Compared to a HashTable, it is rapidly "slow". A HashTable is almost 5, 10 to 20+ time faster for a single retrieve ! The memory hit is a lot more limited and specific use case optimizations can be done using specific properties of the ID format I'm using.

The idea is to maintains a `HashTable`, in real-time, to search the DbValue first in the HashTable and then, via a MDBX Get() call if not in the HashTable. This HashTable is then used as a cache mechanism.

This may be a globally applied mechanism, but may be more efficient to be used only on the "object" tables.

### A generationnal HashTable
For each Id, the HashTable will maintain a list of pointers that correspond to generations (= the last comited write transaction id) :

- When a Read Transaction is opened:
	- The current generation user count is incremented.
	- When a retrieve operation is done, first, we search in the HashTable :
		- if there is no entry, a Get() is performed on the MDBX transaction. If found, a new entry is created and the DbValue found is associated to the current generation in the HashTable entry.
		- If there is an entry, we take the pointer associated to the Transaction generation or previous generation if one. The generation corresponding to the current Write transaction is ignored. If a pointer is associated to a generation that is no more necessary, this entry can be recycled.
		  
	- When the last read transaction is closed for a given generation, the pointers values associated to this generation can be removed on the next Get() operation. The maximal generation that need to be preserved is the lowest one of all opened transactions.
	  
- When a Write Transaction is opened:
	- The retrieve is following the same logic than the Read Transactions. When find in the HashTable, the maximal generation pointer is taken.
	- When a Key/Value pair is updated, the new Pointer is set as the current Write Transaction generation.

The probable result is a limited generation count per entry, while the number of concurrent Read Transactions have to be maintained low.

### The problem
To build and maintains this HashTable, it is needed to track ALL key/value pairs movements : for the one explicitly done using the MDBX transaction Put() calls, it is easy. But the fact is that it not sufficient, because non explicitly changed key/value pairs can be relocated for various reasons:
- Leaf split, grouping or rebalance.
- Compaction.

### The need for a change notification
For this reason, it is necessary to have something that looks like a callback, that is called every time a Key/Value pair is implicitly moved. This is only true for Write Transactions. This callback may send both the DBI, the DbKey and the DbValue that have changed, or a list of changes. It may be a range of keys (a start and end key).

For exemple, if a leaf is modified, all the key/value pairs of the parent node is a good approximation.

When the callback is called, she update all pointers (DbValue) for each key in the HashTable.

This callback do not have to be extremely precise : if some notified key/value pairs are notified changed while not the case, it is not a problem. The ideal goal is to do not notify extremely large set of keys to update when only few have really changed.

If the DBI is not available in the callback, it is not too much a problem because the DbKey contains the type identifier to maintains the right HashTable (the one of the right DBI).

## Concurrent Write Transaction
To come.
## Complete Database Live Rebuild
To come.