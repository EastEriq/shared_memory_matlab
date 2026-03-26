# Considerations on options for sharing memory data among different matlab application instances

## matlabs own `memmapfile`

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
  plain structure of its fields. therwise, contraptions like saving the structure to a file and reloading...
- only supports numeric or logic matrices, meaning that every other variable type will have to be flattened.
  Integer matrices shared are retrieved as double by `.get_data()`.
- I managed to crash it at first attempt, but probably only because I was careless with multiple calls without
  proper cleanup of old pointers. Worked within its limitations, otherwise

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

## [Joshua Dillon's sharedmatrix](https://www.mathworks.com/matlabcentral/fileexchange/28572-sharedmatrix)

- compiles easily running `sharedmatrix_install.m`
- has a single calling function `sharedmatrix()` with different keys for different tasks
- see `help sharedmatrix` for detailed info
- the pointer required is a simple numeric key - can easily be transmitted via Messengers
- can share cells and char arrays


## [Gene Harvey's mathshare](https://github.com/gharveymn/matshare)


[Also on matlabcentral](https://www.mathworks.com/matlabcentral/fileexchange/68161-matshare?s_tid=ta_fx_results)


