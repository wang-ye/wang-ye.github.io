When it comes to file/directory moving, removing and copying, the python module [*shutil*](https://docs.python.org/3/library/shutil.html) , which provides high-level operations for files/directories, is very handy. [shutil](https://docs.python.org/3/library/shutil.html) takes care of different aspects, including
1) the source and destination files are in the same server.
2) handling symbol links.
3) file Metadata preservation (modes, modification time ...)
4) the differences of the operating systems.

When reading the method definitions, I find that shutil provides two different methods for file copying, *shutil.copy* and *shutil.copy2*. The difference is really subtle, In this post I will dig deeper and unveil the connection and differences between the two methods. First, let's take a look at the source code.

```python
# Both from shutil.py
def copy2(src, dst, *, follow_symlinks=True):
    """Copy data and metadata. Return the file's destination.
    Metadata is copied with copystat(). Please see the copystat function
    for more information.
    The destination may be a directory.
    If follow_symlinks is false, symlinks won't be followed. This
    resembles GNU's "cp -P src dst".
    """
    if os.path.isdir(dst):
        dst = os.path.join(dst, os.path.basename(src))
    copyfile(src, dst, follow_symlinks=follow_symlinks)
    copystat(src, dst, follow_symlinks=follow_symlinks)
    return dst

def copy(src, dst, *, follow_symlinks=True):
    """Copy data and mode bits ("cp src dst"). Return the file's destination.
    The destination may be a directory.
    If follow_symlinks is false, symlinks won't be followed. This
    resembles GNU's "cp -P src dst".
    If source and destination are the same file, a SameFileError will be
    raised.
    """
    if os.path.isdir(dst):
        dst = os.path.join(dst, os.path.basename(src))
    copyfile(src, dst, follow_symlinks=follow_symlinks)
    copymode(src, dst, follow_symlinks=follow_symlinks)
    return dst
```

They are almost identical except the call to *copystat* vs *copymode* in the end. The *copymode* method is relatively simple.

```python
# From shutil.py
def copymode(src, dst, *, follow_symlinks=True):
    """Copy mode bits from src to dst.
    If follow_symlinks is not set, symlinks aren't followed if and only
    if both `src` and `dst` are symlinks.  If `lchmod` isn't available
    (e.g. Linux) this method does nothing.
    """
    if not follow_symlinks and os.path.islink(src) and os.path.islink(dst):
        if hasattr(os, 'lchmod'):
            stat_func, chmod_func = os.lstat, os.lchmod
        else:
            return
    elif hasattr(os, 'chmod'):
        stat_func, chmod_func = os.stat, os.chmod
    else:
        return

    st = stat_func(src)
    chmod_func(dst, stat.S_IMODE(st.st_mode))
```
This method is only concerned with the file mode. If follow_symlinks is set to False, and both the source and destination files are symbol links, then we will do nothing unless the system is POSIX-compliant ("lchmod"). If the follow_symlinks condition is true, the method copies the file mode from source to destination when *chmod* exists.

The copystat method, on the other hand, is a bit more complex.

```python
def copystat(src, dst, *, follow_symlinks=True):
    """Copy file metadata
    Copy the permission bits, last access time, last modification time, and
    flags from `src` to `dst`. On Linux, copystat() also copies the "extended
    attributes" where possible. The file contents, owner, and group are
    unaffected. `src` and `dst` are path names given as strings.
    If the optional flag `follow_symlinks` is not set, symlinks aren't
    followed if and only if both `src` and `dst` are symlinks.
    """
    def _nop(*args, ns=None, follow_symlinks=None):
        pass

    # follow symlinks (aka don't not follow symlinks)
    follow = follow_symlinks or not (os.path.islink(src) and os.path.islink(dst))
    if follow:
        # use the real function if it exists
        def lookup(name):
            return getattr(os, name, _nop)
    else:
        # use the real function only if it exists
        # *and* it supports follow_symlinks
        def lookup(name):
            fn = getattr(os, name, _nop)
            if fn in os.supports_follow_symlinks:
                return fn
            return _nop

    st = lookup("stat")(src, follow_symlinks=follow)
    mode = stat.S_IMODE(st.st_mode)
    lookup("utime")(dst, ns=(st.st_atime_ns, st.st_mtime_ns),
        follow_symlinks=follow)
    try:
        lookup("chmod")(dst, mode, follow_symlinks=follow)
    except NotImplementedError:
        # if we got a NotImplementedError, it's because
        #   * follow_symlinks=False,
        #   * lchown() is unavailable, and
        #   * either
        #       * fchownat() is unavailable or
        #       * fchownat() doesn't implement AT_SYMLINK_NOFOLLOW.
        #         (it returned ENOSUP.)
        # therefore we're out of options--we simply cannot chown the
        # symlink.  give up, suppress the error.
        # (which is what shutil always did in this circumstance.)
        pass
    if hasattr(st, 'st_flags'):
        try:
            lookup("chflags")(dst, st.st_flags, follow_symlinks=follow)
        except OSError as why:
            for err in 'EOPNOTSUPP', 'ENOTSUP':
                if hasattr(errno, err) and why.errno == getattr(errno, err):
                    break
            else:
                raise
    _copyxattr(src, dst, follow_symlinks=follow)
```

The *copystat* method tries to update multiple things including mode, update time and user-defined flags, while *copymode* only updates mode. Now comes the question. Does the mode update here has the same effect as *copymode*? To answer this question, let's compare the two follow link conditions separately.

1) When ```follow_symlinks or not (os.path.islink(src) and os.path.islink(dst))``` condition is true, the effect is the **same**.
2) When the condition is false, the *copymode* method checks whether *lchmod* exists and use it, otherwise there is no effect. Also, the *copystat* method uses *chmod* when *chmod* is in *os.supports_follow_symlinks*, and has no effect otherwise.

To really figure out case 2), let's start with the first question - when *chmod* will be in *os.supports_follow_symlinks*. Looking into the code in **os.py**, we find the following code:

```python
# IN os.py file
def _add(str, fn):
    if (fn in _globals) and (str in _have_functions):
        _set.add(_globals[fn])
_set = set()
_add("HAVE_FACCESSAT",  "access")
_add("HAVE_LCHMOD",     "chmod")
...
supports_follow_symlinks = _set
```

**HAVE_LCHMOD** is a flag from the c source files. So, ```chmod is in os.supports_follow_symlinks``` is true if and only if the operating system supports *lchmod*.

The second question is the difference between the lchmod and chmod. To resolve this, we have to dive into the posix c source file, which provides the actual definition of chmod and lchmod.

```c
// https://github.com/python/cpython/blob/4db62e115891425db2a974142a72d8eaaf95eecb/Modules/posixmodule.c#L2794
static PyObject *
os_chmod_impl(PyObject *module, path_t *path, int mode, int dir_fd,
              int follow_symlinks) {
    // ...
    #ifdef HAVE_LCHMOD
    if ((!follow_symlinks) && (dir_fd == DEFAULT_DIR_FD))
        result = lchmod(path->narrow, mode);
    else
    #endif
    // ...
}
```

So for chmod implementation, it will call *lchmod* if *HAVE_LCHMOD* is defined. To summarize, in copystat calls, when the follow condition is false, the operation will only be triggered when the system has lchmod, and the underlying call is all lchmod. They are essentially doing the same thing.

## Conclusion
By looking into the underlying source code, we conclude that the *shutil.copy2* contains a superset operation compared to *shutil.copy*.
There are some other ways to make file operations. For example, you can call the shell commands directly by using subprocess. However, shutil provides a set of utility methods to simplify the file operations and should be the first choice for file/directory operations.