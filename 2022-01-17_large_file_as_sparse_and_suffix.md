# Simulating a large file using a sparse file + suffix file: Tezos usecase



We want to simulate a large file using a sparse file and a suffix file.

```
[sparse file][suffix file]
             ^suffix file offset
```

So, the sparse file accounts for a range $(0,len)$, and the suffix file accounts for $(len,...)$.

The general situation is that we have a sparse file and a suffix file. In the beginning, we can model the suffix file as covering the range $(-1,0)$ (we never need to go to the suffix file until the first "freeze").

A "freeze" can be triggered when the suffix file is already quite full. And the commit for the freeze might reside quite early in the suffix file. 

```
[sparse][suffix ... commit ...data-after-commit... ]
                    ^freeze triggered for this commit
```

Here we want the data-after-commit to live in the new suffix file.

In this case, we can either copy the data-after-commit to a new file, or somehow start a new file and remember that part of the suffix actually lives in the old file. It seems easiest just to copy the data.

```
[sparse][suffix ... commit data-after-commit]
  
  becomes 2 new files
  
[new sparse upto commit][new suffix, data-after-commit onwards]
```

And we can delete the old `sparse` and `suffix` files.

In order not to block, we can construct the new-sparse in parallel, whilst still writing to the old suffix. Then we copy all the data-after-commit from the old suffix to the new (again, can be done in parallel, but at some point we probably block and copy the last remaining bit), and switch over.

Now, exposing file descriptors is potentially dangerous, because they are invalid when we switch over. So the suffix file should probably provide some interface that is file-like, but doesn't use fds. Something like "append_bytes" (which is what they currently ue I think).

In addition, there is a need to record the **last flushed position** in order to recover from crashes on the append-only file.

