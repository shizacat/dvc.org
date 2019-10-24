# Keep Data and Cache on External Drive

Sometimes the data may be stored on an
[external hard drive](https://whatis.techtarget.com/definition/external-hard-drive).
Usually such data is huge, which means that it won't fit on our local drive, and
even if it did, it would certainly take a long time to copy it back and forth
from the external drive to the internal one. For example let's say that the size
of the external drive is 16TB, while the local drive is only 320GB.

In this case we would like to process the data where it is already located (on
the external drive). We also would like to save the results there, and certainly
to store the <abbr>cached</abbr> files there as well.

<details>

Alternative solution: Initialize the workspace on the external drive itself

The easiest way to do this would be to initialize the workspace on the external
drive itself. If we assume that the external drive is mounted on
`/mnt/external-drive/` and the data is on `/mnt/external-drive/data/raw/`, then
it could be done like this:

```dvc
$ cd /mnt/external-drive/
$ git init
$ dvc init
$ dvc add data/raw/
...
```

But often this is not possible (or is not preferable). So we have to setup the
workspace in our local drive, while all the data files and their caches stay on
the external drive. This is the solution that is described in the rest of this
page.

</details>

## Make the data directory accessible

Let's assume that the external drive is mounted on `/mnt/external-drive/`. First
we have to make sure that we can read and write the data directory
`/mnt/external-drive/`. The most straightforward way to do this is by setting
proper ownership and permissions to it, like this:

```dvc
$ sudo chown <username>: -R /mnt/external-drive/
$ chmod u+rw -R /mnt/external-drive/
```

## Start a DVC project and setup an external cache

An [external cache](/doc/user-guide/external-outputs) is called so because it
resides outside of the workspace directory.

- Let's create a directory for it on `/mnt/external-drive/`:

  ```dvc
  $ mkdir -p /mnt/external-drive/dvc-cache
  ```

- Now you can initialize a <abbr>project</abbr> on your home directory:

  ```dvc
  $ cd ~/project/
  $ git init
  $ dvc init
  ```

- Configure it to use the external directory for caches:

  ```dvc
  $ dvc config cache.dir /mnt/external-drive/dvc-cache
  $ cat .dvc/config
  [cache]
  dir = /mnt/external-drive/dvc-cache
  ```

- Commit changes to git:

  ```dvc
  $ git add .dvc/config
  $ git commit -m 'Initialize DVC with external cache'
  ```

<details>

### Transfer the content of the cache to the external directory

In this example we can remove the default cache directory `.dvc/cache/` because
we just initialized the project and we know that it is empty (there's nothing
stored in it):

```dvc
$ rm -rf .dvc/cache/
```

If we had an existing project, we could preserve the content of the cache by
moving it to the new cache directory, before deleting the old one:

```dvc
$ mv -a .dvc/cache/* /mnt/external-drive/dvc-cache/
$ rm -rf .dvc/cache/
```

</details>

## Tracking external dependencies and outputs

Now, when you refer to the data files and directories, you have to use their
**absolute path**. The <abbr>DVC-files</abbr> will be created on the project
directory, and you can track their modifications with `git` as usual.

For example, let's say that the raw data files are on
`/mnt/external-drive/data/raw/` and you are cleaning them up. You could do it
like this:

```dvc
$ dvc add /mnt/external-drive/data/raw

$ dvc run \
      -f clean.dvc \
      -d /mnt/external-drive/data/raw \
      -o /mnt/external-drive/data/clean \
      ./cleanup.py \
          /mnt/external-drive/data/raw \
          /mnt/external-drive/data/clean
```

<details>

### Using an environment variable for the data path

In a real life situation probably you would declare an environment variable
`DATA=/mnt/external-drive/data` and use it to shorten the command options, like
this:

```dvc
$ dvc add $DATA/raw

$ dvc run -f clean.dvc -d $DATA/raw -o $DATA/clean \
          ./cleanup.py $DATA/raw $DATA/clean
```

</details>

If you check the contents of `raw.dvc` (and `clean.dvc`) you'll notice that the
`path` field refers to the external directories:

```yaml
md5: 9cbbacd47133debf91dcb41891c64730
wdir: .
outs:
  - md5: 0ee0a6bc0a1f1be0610f7a3f67f1cb54.dir
    path: /mnt/external-drive/data/raw
    cache: true
    metric: false
    persist: false
```

You can also check and verify that indeed all the data and cache files are
stored on the external drive:

```dvc
$ ls /mnt/external-drive/
data  dvc-cache

$ ls /mnt/external-drive/data/
clean  raw

$ ls /mnt/external-drive/dvc-cache
...
```

Now you can add and commit the DVC-files to git:

```dvc
$ git add .
$ git commit -m 'Cleanup raw data'
```

<details>

### Optimizing the data management

Since we are talking about large data, it is worth spending some time for
understanding
[how DVC can optimize data management](/doc/user-guide/large-dataset-optimization),
so that it does not make unnecessary copies of large data.

In short, if your external drive is formatted with XFS, Btrfs, ZFS, or any other
file system that supports reflinks, DVC will automatically use the most
efficient way of handling large datasets, and there is no further configuration
that needs to be done.

If _reflinks_ are not available, then you should consider setting the cache type
to _symlink_ or _hardlink_, like so:

```dvc
$ dvc config cache.type "reflink,symlink,hardlink,copy"
$ dvc config cache.protected true
```

However this implies that for data files that are added to the project with
`dvc add <datafile>`, you may need to run `dvc unprotect <datafile>` before
modifying them. For more details make sure to read the man page of
[dvc unprotect](/doc/commands-reference/unprotect).

</details>

## Other similar cases

If instead of an external drive we have a
[network-attached storage (NAS)](https://searchstorage.techtarget.com/definition/network-attached-storage)
mounted on the directory `/mnt/external-drive/` (through NFS, Samba, etc.), the
solution would be the same.

However, in this case the data is most probably used by a team of people, so
make sure to check also the case of
[Shared Development Server](/doc/use-cases/multiple-data-scientists-on-a-single-machine).