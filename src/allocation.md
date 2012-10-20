    #!/bin/bash

# Introduction

This file provides allocation, reference handling, and garbage collection.
Bash-lambda allocates objects into a common directory in the temporary
directory. Each running process maintains a separate directory containing its
own references; these directories are treated as parts of the root set for the
purposes of garbage-collecting the shared heap.

Objects in the shared heap are generally immutable once allocated. They are
also named after a hash of their contents (see the manpage for git-hash-object
for details). This second property is important: it means, for practical
purposes, that two files with the same name will have the same contents. This,
in turn, means that copying an object from one heap to another is idempotent,
and it makes it possible to share heap objects across machines via SSH.

    alias bash-lambda-hash='git hash-object --stdin'

# Reference semantics

There are four ways for a shell process to maintain a reference to an object in
the shared heap:

    1. Define a variable that points to it (e.g. f=$heap_object_name)
    2. Start a process that knows about it (e.g. cat $heap_object_name)
    3. Reference it in the text contents of any file in its local heap
    4. Reference it as the symlink destination of any file in its local heap

All of these situations are garbage collector root cases: as long as the shell
is running, any object referred to by any of these methods will remain alive.

    alias bash-lambda-gc-roots='(declare; ps ax)'

# Shared heap GC

Any running shell can start a GC in the shared heap. In order to do this, it
needs to grab the `.bash-lambda-gc` directory lock. If this directory exists,
any shell that allocates an object must symlink the object into this directory.
This informs the running GC that the object was allocated after the root set
was retrieved, so the GC will conservatively consider the new allocation to be
live whether it really is or not.

    declare -r heap_prefix=${TMPDIR:-/tmp}/bash-lambda-$(whoami)
    declare -r shared_heap=$heap_prefix/shared
    declare -r local_heap=$heap_prefix/$(hostname)-$$

    bash-lambda-mkheap() { mkdir -p $1; ln -s . $1/.weak; touch $1/.alive; }
    bash-lambda-mkheap $shared_heap
    bash-lambda-mkheap $local_heap

# Local heap expiration

When the shell exits, we need to mark the local heap as being available for
collection. We don't nuke it immediately, since it is vaguely possible that
someone else is referring to it (like a background job, for instance). Rather,
we remove its .alive file so that the shared GC knows that the heap's objects
are no longer roots.

    bash-lambda-free-local-heap() { rm $local_heap/.alive; }
    trap bash-lambda-free-local-heap EXIT

# Object allocation

Objects are allocated as files into the global heap, and symlinks are created
from local heaps to mark the global objects as live. You allocate stuff by
calling the 'new' function:

    $ seq 1 10 | new
    /tmp/bash-lambda-spencertipping/hostname-5012/f00c965d...f162
    $ l=$(seq 1 10 | new)
    $ $map $($fn x 'echo $((x + 1))') $l
    2
    3
    ...
    11
    $

Unlike prior versions, this version of bash-lambda does not integrate reference
types into the allocation strategy. Runtime type information is encoded as a
higher-level abstraction.

The 'new' function accepts an optional argument specifying the allocation
directory. This argument defaults to the local heap.

    new() { declare temp=$(mktemp ${1:-$local_heap}/new-XXXXXXXXXXXX); cat > $temp
            declare name=$(git hash-object $temp)
            touch $shared_heap/.bash-lambda-gc/$name >& /dev/null && \
              ln -s $shared_heap/$name $local_heap/ && \
              mv $temp $shared_heap/$name && chmod 555 $shared_heap/$name && \
              echo $local_heap/$name; }