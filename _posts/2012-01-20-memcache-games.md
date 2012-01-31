---
layout: post
title: "Memcache Games - Feeds"
categories: 
  - memcache
date: Jan 30, 2012
time: "01:29"
date_time: "30 Jan, 2012 - <a href='http://pinterest.com/martaaay/'>Marty Weiner</a>"
---

We love data feeds.  Our whole site is one giant feed.  User feeds, pin feeds, board feeds,
feeds of pins inside boards, feeds of users who follow a board, feeds of followers who
pin on boards created by users with feeds.  Ok, scratch that last one.

We also love Redis, which for the uninitiated is cache + non-volatility + data structures (sets, lists, etc).
But, you pay extra RAM for the privilege of having these data structures.  In some of our calculations, lists cost as little as 2x
overhead and as much as 9.6x overhead.  But Redis is simple, and fast, and works.

At the same time, our feeds tend to be simple *mostly* FIFO.  What I mean is, we insert numbers to the front of a list
and rarely remove from the middle.  If we don't need any other list features or persistence, we'd like to avoid the overhead of Redis lists if possible.

We could use Memcache and store compressed lists of integers and maybe save on RAM.  But Memcache doesn't have special data structures --
it's a straight key-value store.  Inserts require pulling the compressed list out, modifying it, and putting it back in.

<center>&lt;suspense><b>Or do they?</b>&lt;/suspense></center><br><br>

<br>
So, we cheated.  Memcache does have an *append* operator which we use to append integers onto a value.  We just
need a separator:

    class MemcacheList(object):
      def push(self, key, value):
        """ Add an element to the front of the list """
        self.connection.append(key, value+",")
    
      def get(self, key):
        """ Get the list """
        return self.connection.get(key).split(",")
    
      def set(self, key, values):
        """ Set the list """
        self.connection.set(key, ",".join(values))

Why *append* and not *add*?  One feature of *append* is that it does **NOT** create a new cache entry on key miss.  Who cares?!
Suppose you introduce your new cache into production and view your list of Star Wars toys.  Cache is empty so you grab the list from the database and set the cache.  Great!  But suppose you add
Chewbacca to your list of toys before reading the empty cache.  You write the new toy to the database and 
add it into the cache.  Now the cached list just has Chewbacca in it.  Upon reading that list again, there is only one entry in it.  Fortunately, *append* will
not create the entry on miss.  It will silently fail and the cache row will not be created
until you read the entire list from the database and *set* it in cache.

Memcache supports binary values.  So instead of storing raw lists of comma separated values, we [MessagePack](http://msgpack.org/) our data.
MessagePack converts integers into lightly compressed binary form that is crazy fast and fairly space efficient.  It supports lists and
has out-of-the-box iteration of lists.  On average, our 64-bit IDs take 5 bytes to store with MessagePack.  The revised code looks like this:

    class MemcacheList(object):
      def push(self, key, value):
        """ Add an element to the front of the list """
        packed = msgpack.packb(value)
        self.connection.append(key, packed)
    
      def _unpack(self, data):
        if data == '\x90':
          return [], 0
    
        _unpacker = msgpack.Unpacker()
        _unpacker.feed(data)
        elems = []
        num_elems = 0
        for val in _unpacker:
          if val:
            num_elems += 1
            elems.append(val)
        return elems, num_elems
    
      def get(self, key):
        """ Get the list """
        data = self.connection.get(key)
        if not data: return None
    
        elems, num_elems = self._unpack(data)
    
        return list(reversed(elems))
    
      def set(self, key, values):
        """ Set the list """
        s = ""
        for x in reversed(values):
            s += msgpack.packb(x)
        self.connection.set(key, s)

We get a bit fancier, but this is the core of our Memcache List.

This scheme puts more burden on removing an element from a list.  Our remove is not atomic (requires read-modify-write) and is relatively slow since we have to 
get the list, find the element to remove, and set the modified list back to cache.  You could use CAS to achieve atomicity, but our data and use cases
are not that strict and we prefer to keep the algorithm simple.  Additionally, many of our data access patterns are 99.9% or higher appends and 0.1%
removes, so we're cool with pushing more pain to the remove function.

      def remove(self, key, value):
        data = self.get(key)
        if not data: return False
    
        if elem in data:
            data.remove(value) # Uses python's list.remove
            self.set(key, data)
            return True
    
        return False

Alternatively, you could simply make remove invalidate the cache line -- the penalty being added DB pressure.

### Quick Afterward - Why not use Redis Strings instead of Memcache?

Because Redis strings don't support an append operation that fails on miss.  Also, we tend toward Memcache when
possible.  It's much more developmentally stable and simple -- it's basically memory attached sockets.



