# Considerations on options for sharing memory data among different matlab application instances

Underlying concepts (?): System V shared memory operations, `man shm_overview`, `/dev/shm`, `man icps`.

## [matlab's own `memmapfile`](https://www.mathworks.com/help/matlab/import_export/share-memory-between-applications.html)

- Requires that one copy of the data resides on a disk file. That solves the problem of an unique pointer
  to it visible by all instances, and perhaps some access mutexing, but demands i/o.

Wrong use model for us. The data we want to share is not the product we want to store permanently on file.
Our shared data is written once and read once only.

Forget it.

# [Xuebin Zhou, shared_matrix](https://github.com/qhgz2013/shared_matrix)

- compiles easily (run `compile.m`)
- creates an object `shared_matrix_host` for each individual shared variable, containing the pointer, which
  needs to be transmitted to all parties
- the example given is for a `parfor` pool, which handles by itself the problem of sharing the object
- the object could be shared perhaps using our Messengers, which transmit is as structure, and recasting
  the structure to the `shared_matrix_host` object. I didn't succeed though, see my example below.
- Perhaps, that could work augmenting the class with a method for populating the object from a
  plain structure of its fields. Otherwise, contraptions like saving the structure to a file and reloading...
- only supports numeric or logic matrices, meaning that every other variable type will have to be flattened.
  Integer matrices shared are retrieved as double by `.get_data()`.
- I managed to crash it at first attempt, but probably only because I was careless with multiple calls without
  proper cleanup of old pointers. Worked within its limitations, otherwise.


My example:

In master:
```
S=obs.util.SpawnedMatlab; S.spawn; S.connect
a=rand(3,3)
smh=shared_matrix_host(a)
save('/tmp/smh.mat',smh)
...
clear shm
```
In spawned matlab
```
addpath /home/enrico/Eran/shared_memory_matlab/shared_matrix
s=MasterMessenger.query('smh')
smh=shared_matrix_host(1) % just to create it somehow
smh.Name=s.Name; smh.Handle=s.Handle; smh.BasePointer=s.BasePointer
  You cannot set the read-only property 'Name' of 'shared_matrix_host'.
load('/tmp/smh.mat')
accessor=smh.attach
a=accessor.get_data()
...
accessor.detach
```

Timing:
```
>> a=uint16(rand(6000,9000)*65536);
>> tic;smh=shared_matrix_host(a);toc
Elapsed time is 0.102258 seconds.
```

## [Joshua Dillon's sharedmatrix](https://www.mathworks.com/matlabcentral/fileexchange/28572-sharedmatrix)

- compiles easily running `sharedmatrix_install.m`
- has a single calling function `sharedmatrix()` with different keys for different tasks
- see `help sharedmatrix` for detailed info
- the pointer required is a simple numeric key - can easily be transmitted via Messengers
- besides numeric and logic matrices it can share char arrays and cells containing mixtures of the previous two.
  Even cells whose elements are cells themselves. Not structures or objects, but it's already something.
- It is also easy to crash matlab (e.g. `double free or corruption (out)`) if sharedmatrix is not freed
  before being recloned, or detached after being read once with 'attach'.
- timing for cloning a variable of the size of an image is non negligible. Timing for attaching is:

```
a=uint16(rand(6000,9000)*65536);
tic;shmsiz=sharedmatrix('clone',111111,a);toc;
Elapsed time is 0.091034 seconds.
tic;c=sharedmatrix('attach',111111);toc
Elapsed time is 0.000507 seconds.
sharedmatrix('detach',111111,c)
sharedmatrix('free',111111)
```

## [Gene Harvey's matshare](https://github.com/gharveymn/matshare)

[Also on matlabcentral](https://www.mathworks.com/matlabcentral/fileexchange/68161-matshare?s_tid=ta_fx_results)

- compile with `source/INSTALL.m`. This gives a warning though: "_Compiling in compatibility mode;
  the R2018aMEX API may not support certain functions which are integral to this function_"
- it is much more complete than the previous two. It includes locks, implementation of in place
  operations (including IIUC `.overwrite` subarrays of shared matrices, which might be a way for us
  to implement a circular image buffer)
- can do structures
- the `.data` field of a fetched `matshare` object is linked, i.e. all the objects in all sessions 
  see the changing values
- has the option of including own variable names to the share
- but there seems to be only _one_ persistent share for all attached processes (hence no need to
  pass keys, but also the drawback of no private sharespace).
- interestingly, all variables pushed to the share with `matshare.share` remain there till cleaned
- there are options for getting the first pushed variable (LIFO), the variables in the order they were
  pushed (not a FIFO, but may have some application)
- I suspect that `.clearshm` and `.mshreset` are buggy and crash matlab,
  but it may be my misuse. But if that is the case, there is no way to clean the share, except for overwriting...

Timing:
```
>> tic;matshare.share(a),toc

ans = 

  matshare object storing 6000x9000 uint16:
    data: [6000×9000 uint16]

Elapsed time is 0.100741 seconds.
```

# first impression's conclusion

I'd probably leverage on *matshare*, but there is much of its features to ponder carefully for a viable pipeline
workers implementation. And if I can't get round the clearing bug, I'd go for *sharedmatrix* as a second choice,
though it is much more basic.
