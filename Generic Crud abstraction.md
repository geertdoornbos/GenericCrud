# Standardized Generic CRUD Abstraction

## 1 Introduction

As most software engineers know, the acronnym CRUD stands for Create, Read, Update, Delete. These are four functions for simple persistance and has some reflection to database operations (*INSERT, SELECT, UPDATE, DELETE*). The same similarities can be found on RESTful services; HTTP methods (*POST, GET, PUT/PATCH, DELETE*).

For various reasons, a generic abstraction for these functions can be valuable. For instance, it can simplify development due to repeatability of known behavior. Automatic code generation and testing can also come into view.

Implementations of such abstraction can only be succesful if usage and behavior are crisp and clear and standardized.

There have been many attempts to create generic repository abstractions and implementations in the last two decades. Still, there is no standard. It's a fragmented landscape. Anyone who has worked on this can confirm that CRUD is some sort of part of a so called 'repository pattern'. Let's start by abstracting CRUD and make it implementation- and technology agnostic.

## 2 Characteristics

CRUD operation will always refer to one type of object and just a single object. All operations are related / depending on each other. An object is only readable once it has been created. An object is not created if there is no intention to read it back later. An object can only be updated if it has been created before. An object can only be deleted if it has been created before.

The type of object is unknown and unrestricted in this generic approach. It's required to have the exact same object state when reading an object as it was before, when creating or updating the object.

The CRUD operations assume the implementation to persist a collection of objects of the same type. Nothing is known about the storage technology behind implementations, but it is probably I/O related.

The CRUD operations do not enable the retrieval of a subset of the persisted collection unless the object **is** a collection. Also, CRUD operations do not enable querying for subsets of data of the collection.

The CRUD operations do not specify durability aspects of the persisted collection. All CRUD operations are effective immediately.

All CRUD operations use a reference or key to store or change stored objects and retrieve them back later using that key of reference. The same for applies to deletion.

Keys, as reference to an object, can be defined before the create operation or they are issued by the implementation (for instance by auto generated identity columns in databases). The correct key will always be returned by the create operation.
When an object has been created, a key has been predefined or issued by the implementation. If the key is issued by the implementation, the key must be encapsulated inside the object (property).
The implementation is responsible for maintaining the relation between the (unique) key and the object.
The caller is responsible for the key.

Since there is a key, it must be able to read the object back again. Inspection or retrieving (sub)sets of the collection is not a part of the CRUD interface.

Since there is a key, the object can be deleted. After deleting, the key is of no use anymore and should be used anymore.

## 3 Interface specification

The CRUD abstraction is defined as an generic interface. This way it can be implemented by any class.
Due to the characteristics, the interface as a whole is generic and not the methods. The same applies to the keys.

<!-- ![CRUD interface](diagrams/images/Crud%20interface.png) -->
<br />
    <div align=center>
        <img src="diagrams\images\Crud%20interface.png" />
    </div>
<br />

To store an object a key is needed to uniquely identify the object. This key needs to be supplied to each of the CRUD operations. For easy of use, this is often a member of the object itself, but it's not required. If the key is stored inside the object itself, the implementation could add another interface for querying objects and keys. That is not part of the CRUD operations.

<!-- ![](diagrams/images/Crud%20and%20Queryable%20interface.png) -->
<br />
    <div align=center>
        <img src="diagrams\images\Crud%20and%20Queryable%20interface.png" />
    </div>
<br />

The CRUD operations require real objects and real keys; they must not be nullable or be null.

The interface should not have ***no prior knowledge*** to any implementation.

```csharp
public interface ICrud<T, TKey>
    where T: notnull 
    where TKey : notnull
{
    Task<TKey> CreateAsync(T @object, TKey? key = default, CancellationToken cancellationToken = default);

    Task<T> ReadAsync(TKey key, CancellationToken cancellationToken = default);

    Task UpdateAsync(TKey key, T @object, CancellationToken cancellationToken = default);

    Task DeleteAsync(TKey key, CancellationToken cancellationToken = default);
}
```

Each method says what it does and each method must do what it says. It's not forgiving; the method either succeeds or fails with an exception. Failure should be exceptional.

The methods are asynchronous, specified by return type Task. Since implementations presumably will not complete synchronously due to storage nature, ValueTask has not been chosen. This also minimizes interface usage specifications related to [awaiting and concurrency](https://devblogs.microsoft.com/dotnet/async-valuetask-pooling-in-net-5/).

The CRUD operations can be cancelled using the cancellation token. This is a [common method](https://learn.microsoft.com/en-us/dotnet/standard/threading/cancellation-in-managed-threads) to cancel tasks.

```text
Remarks:
- The generic type T can be a collection type, but then the whole collection is referred to by a single key.
- Implementations can choose to make this interface part of a (database) transaction or **unit of work** scenario's. That should not change any aspect of this CRUD abstraction.
- The interface behavior is not idempotent. If an idempotent behavior is required, a processing service can be used prior to the CRUD interface.
```

### 3.1 CreateAsync

```csharp
Task<TKey> CreateAsync(T @object, TKey? key = default, CancellationToken cancellationToken = default);
```

#### 3.1.1 CreateAsync behavior

Calling this method will *create* (or *store*) the object given in the **@object** argument. The **key** argument suggests the key to be used as a reference to the object. When the key is ommitted, the implementation must generate a key. Even if **key** is supplied, the implementation always determines the key to be returned.  
The method always returns the key witch can be used later on to retrieve the object again.

If the **CreateAsync** operation succeeds, a key has been returned. The caller should immediately be able to call:

- ReadAsync with the same key to get the object back in the same state.
- UpdateAsync with the same key to store the change the object.
- DeleteAsync with the same key to remove the object from the collection.

The **cancellationToken** should be used by the implementation.

#### 3.1.2 CreateAsync exceptions

If the **@object** parameter is null the execution should fail by throwing exception:

- System.ArgumentNullException("Argument **@object** of type *TypeName* is null which is not allowed.")

If the **key** parameter is null and the implementation can not issue keys, the execution should fail by throwing exception:

- System.ArgumentNullException("Argument **key** is required. The implementation cannot issue key's.")

If the **key** parameter is supplied and the object also contains the key then they should be identical otherwise the execution should fail by throwing exception:

- System.ArgumentNullException("Argument **key** does not match the object's key.")

If the object cannot be created (*stored*) due to the fact that there already is an object with specified key the execution should fail by throwing exception:

- System.ArgumentException("An object of type *TypeName* with the same key has already been created. Key: *key*)

When the **cancellationToken** parameter is applied, the method can fail with exception:

- TaskCanceledException

Example:

```csharp
public class Student
{
    public Guid Id { get; set; }
    public string Name { get; set; }
}

ICrud<Student, Guid> crudStudents = GetCrudStudents();
```

Create operation with the key passed to the method:

```csharp
var student = new Student 
{
    Id = Guid.NewGuid(),
    Name = "John Doe"
};

var key = await crudStudents.CreateAsync(student, student.Id);
```

Create operation with the key issued by the implementation:

```csharp
var student = new Student();
student.Name = "John Doe";

Guid key = await crudStudents.CreateAsync(student);
student.Id = key;
```

### 3.2 ReadAsync

```csharp
Task<T> ReadAsync(TKey key, CancellationToken cancellationToken = default);
```

#### ReadAsync behavior

Calling this method will *read* the object given in the **key** argument. Read is not an attempt to find an object. If there is a key, it's most likely the operation will succeed, otherwise, the key should not exist.

If the **ReadAsync** operation succeeds, the object referred to by the given key has been returned. The caller should immediately be able to call:

- UpdateAsync with the same key to store the change the object.
- DeleteAsync with the same key to remove the object from the collection.

The **cancellationToken** should be used by the implementation.

#### 3.2.2 ReadAsync exceptions

If the specified **key** is null the execution should fail by throwing exception:

- System.ArgumentNullException("Key of type *TypeName* is null which is not allowed.")

When the **cancellationToken** parameter is applied, the method can fail with exception:

- TaskCanceledException

### 3.3 UpdateAsync

```csharp
Task UpdateAsync(TKey key, T @object, CancellationToken cancellationToken = default);
```

#### 3.3.1 UpdateAsync behavior

If the **UpdateAsync** operation succeeds, the changed object referred to by the given key has been stored. The caller should immediately be able to call:

- ReadAsync with the same key to get the updated object back in the newly state.
- UpdateAsync with the same key to store a changed object again.
- DeleteAsync with the same key to remove the object from the collection.

The **cancellationToken** should be used by the implementation.

#### 3.3.2 UpdateAsync exceptions

If the specified **key** is null the execution should fail by throwing exception:

- System.ArgumentNullException("Key of type *TypeName* is null which is not allowed.")

If the **@object** parameter is null the execution should fail by throwing exception:

- System.ArgumentNullException("Argument **@object** of type *TypeName* is null which is not allowed.")

When the **cancellationToken** parameter is applied, the method can fail with exception:

- TaskCanceledException

### 3.4 DeleteAsync

```csharp
Task DeleteAsync(TKey key, CancellationToken cancellationToken = default);
```

#### 3.4.1 DeleteAsync behavior

If the **DeleteAsync** operation succeeds, the object referred to by the given key has been removed. The caller should not use the key anymore.

The **cancellationToken** should be used by the implementation.

#### 3.4.2 DeleteAsync exceptions

If the specified **key** is null the execution should fail by throwing exception:

- System.ArgumentNullException("Key of type *TypeName* is null which is not allowed.")

When the **cancellationToken** parameter is applied, the method can fail with exception:

- TaskCanceledException
