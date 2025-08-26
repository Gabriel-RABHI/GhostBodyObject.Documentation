# Quick Overview
## What you must know
The GBO technology permit to manage large - or huge, up to terabytes - C# object collections. This objects can be added to .Net standard collections or added to a dedicated Ghost Repository, a GBO transactional object store.

A Repository can be persisted on disk or still in memory. All objects are accessed at memory speed, without latency. There is no "lazy-loading" and no loading operations : you access objects nearly as fast as in any .Net standard collection. When stored on disk, a Repository do not need any reloading, all objects are streamed transparently from the disk - even if the total size of objects collections are many time larger than the available physical memory.

When inserted in a Repository, object relations can be managed: an object can have references to another objects or can have internal collections of object references. As an example, an Order object can have a collection of OrderLine, and each OrderLine can have a reference to their Order parent object. Constraints can be defined to maintain high data coherency :
- Cascading deletion : when we delete en object, a graph of linked object is deleted too.
- Deletion interdiction: when some objects still referenced by another one, this object cannot be deleted.
- Cardinality rules : 1-1, 1-N, N-1 and N-N relations between objects can be established.
- Indexes : indexes can be simple or composed, and used for fast search (exact hit and range queries) and unicity checks.

This features guarantee a strong data coherency and avoid orphans. When created, relations are generating edges between objects that are useful for graph oriented processing : relations search, path finding, ranking based on relations.

The GBO library manage multiple version of each object type using converter for descendant and ascendant object compatibility. Version conversions are transparently done on the fly. It support built-in fixed or floating master-slave replication.

All this features combined permit to use GBO as a powerful, extremely-high-performance database engine.

## Foundational Principles
A GBO objet is split in two distincts parts : 
- The **Ghost** part is a memory block that contains the values of the fields of each object.
- The **Body** part is a class that permit to manage a **Ghost**. Fields values are accessed trough the getter and setter of each **Body** class member.

An **Object** is the combination of both technical concepts. When an object is a member of a complete object web with complexe relationships, each object can be defined as an **Entity**. An **Entity** is generally a concept in the application, like an `Order` with `OrderLine` and `Product` in a selling system.

A **Ghost** is a simple, black-box memory block managed by the GBO library. To create, save, retrieve and modify objects, and to manage Ghosts relationships, the Body class is used.

## Define the Entities
To start to use GBO, include the package `GhostBodyObject` and `GhostBodyObject.SourceGenerator` in your project. Then, you can define a GBO object :

```c#
using GhostBodyObject;

[ObjectBody]
[Public]
public interface MUserAccount {
	public string FirstName {get;set;}
	
	public string LastName {get;set;}
	
	public DateTime BirthDate {get;set;}
}
```

Then you can use the generated Body class :

```c#
var account = new UserAccount() {
	FirstName = "Gabriel",
	LastName = "Rabhi",
	BirthDate = new DateTime(1975, 08, 02)
};

Console.WriteLine(account);

```

Few rules must be followed :
- A GBO object is defined using an interface. This model interface name must start with "M", for "Model". This interface is exclusively used to define a GBO model. It does not define any contract.

The interface is generated with the concrete implementation of the model : both the interface and the Body class are partial declaration, that enable to add code.

```c#
public partial interface IUserAccount {
	int Age { get; }
}

public partial class UserAccount {
	int Age => (int)(DateTime.Now - BirthDate).TotalYears;
}
```

## Use a Ghost Repository
A Ghost Repository is a store that retains the Ghost part of an object. To insert a Ghost, you have to use a Body instance.

```c#
using (var repo = new GhostRepository()) {
	// Add a Ghost
	using (var tnx = repo.OpenWriteTransaction()) {
		tnx.Add(new UserAccount() {
			FirstName = "Gabriel",
			LastName = "Rabhi",
			BirthDate = new DateTime(1975, 08, 02)
		});
		tnx.Commit();
	}
	// Retreive
	using (var tnx = repo.OpenReadTransaction()) {
		Console.WriteLine("Count = " + tnx.Enumerate<UserAccount>().Count(u => u.LastName == "Rabhi"));
	}
}
```

Note that a Repository is a transactional store that support **Write Transactions** and **Read Transactions**. The default **Write** transaction are serialized, while a lot of **Read** Transactions can be opened concurrently. Write do not block read transactions, and vice-versa. Write Transactions can be opened with `TrasactionMode.OptimisticConcurrent` (check write conflict) or `TransactionMode.PessimisticConcurrent` (check that queries performed during the transaction are valid).

A Repository can be volatile or persistent. A path can be specified to define where to store the memory mapped files needed to manage Ghosts.

Each time a Ghost is added to a Repository, it is added to a dedicated internal collection, a **Ghost Table**. Each object type had his dedicated table. In the `Enumerate<TBody>` method, each time a new Ghost is retrieved, a new Body class is created that is linked to the Ghost.

To perform mutations in the Repository, a Write Transaction must be opened. To retrieve an object, you can use both a Write Transaction or a Read Transaction.

An object (the Body and an associated Ghost) can have few statues :
- The Body is linked to a **Standalone** Ghost : it is the case when you create a new Body class. The Body implicitly create a standalone Ghost. A standalone Ghost can be changed using members's getter and setters.
- The Body is linked to a Ghost that is **Mapped** in a Repository :
	- The **Ghost** was retrieved trough a Write Transaction or was recently added to a Repository : this object is editable - we can change the values of the fields trough member's getter and setter.
	- The **Ghost** war retrieved using a Read Transaction : this object is read-only, any attempt to modify it throw an `ReadOnlyGhost` exception.

Each Body have a Context member, that provide the same feature than Write or Read Transactions. If the Ghost is standalone, the Context is null.

```c#
var body = (IBody)account;
var context = body.Context;
if (context == null) {
	if (body.ReadOnlyl)
		Console.WriteLine("In an undefined containner");
	else
		if (body.Standalone)
			Console.WriteLine("Is standalone");
		else
			Console.WriteLine("Undefined ghost state");
} else {
	if (context is IWriteTransaction)
		Console.WriteLine("Write Transaction");
	else
		if (context is IReadTransaction)
			Console.WriteLine("Read Transaction");
		else
			Console.WriteLine("Undefined context");
}
```

In any code, the Context can be retrieved to perform mutations :

```c#
void AddFriend(UserAccount dst, string fn, string ln) {
	if (((IBody)dst).Context is WriteTransaction wtnx)
		wtnx.Add(new UserAccount() {
			FirstName = fn,
			LastName = ln
		});
	else
		throw new WriteContextNeeded();
}
```

## Object Ghost vs Structure Ghost
The GBO library permit to generate two types of objects :
- The Object Ghost is an object that can be inserted in a Ghost Repository as an Entity because a globally unique identifier is created and assigned to each object when created. The Identifier is accessible via the `IObjectBody` contract `Id` member of type `GhostId`.
- The Structure Ghost is an object that do not have an identifier: it is only a value carier. It can be inserted in a Repository in a **Queue**, a **Stack**, a **Set** or a **Map**.

A **Queue** (`Queue<T>`) is a table that support `Enqueue`, `Dequeue` methods. Here a sample of a enqueued message structure :

```c#
[StructureBody]
[Public]
public interface MOrderValidatedMessage {
	public GhostId OrderId {get;set;}
}

using (var repo = new GhostRepository()) {
	using (var tnx = repo.OpenWriteTransaction()) {
		var order = tnx.Enumerate<Order>().OrderBy(o => o.Edited).Last();
		var queue = tnx.GetOrCreateQueue<IBody>("msg-bus");
		queue.Enqueue(new MOrderValidatedMessage() { OrderId = ((IobjectBody)order).Id });
		tnx.Commit();
	}
}
```

A **Stack** (`Stack<T>`) is a table that support `Push`, `Pop`.

A **Set** (`Set<T>`) is a table that support `Add`, `Remove`, `Contains`.

A **Map** (`Map<TKey, TValue>`) collection is a Key-Value collection that support `Set`, `KeyExist`, `Find`, `FindNextNeighbor`, `FindPreviousNeighbor`.


