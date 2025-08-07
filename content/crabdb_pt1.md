+++
title = "Building CrabDB Part 1"
date = 2025-08-07
+++

This is the first post on my journey in building a NoSQL key value database known as CrabDB. An impoprtant anecdote is that CrabDB is in early stages of
development so whatever is stated in this blog post can and most likely will change. This post should be treated more as a point in time snapshot of CrabDB
in its current state.

Also the code show in this blog post will be mostly devoid of any real error handeling and such things all code is just use for illustrative purposes if you
want to see the actual code go to the CrabDB repo.

# About CrabDB

A couple of months ago I decided that I wanted build a database from scratch so that I could better understand how the internals of these programs
I use every day work. I made the following decisions on my database:
- It will be a NoSQL database as at the time I just wasn't in the mood to make a SQL compiler.
- I will use no external dependencies since I one do really like programming and two I think I will learn a lot more if I code everything by hand. So in general
if it is not offered by the languages standard library I won't use it.
- The database engine will be programmed in [Rust](https://www.rust-lang.org/) so that is can be **blazingly fast 🔥** and since it currently is my favourite programming
language.

Now after I decided all this I made the decision that I would be making a key/value database since I recently read a blog post on implementing a concurrent hashmap and
thought it would be cool to build one myself. I then named the database CrabDB since "crab" sounds like "grab" and crab is nice and rusty.

Without further adieu lets get into the meet of the blog.

# CrabDB Internals

CrabDB is split into the following main sections:
- object: Concerns itself with how objects are stored and represented as well as what data types can be stored in the database.
- server: This system is responsible for listening for connections as well as receiving data and sending data back.
- storage: All logic pertaining to how data is stored in the database is handeled here.
- engine: Everything comes together here and this can be thought of the manager of all the other systems, it contains the `main.rs` file. 
- There are other miscellaneous crates litered around but they are not integral to the functionality but will still be described in this post.

## Object

An object is an item stored in CrabDB for now the following types that I want CrabDB to support are as follows:
- String (known as Text in CrabDB)
- Int
- List
- Map (aka json object, aka dictionary, aka record, aka struct)

This is where the most amount of changes and rewrites occured in these initial stages of the database and at the current rate it will be rewritten again at some point.
Since I had to suffer with my own insanity and indecisiveness I will explain to all the readers every implementation, I will even number them for clarity.

### Attempt 1 - Enums

The most natural way to model what I wanted to achieve in Rust with an enum with a varient per data type as follows:
```rust
struct Text { /* ... */}
struct Int { /* ... */}
struct List { /* ... */}
struct Map { /* ... */}

enum Object {
  Text(Text),
  Int(Int),
  List(List),
  Map(Map),
}
```

This is exactly what I did and it worked very well. All data is recieved as bytse over Tcp so a payload could very easily be structured with a single byte for the type
followed by the bytes to build the object so the code to serialize an deserialize the objects is as follows:

```rust
impl Object {
  fn serialize(&self) -> Vec<u8> {
    // This sucks though that I am just constantly repeating code
    match self {
      Self::Text(obj) => obj.serialize(),
      Self::Int(obj) => obj.serialize(),
      Self::List(obj) => obj.serialize(),
      Self::Map(obj) => obj.serialize(),
    }
  }

  fn deserialize(bytes: &[u8]) -> Self {
    let type_id = bytes[0];

    match type_id {
      0 => Self::Text(Text::from_bytes(&bytes[1..]))
      1 => Self::Int(Int::from_bytes(&bytes[1..]))
      2 => Self::List(List::from_bytes(&bytes[1..]))
      3 => Self::Map(Map::from_bytes(&bytes[1..]))
      _ => // error no type
    }
  }
}
```

This is great and its easy plus its super simple for future Michael to go and debug. An each varient has its own struct that handles the underlying logic. It is also
important to note at this time data would be deserialized and stored in the deserialized format (this will be important later).

### Attemp 2 - Boxes, Traits and Dynamic Dispatch

I am a programmer so I love to struggle and suffer and why do things the easy way when we can just suffer instead. Essentially what happend was I had the idea: what if
the user wants to create a custom type with some format. Now I could just add another varient to the enum to handle this but instead what if we made an Object trait.

```rust
trait Object {
  fn serialize(&self) -> Vec<u8>;
}
```

Have `Text`, `Int`, `List` and `Map` implement that then we make use of dynamic dispatch with `Box<dyn Object>` and then create an `ObjectFactory` in which you can register
types with function to build them from a byte slice which then returns a `Box<dyn Object>`.

```rust
type TypeId = u8;

// This won't compile its missing higher ranked trait bounds and the object should really also have at least Send implemented
struct ObjectFactory {
  factories: HashMap<TypeId, Box<dyn Fn(&[u8]) -> Box<dyn Object>>> 
}

impl ObjectFactory {
  fn add_factory(&mut self, type_id: TypeId, factory: Box<dyn Fn(&[u8]) -> Box<dyn Object>>) -> Box<dyn Fn(&[u8]) -> Box<dyn Object>> {
    // code
  }

  fn create(&mut self, type_id: TypeId, bytes: &[u8]) -> Box<dyn Fn(&[u8]) -> Box<dyn Object>> {
    // code
  }
}
```

Now the ObjectFactory was very cool and I had it has a "singleton" but it was difficult to work worth and the whole time I was doubting whether it was even worth besides
difficulty in implementing there were other easier wasys to achieve this that would also probably be faster since as a general rule of thumb static dispatch with enums
is almost always better.

**sigh**

### Attemp 3 - Enums... again

In the end everything was complicated the code while not spaghetti would give me headache while looking at it so I rewrote it back to the original enum implementation
but I also though while I was at it I will delete all the other code I had effectively starting from scratch. And you know what while all what I did was go back to the
original implementation somehow coding it from scratch resulted in less code and more importantly cleaner code. A good rule is you will always code something better the second,
third, ...,  nth time better.

### Internal Repersentation

Originally all data when recieved was desiarlized into the actual type it was. What this essentially means is that data would be recieved in raw bytes and I would convert it
to what it was defined as for example a list would be deserialized to a type like this:

```rust
struct List {
  items: Vec<Object> // remember the object definition from above
}
```

This has some benefits for example if you want to perform operations on data types they are alread in the correct format. Naturally though this does have downsides. Firstly
the above representation is less memory effcient and also deserializing costs time so the whole database operation is slower and you end up paying this cost twice once when
you deserialize and again when you need to serialize to send it back over Tcp. These costs are even worse when you consider that types like Map and Lists can contain maps
and lists themselves so they are recursive types.

Due to the above considerations there was one more major change to the Object crate and that was to instead store that data as a flat byte array with a tag specifying what
data type is being stored. If any operations are now to be performed the data has to be first deserialzed and then re-serialized which obviously also has a cost but considering
the most common operation on a database is reads having data be sent to clients faster due to no serializing needing to take place is worth it. The current implementation looks
something like this:

```rust
enum ObjectKind {
  Text,
  Int,
  List,
  Map,
}

struct Object {
  object_kind: ObjectKind,
  data: Vec<u8>,
}
```

## Server

This part of the database is responsible for listening for client connections, recieving and sending data. The networking is not all that interesting as all what it
requires is designing a protocol to send and receive data and deciding what that protocol is build off of. CrabDB uses Tcp as the underlying protocol since it has low
overhead (relative to http) and is pretty fast. As all good network protocols work they start with stating the how much data is being sent, this is then followed up
by the command that is being executed (GET, SET, etc), this is then followed up by the type id (Text, Int, etc) and finally the actual data for the type if there is
any. A full overview of the protocol can be found on the GitHub for CrabDB.

In general as you can see the networking part is relatively boring and as of yet I haven't made any cool optimisations but the following commands are supported:
- GET: gets an object
- SET: sets an object
- DELETE: deletes and objet
- CLOSE: closes the connection

## Storage

This is again where things become more interesting. CrabDB is designed to be a hyper performent database so that means most if not all the data should be kept in memory.
To this end the initial implementation of CrabDB only supported in memory storage with zero persistence. First a trait was defined:

```rust
trait Storage {
  fn set(&mut self, key: &str) -> Option<Object>; // returns the previous object under that key if there is one

  fn get(&self, key: &str, object: Object) -> Option<Object>;

  fn delete(&mut self, key: &str) -> Option<Object>; // returns the deleted object
}
```

Remember this is not exactly how the trait looks in the actual implementation it is much more sophisticated there using a `Key` type as well as returning a `Result` for
better error handeling, the code shown in this blog post removes all the boring fluff.

### An in Memory Only Storage Solution

The most simply way to create a type that implements this trait is with a simple Rust `HashMap`.

```rust
struct InMemoryStorage {
  map: HashMap<&str, Object>,
}

impl Storage for HashMap {
  // ... you can probably figure out what goes here
}
```

And while this works and is fast its a little bit sad because its single threaded and multithreaded code is always better right? right? So now you want your code to
be multithreaded so you slap on a mutex or even better a reader writers lock and call it a day. Unfortunetly at your grand database unveiling people notive that
if they have a task with many mixed reads and writes your database starts taking a bit too long to well do anything. We in the business call that **contention**.
If the problem is not immediately obvious what is happening is that your database performs fine when you write infrequently and almost always perform reads but when you
write a lot you need to take a writers lock which blocks all other access to the data. Now you end up going to down a rabbit hole trying to figure out how to
implement a truly concurrent hash map since if you recall we are not allowed to use third party packages and everything must be home grown. **sigh**. In your travels
of the wider internet you keep coming across people mentioning (which I must assume at this point) Java's legendary concurrent hash map. As an aside this irks me, since
I think Java is one of the worst programming languages ever made.

Here is a rundown of a simple implementation of a concurrent hashmap, if you would like a more fully cable concurrent hashmap I would recommend checking out [dashmap](https://github.com/xacrimon/dashmap),
[flashmap](https://github.com/Cassy343/flashmap) or even if you must [Java's concurrent hashmap](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentHashMap.html).

```rust
struct ConcurrentHashMap<K, V> {
  // The trick is to have a list of hashmaps
  shards: Vec<RwLock<HashMap<K, V>>>,
}

impl<K, V> ConcurrentHashMap<K, V> {
    // Creates a new concurrent hashmap with the specified number of shards
    pub fn new(num_shards: usize) -> Self {
        let mut shards = Vec::with_capacity(num_shards);

        for _ in 0..num_shards {
            shards.push(RwLock::new(HashMap::default()));
        }

        Self { shards }
    }
}

impl<K, V> ConcurrentHashMap<K, V>
where
    K: Hash + Eq, // required for hashmap
{
    // Gets the shard index that a key belongs to
    fn shard_index(&self, key: &K) -> usize {
        let mut hasher = DefaultHasher::new();
        key.hash(&mut hasher);
        hasher.finish() as usize % self.num_shards()
    }

    // Gets the shard that a key belongs too
    fn shard(&self, key: &K) -> &RwLock<HashMap<K, V>> {
        // getting the index that the key belongs in
        let shard_index = self.shard_index(&key);
        // getting shard
        let shard = self.shards[shard_index] };

        shard
    }

I    // Inserts a new element into the ConcurrentMap
    // and returns the element was already there if one exists
    pub fn insert(&self, key: K, value: V) -> Option<V> {
        // getting the shard
        let shard = self.shard(&key);
        // inserting the value
        let mut map = shard.write().unwrap();
        map.insert(key, value)
    }

    /// Reads an element from the map and returns None if no element is found
    pub fn get<'a>(&'a self, key: &K) -> Option<Ref<'a, K, V>> where V: Clone{
        // getting the shard
        let shard = self.shard(key);

        // getting the element
        let map_guard = shard.read().unwrap();
        map_guard.get(key).cloned()
    }
}
```

I left out the delete since it is not complicated ot implement as well as any other niceties. It its now obvious the trick to having a concurrent hashmap work
is by having a list of non concurrent hashmaps each protected by a reader writer lock so that multiple hashmaps can be accessed concurrently. The actual
implmentation is very simple; first hash the key has you would then just mod it by the number of shards you have to decide which shard this key belongs too. The ideal number
of shards though is a heuristic that must be tested to determine the best number or an even cooler way to do it would be to have it dynamically grow and shrink.
This also makes use of [interior mutability](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html) so only a `&self` needs to be taken and the `Storage`
trait is also updated to only use `&self`'s so that it can safely be shared across threads and the underlying type must make sure of the safety gaurentees.

Now they above implementation does have some issues that a more seasoned concurrent hashmap wouldn't have for example:
- The get clones data, surely returning a reference would be better and it is, the actual implementation of CrabDB's  concurrent hashmap does this.
- The default Rust hasher is used but there are faster ones out there.
- If one hashmap keeps getting hit we essentially have our previous problem so the hashing algorithm used should try to split operations evenly between shards.

Overall this is a good enough first attempt at a concurrent hashmap to get CrabDB going, next we are going to go into persistence since losing all data on
restart does kind of suck.

### Persistant Storage

Being able to concurrently read and write data to memory is very cool but unfortunately many users would ideally want their data persisted between database restarts or
god forbid a crash. Typically there are two main mechanisms used for persisting data in an in memory database like memcached or redis.
- Append only file (AOF) which basically means any write operation (currently only set and delete) for us is written to a log file. On startup this log file can then be
read and the in memory part of the database can be recreated by sequentially applying the operations.
- Taking periodic snapshots which is just what it sounds like where you on some interval take a snapshot of the database.

These two solutions both have their pros and cons. AOF is very fast when writing since it is just appending to a log file but when you need to read the data back in it can
take variable amount of time as well as pointpless operations can occur for example:
```
  set key1 0
  del key1
  set key1 0
```
The same exact set is used twice and since a set occurs after the delete the delete is pointless. This can be negated by periodically compacting the log files but this then
pasuses all writes to the database for that time. Taking snapshots is much faster to read but while a snapshot is being created no writes to the database can occur and on
top of that any data after a snapshot is not saved until the next snapshot.

Most databases implement both of these where they will first load from a snapshot and only read the log for all data after the backup was taken offering the best of both worlds.

For the initial implementation AOF was implemented in which there are two options to implemnt which is what will be discussed below:

#### Multiple Files with Sharding

This is the current implementation that CrabDB uses it works in almost the same fashion as the concurrent hashmap where there are multiple files (shards) and
each shard in the hashmap maps to a file in a 1:1 manner. Since data is only written to the in memory storage after successfully being written to the file this
reduces contention. The biggest downside of this manenr of doing it is that the number of shards can not be changed since each file maps to a shard weird things
can happen if you are not careful. This does have the benefit that all files can be reingested in parallel.

#### Single File with Channel

This is most likely what CrabDB will switch to in the near future. Instead of having multiple files where a shard maps to a file each write operation sends the operation
and data done through a channel and a seperately running thread receives the sent data and writes it to a file. This allows for then changing the number of shards since there
is only a single source of truth. This does naturally have some downsides for example what is the upper limit of writing to a file (probably more then you think) and you can
no longer read data in concurrently.

## Engine

This crate is fairly boring and just acts as the glue for all the other crates, it creates the server using the server crate and then passes data to the storage crate
through the medium of the object crate. Yup thats it nothing special or really interesting.

## Other Crates

There are other crates involved but they are not considered as core as the crate already mentioned.

### Threadpool

This is just a threadpool that I implemented which pre-spawns a set number of threads and allows for queueing up jobs. Every time a client connects to the database a new
job is submitted to the threadpool for it.

### Concurrent Map

This is just an implementation of a concurrent map as described above.

# Conclusion

This is just the first part of a series and much will still change. This mainly made as a learning project for now but who knows what will happen. Check out the code on [github](https://github.com/MichaelDuPlessis/crabdb).
