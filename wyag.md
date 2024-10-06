## 1\. Introduction

(August 2023) Wyag is now complete\!

This article is an attempt at explaining the [Git version control
system](https://git-scm.com/) from the bottom up, that is, starting at
the most fundamental level moving up from there. This does not sound too
easy, and has been attempted multiple times with questionable success.
But there’s an easy way: all it takes to understand Git internals is to
reimplement Git from scratch.

No, don’t run.

It’s not a joke, and it’s really not complicated: if you read this
article top to bottom and write the code (or just [download
it](./wyag.zip) as a ZIP — but you should write the code yourself,
really), you’ll end up with a program, called `wyag`, that will
implement all the fundamental features of git: `init`, `add`, `rm`,
`status`, `commit`, `log`… in a way that is perfectly compatible with
`git` itself — compatible enough that the commit finally adding the
section on commits was [created by wyag itself, not
git](https://github.com/thblt/write-yourself-a-git/commit/ed26daffb400b2be5f30e044c3237d220226d867).
And all that in exactly 988 lines of very simple Python code.

But isn’t Git too complex for that? That Git is complex is, in my
opinion, a misconception. Git is a large program, with a lot of
features, that’s true. But the core of that program is actually
extremely simple, and its apparent complexity stems first from the fact
it’s often deeply counterintuitive (and [Git is a
burrito](https://byorgey.wordpress.com/2009/01/12/abstraction-intuition-and-the-monad-tutorial-fallacy/)
blog posts probably don’t help). But maybe what makes Git the most
confusing is the extreme simplicity *and* power of its core model. The
combination of core simplicity and powerful applications often makes
thing really hard to grasp, because of the mental jump required to
derive the variety of applications from the essential simplicity of the
fundamental abstraction (monads, anyone?)

Implementing Git will expose its fundamentals in all their naked glory.

**What to expect?** This article will implement and explain in great
details (if something is not clear, please [report it](#feedback)\!) a
very simplified version of Git core commands. I will keep the code
simple and to the point, so `wyag` won’t come anywhere near the power of
the real git command-line — but what’s missing will be obvious, and
trivial to implement by anyone who wants to give it a try. “Upgrading
wyag to a full-featured git library and CLI is left as an exercise to
the reader”, as they say.

More precisely, we’ll implement:

  - `add` ([wyag source](#cmd-add)) [git man
    page](https://git-scm.com/docs/git-add)
  - `cat-file` ([wyag source](#cmd-cat-file)) [git man
    page](https://git-scm.com/docs/git-cat-file)
  - `check-ignore` ([wyag source](#cmd-check-ignore)) [git man
    page](https://git-scm.com/docs/git-check-ignore)
  - `checkout` ([wyag source](#cmd-checkout)) [git man
    page](https://git-scm.com/docs/git-checkout)
  - `commit` ([wyag source](#cmd-commit)) [git man
    page](https://git-scm.com/docs/git-commit)
  - `hash-object` ([wyag source](#cmd-hash-object)) [git man
    page](https://git-scm.com/docs/git-hash-object)
  - `init` ([wyag source](#cmd-init)) [git man
    page](https://git-scm.com/docs/git-init)
  - `log` ([wyag source](#cmd-log)) [git man
    page](https://git-scm.com/docs/git-log)
  - `ls-files` ([wyag source](#cmd-ls-files)) [git man
    page](https://git-scm.com/docs/git-ls-files)
  - `ls-tree` ([wyag source](#cmd-ls-tree)) [git man
    page](https://git-scm.com/docs/git-ls-tree)
  - `rev-parse` ([wyag source](#cmd-rev-parse)) [git man
    page](https://git-scm.com/docs/git-rev-parse)
  - `rm` ([wyag source](#cmd-rm)) [git man
    page](https://git-scm.com/docs/git-rm)
  - `show-ref` ([wyag source](#cmd-show-ref)) [git man
    page](https://git-scm.com/docs/git-show-ref)
  - `status` ([wyag source](#cmd-status)) [git man
    page](https://git-scm.com/docs/git-status)
  - `tag` ([wyag source](#cmd-tag)) [git man
    page](https://git-scm.com/docs/git-tag)

You’re not going to need to know much to follow this article: just some
basic Git (obviously), some basic Python, some basic shell.

  - First, I’m only going to assume some level of familiarity with the
    most basic **git commands** — nothing like an expert level, but if
    you’ve never used `init`, `add`, `rm`, `commit` or `checkout`, you
    will be lost.
  - Language-wise, wyag will be implemented in **Python**. Again, I
    won’t use anything too fancy, and Python looks like pseudo-code
    anyways, so it will be easy to follow (ironically, the most
    complicated part will be the command-line arguments parsing logic,
    and you don’t really need to understand that). Yet, if you know
    programming but have never done any Python, I suggest you find a
    crash course somewhere in the internet just to get acquainted with
    the language.
  - `wyag` and `git` are terminal programs. I assume you know your way
    inside a Unix terminal. Again, you don’t need to be a l77t h4x0r,
    but `cd`, `ls`, `rm`, `tree` and their friends should be in your
    toolbox.

**Note for Windows users**

`wyag` should run on any Unix-like system with a Python interpreter, but
I have absolutely no idea how it will behave on Windows. The test suite
absolutely requires a bash-compatible shell, which I assume the WSL can
provide. Also, if you are using WSL, make sure your `wyag` file uses
Unix-style line endings ([See this StackOverflow solution if you use VS
Code](https://stackoverflow.com/questions/48692741/how-can-i-make-all-line-endings-eols-in-all-files-in-visual-studio-code-unix)).
Feedback from Windows users would be appreciated\!

****Acknowledgments****

This article benefited from significant contributions from multiple
people, and I’m grateful to them all. Special thanks to:

  - Github user [tammoippen](https://github.com/tammoippen), who first
    drafted the `tag_create` function I had simply… forgotten to write
    (that was
    [\#9](https://github.com/thblt/write-yourself-a-git/issues/9)).
  - Github user [hjlarry](https://github.com/hjlarry) fixed multiple
    issues in
    [\#22](https://github.com/thblt/write-yourself-a-git/pull/22).
  - GitHub user [cutebbb](https://github.com/cutebbb) implemented the
    first version of ls-files in
    [\#27](https://github.com/thblt/write-yourself-a-git/pull/27/), and
    by doing so finally brought wyag to the wonders of the staging
    area\!

## 2\. Getting started

You’re going to need Python 3.10 or higher, along with your favorite
text editor. We won’t need third party packages or virtualenvs, or
anything besides a regular Python interpreter: everything we need is in
Python’s standard library.

We’ll split the code into two files:

  - An executable, called `wyag`;
  - A Python library, called `libwyag.py`;

Now, every software project starts with a boatload of boilerplate, so
let’s get this over with.

We’ll begin by creating the (very short) executable. Create a new file
called `wyag` in your text editor, and copy the following few lines:

``` src src-python
#!/usr/bin/env python3

import libwyag
libwyag.main()
```

Then make it executable:

``` example
$ chmod +x wyag
```

You’re done\!

Now for the library. it must be called `libwyag.py`, and be in the same
directory as the `wyag` executable. Begin by opening the empty
`libwyag.py` in your text editor.

We’re first going to need a bunch of imports (just copy each import, or
merge them all in a single line)

  - Git is a CLI application, so we’ll need something to parse
    command-line arguments. Python provides a cool module called
    [argparse](https://docs.python.org/3/library/argparse.html) that can
    do 99% of the job for us.
    
    ``` src src-python
    import argparse
    ```

  - We’ll need a few more container types than the base lib provides,
    most notably an `OrderedDict`. It’s in
    [collections](https://docs.python.org/3/library/collections.html#collections.OrderedDict).
    
    ``` src src-python
    import collections
    ```

  - Git uses a configuration file format that is basically Microsoft’s
    INI format. The
    [configparser](https://docs.python.org/3/library/configparser.html)
    module can read and write these files.
    
    ``` src src-python
    import configparser
    ```

  - We’ll be doing some date/time manipulation:
    
    ``` src src-python
    from datetime import datetime
    ```

  - We’ll need, just once, to read the users/group database on Unix
    (`grp` is for groups, `pwd` for users). This is because git saves
    the numerical owner/group ID of files, and we’ll want to display
    that nicely (as text):
    
    ``` src src-python
    import grp, pwd
    ```

  - To support `.gitignore`, we’ll need to match filenames against
    patterns like \*.txt. Filename matching is in… `fnmatch`:
    
    ``` src src-python
    from fnmatch import fnmatch
    ```

  - Git uses the SHA-1 function quite extensively. In Python, it’s in
    [hashlib](https://docs.python.org/3/library/hashlib.html).
    
    ``` src src-python
    import hashlib
    ```

  - Just one function from
    [math](https://docs.python.org/3/library/math.html):
    
    ``` src src-python
    from math import ceil
    ```

  - [os](https://docs.python.org/3/library/os.html) and
    [os.path](https://docs.python.org/3/library/os.path.html) provide
    some nice filesystem abstraction routines.
    
    ``` src src-python
    import os
    ```

  - we use *just a bit* of regular expressions:
    
    ``` src src-python
    import re
    ```

  - We also need [sys](https://docs.python.org/3/library/sys.html) to
    access the actual command-line arguments (in `sys.argv`):
    
    ``` src src-python
    import sys
    ```

  - Git compresses everything using zlib. Python [has
    that](https://docs.python.org/3/library/zlib.html), too:
    
    ``` src src-python
    import zlib
    ```

Imports are done. We’ll be working with command-line arguments a lot.
Python provides a simple yet reasonably powerful parsing library,
`argparse`. It’s a nice library, but its interface may not be the most
intuitive ever; if need, refer to its
[documentation](https://docs.python.org/3/library/argparse.html).

``` src src-python
argparser = argparse.ArgumentParser(description="The stupidest content tracker")
```

We’ll need to handle subcommands (as in git: `init`, `commit`, etc.) In
argparse slang, these are called “subparsers”. At this point we only
need to declare that our CLI will use some, and that all invocation will
actually *require* one — you don’t just call `git`, you call `git
COMMAND`.

``` src src-python
argsubparsers = argparser.add_subparsers(title="Commands", dest="command")
argsubparsers.required = True
```

The `dest="command"` argument states that the name of the chosen
subparser will be returned as a string in a field called `command`. So
we just need to read this string and call the correct function
accordingly. By convention, I’ll call these functions “bridges
functions” and prefix their names by `cmd_`. Bridge functions take the
parsed arguments as their unique parameter, and are responsible for
processing and validating them before executing the actual command.

``` src src-python
def main(argv=sys.argv[1:]):
    args = argparser.parse_args(argv)
    match args.command:
        case "add"          : cmd_add(args)
        case "cat-file"     : cmd_cat_file(args)
        case "check-ignore" : cmd_check_ignore(args)
        case "checkout"     : cmd_checkout(args)
        case "commit"       : cmd_commit(args)
        case "hash-object"  : cmd_hash_object(args)
        case "init"         : cmd_init(args)
        case "log"          : cmd_log(args)
        case "ls-files"     : cmd_ls_files(args)
        case "ls-tree"      : cmd_ls_tree(args)
        case "rev-parse"    : cmd_rev_parse(args)
        case "rm"           : cmd_rm(args)
        case "show-ref"     : cmd_show_ref(args)
        case "status"       : cmd_status(args)
        case "tag"          : cmd_tag(args)
        case _              : print("Bad command.")
```

## 3\. Creating repositories: init

Obviously, the first Git command in chronological *and* logical order is
`git init`, so we’ll begin by creating `wyag init`. To achieve this,
we’re going to first need some very basic repository abstraction.

### 3.1. The Repository object

We’ll obviously need some abstraction for a repository: almost every
time we run a git command, we’re trying to do something to a repository,
to create it, read from it or modify it.

A git repository is made of two things: a “work tree”, where the files
meant to be in version control live, and a “git directory”, where Git
stores its own data. In most cases, the worktree is a regular directory
and the git directory is a child directory of the worktree, called
`.git`.

Git supports *much more* cases (bare repo, separated gitdir, etc) but we
won’t need them: we’ll stick to the basic approach of `worktree/.git`.
Our repository object will then just hold two paths: the worktree and
the gitdir.

To create a new `Repository` object, we only need to make a few checks:

  - We must verify that the directory exists, and contains a
    subdirectory called `.git`.
  - We read its configuration in `.git/config` (it’s just an INI file)
    and control that `core.repositoryformatversion` is 0. More on that
    field in a moment.

The constructor takes an optional `force` which disables all checks.
That’s because the `repo_create()` function which we’ll create later
uses a `Repository` object to *create* the repo. So we need a way to
create repository even from (still) invalid filesystem locations.

``` src src-python
class GitRepository (object):
    """A git repository"""

    worktree = None
    gitdir = None
    conf = None

    def __init__(self, path, force=False):
        self.worktree = path
        self.gitdir = os.path.join(path, ".git")

        if not (force or os.path.isdir(self.gitdir)):
            raise Exception("Not a Git repository %s" % path)

        # Read configuration file in .git/config
        self.conf = configparser.ConfigParser()
        cf = repo_file(self, "config")

        if cf and os.path.exists(cf):
            self.conf.read([cf])
        elif not force:
            raise Exception("Configuration file missing")

        if not force:
            vers = int(self.conf.get("core", "repositoryformatversion"))
            if vers != 0:
                raise Exception("Unsupported repositoryformatversion %s" % vers)
```

We’re going to be manipulating **lots** of paths in repositories. We may
as well create a few utility functions to compute those paths and create
missing directory structures if needed. First, just a general path
building function:

``` src src-python
def repo_path(repo, *path):
    """Compute path under repo's gitdir."""
    return os.path.join(repo.gitdir, *path)
```

(A note on Python syntax: the star on the `*path` makes the function
variadic, so it can be called with multiple path components as separate
arguments. For example, `repo_path(repo, "objects", "df",
"4ec9fc2ad990cb9da906a95a6eda6627d7b7b0")` is a valid call. The function
receives `path` as a list)

The next two functions, `repo_file()` and `repo_dir()`, return and
optionally create a path to a file or a directory, respectively. The
difference between them is that the file version only creates
directories up to the last component.

``` src src-python
def repo_file(repo, *path, mkdir=False):
    """Same as repo_path, but create dirname(*path) if absent.  For
example, repo_file(r, \"refs\", \"remotes\", \"origin\", \"HEAD\") will create
.git/refs/remotes/origin."""

    if repo_dir(repo, *path[:-1], mkdir=mkdir):
        return repo_path(repo, *path)

def repo_dir(repo, *path, mkdir=False):
    """Same as repo_path, but mkdir *path if absent if mkdir."""

    path = repo_path(repo, *path)

    if os.path.exists(path):
        if (os.path.isdir(path)):
            return path
        else:
            raise Exception("Not a directory %s" % path)

    if mkdir:
        os.makedirs(path)
        return path
    else:
        return None
```

(Second and last note on syntax: because the star in `*path` makes the
functions variadic, the `mkdir` argument must be passed explicitly by
name. For example, `repo_file(repo, "objects", mkdir=True)`.)

To **create** a new repository, we start with a directory (which we
create if doesn’t already exist) and create the **git directory** inside
(which must not exist already, or be empty). That directory is called
`.git` (the leading period makes it “hidden” on Unix systems), and
contains:

  - `.git/objects/` : the object store, which we’ll introduce [in the
    next section](#objects).
  - `.git/refs/` the reference store, which we’ll discuss [a bit
    later](#cmd-show-ref). It contains two subdirectories, `heads` and
    `tags`.
  - `.git/HEAD`, a reference to the current HEAD (more on that later\!)
  - `.git/config`, the repository’s configuration file.
  - `.git/description`, holds a free-form description of this
    repository’s contents, for humans, and is rarely used.

<!-- end list -->

``` src src-python
def repo_create(path):
    """Create a new repository at path."""

    repo = GitRepository(path, True)

    # First, we make sure the path either doesn't exist or is an
    # empty dir.

    if os.path.exists(repo.worktree):
        if not os.path.isdir(repo.worktree):
            raise Exception ("%s is not a directory!" % path)
        if os.path.exists(repo.gitdir) and os.listdir(repo.gitdir):
            raise Exception("%s is not empty!" % path)
    else:
        os.makedirs(repo.worktree)

    assert repo_dir(repo, "branches", mkdir=True)
    assert repo_dir(repo, "objects", mkdir=True)
    assert repo_dir(repo, "refs", "tags", mkdir=True)
    assert repo_dir(repo, "refs", "heads", mkdir=True)

    # .git/description
    with open(repo_file(repo, "description"), "w") as f:
        f.write("Unnamed repository; edit this file 'description' to name the repository.\n")

    # .git/HEAD
    with open(repo_file(repo, "HEAD"), "w") as f:
        f.write("ref: refs/heads/master\n")

    with open(repo_file(repo, "config"), "w") as f:
        config = repo_default_config()
        config.write(f)

    return repo
```

The configuration file is very simple, it’s a
[INI](https://en.wikipedia.org/wiki/INI_file)-like file with a single
section (`[core]`) and three fields:

  - `repositoryformatversion = 0`: the version of the gitdir format. 0
    means the initial format, 1 the same with extensions. If \> 1, git
    will panic; wyag will only accept 0.
  - `filemode = false`: disable tracking of file modes (permissions)
    changes in the work tree.
  - `bare = false`: indicates that this repository has a worktree. Git
    supports an optional `worktree` key which indicates the location of
    the worktree, if not `..`; wyag doesn’t.

We create this file using Python’s `configparser` lib:

``` src src-python
def repo_default_config():
    ret = configparser.ConfigParser()

    ret.add_section("core")
    ret.set("core", "repositoryformatversion", "0")
    ret.set("core", "filemode", "false")
    ret.set("core", "bare", "false")

    return ret
```

### 3.2. The init command

Now that we have code to read and create repositories, let’s make this
code usable from the command line by creating the `wyag init` command.
`wyag init` behaves just like `git init` — with much less
customizability, of course. The syntax of `wyag init` is going to be:

``` example
wyag init [path]
```

We already have the complete repository creation logic. To create the
command, we’re only going to need two more things:

1.  We need to create an argparse subparser to handle our command’s
    argument.
    
    ``` src src-python
    argsp = argsubparsers.add_parser("init", help="Initialize a new, empty repository.")
    ```
    
    In the case of `init`, there’s a single, optional, positional
    argument: the path where to init the repo. It defaults to `.`, the
    current directory:
    
    ``` src src-python
    argsp.add_argument("path",
                       metavar="directory",
                       nargs="?",
                       default=".",
                       help="Where to create the repository.")
    ```

2.  We also need a “bridge” function that will read argument values from
    the object returned by argparse and call the actual function with
    correct values.
    
    ``` src src-python
    def cmd_init(args):
        repo_create(args.path)
    ```

And we’re done\! If you’ve followed these steps, you should now be able
to `wyag init` a git repository anywhere:

``` example
$ wyag init test
```

(The `wyag` executable won’t usually be in your `$PATH`: you’ll want to
call it by its full name, eg `~/projects/wyag/wyag init .`)

### 3.3. The repo\_find() function

While we’re implementing repositories, we’re going to need a function to
find the root of the current repository. We’ll use it a lot, since
almost all Git functions work on an existing repository (except `init`,
of course\!). Sometimes that root is the current directory, but it may
also be a parent: your repository’s root may be in
`~/Documents/MyProject`, but you may currently be working in
`~/Documents/MyProject/src/tui/frames/mainview/`. The `repo_find()`
function we’ll now create will look for that root, starting at the
current directory and recursing back to `/`. To identify a path as a
repository, it will check for the presence of a `.git` directory.

``` src src-python
def repo_find(path=".", required=True):
    path = os.path.realpath(path)

    if os.path.isdir(os.path.join(path, ".git")):
        return GitRepository(path)

    # If we haven't returned, recurse in parent, if w
    parent = os.path.realpath(os.path.join(path, ".."))

    if parent == path:
        # Bottom case
        # os.path.join("/", "..") == "/":
        # If parent==path, then path is root.
        if required:
            raise Exception("No git directory.")
        else:
            return None

    # Recursive case
    return repo_find(parent, required)
```

And we’re done with repositories\!

## 4\. Reading and writing objects: hash-object and cat-file

### 4.1. What are objects?

Now that we have repositories, putting things inside them is in order.
Also, repositories are boring, and writing a Git implementation
shouldn’t be just a matter of writing a bunch of `mkdir`. Let’s talk
about **objects**, and let’s implement `git hash-object` and `git
cat-file`.

Maybe you don’t know these two commands — they’re not exactly part of an
everyday git toolbox, and they’re actually quite low-level (“plumbing”,
in git parlance). What they do is actually very simple: `hash-object`
converts an existing file into a git object, and `cat-file` prints an
existing git object to the standard output.

Now, **what actually is a Git object?** At its core, Git is a
“content-addressed filesystem”. That means that unlike regular
filesystems, where the name of a file is arbitrary and unrelated to that
file’s contents, the names of files as stored by Git are mathematically
derived from their contents. This has a very important implication: if a
single byte of, say, a text file, changes, its internal name will
change, too. To put it simply: you don’t *modify* a file in git, you
create a new file in a different location. Objects are just that:
**files in the git repository, whose paths are determined by their
contents**.

**Git is not (really) a key-value store**

Some documentation, including the excellent [Pro
Git](https://git-scm.com/book/id/v2/Git-Internals-Git-Objects), call Git
a “key-value store”. This is not incorrect, but may be misleading.
Regular filesystems are actually closer to a key-value store than Git
is. Because it computes keys from data, Git could rather be called a
*value-value store*.

Git uses objects to store quite a lot of things: first and foremost, the
actual files it keeps in version control — source code, for example.
Commit are objects, too, as well as tags. With a few notable exceptions
(which we’ll see later\!), almost everything, in Git, is stored as an
object.

The path where git stores a given object is computed by calculating the
[SHA-1](https://en.wikipedia.org/wiki/SHA-1)
[hash](https://en.wikipedia.org/wiki/Cryptographic_hash_function) of its
contents. More precisely, Git renders the hash as a lowercase
hexadecimal string, and splits it in two parts: the first two
characters, and the rest. It uses the first part as a directory name,
the rest as the file name (this is because most filesystems hate having
too many files in a single directory and would slow down to a crawl.
Git’s method creates 256 possible intermediate directories, hence
dividing the average number of files per directory by 256)

**What is a hash function?**

SHA-1 is what we call a “hash function”. Simply put, a hash function is
a kind of unidirectional mathematical function: it is easy to compute
the hash of a value, but there’s no way to compute back which value
produced a hash.

A very simple example of a hash function is the classical `len` (or
`strlen`) function, which returns the length of a string. It’s really
easy to compute the length of a string, and the length of a given string
will never change (unless the string itself changes, of course\!) but
it’s impossible to retrieve the original string, given only its
length. *Cryptographic* hash functions are a much more complex version
of the same, with the added property that computing an input meant to
produce a given hash is hard enough to be practically impossible. (To
produce an input `i` with `strlen(i) == 12`, you just type twelve random
characters. With algorithms such as SHA-1. it would take much, much
longer — long enough to be practically impossible^([1](#fn.1))).

Before we start implementing the object storage system, we must
understand their exact storage format. An object starts with a header
that specifies its type: `blob`, `commit`, `tag` or `tree` (more on that
in a second). This header is followed by an ASCII space (0x20), then the
size of the object in bytes as an ASCII number, then null (0x00) (the
null byte), then the contents of the object. The first 48 bytes of a
commit object in Wyag’s repo look like this:

``` example
00000000  63 6f 6d 6d 69 74 20 31  30 38 36 00 74 72 65 65  |commit 1086.tree|
00000010  20 32 39 66 66 31 36 63  39 63 31 34 65 32 36 35  | 29ff16c9c14e265|
00000020  32 62 32 32 66 38 62 37  38 62 62 30 38 61 35 61  |2b22f8b78bb08a5a|
```

In the first line, we see the type header, a space (`0x20`), the size in
ASCII (1086) and the null separator `0x00`. The last four bytes on the
first line are the beginning of that object’s contents, the word “tree”
— we’ll discuss that further when we’ll talk about commits.

The objects (headers and contents) are stored compressed with `zlib`.

### 4.2. A generic object object

Objects can be of multiple types, but they all share the same
storage/retrieval mechanism and the same general header format. Before
we dive into the details of various types of objects, we need to
abstract over these common features. The easiest way is to create a
generic `GitObject` with two unimplemented methods: `serialize()` and
`deserialize()`, and a default `init()` to create a new, empty object if
needed (sorry pythonistas, this isn’t very nice design but it’s probably
easier to read than superconstructors). Our `__init__` either loads the
object from the provided data, or calls the subclass-provided `init()`
to create a new, empty object.

Later, we’ll subclass this generic class, actually implementing these
functions for each object format.

``` src src-python
class GitObject (object):

    def __init__(self, data=None):
        if data != None:
            self.deserialize(data)
        else:
            self.init()

    def serialize(self, repo):
        """This function MUST be implemented by subclasses.

It must read the object's contents from self.data, a byte string, and do
whatever it takes to convert it into a meaningful representation.  What exactly that means depend on each subclass."""
        raise Exception("Unimplemented!")

    def deserialize(self, data):
        raise Exception("Unimplemented!")

    def init(self):
        pass # Just do nothing. This is a reasonable default!
```

### 4.3. Reading objects

To read an object, we need to know its SHA-1 hash. We then compute its
path from this hash (with the formula explained above: first two
characters, then a directory delimiter `/`, then the remaining part) and
look it up inside of the “objects” directory in the gitdir. That is, the
path to `e673d1b7eaa0aa01b5bc2442d570a765bdaae751` is
`.git/objects/e6/73d1b7eaa0aa01b5bc2442d570a765bdaae751`.

We then read that file as a binary file, and decompress it using `zlib`.

From the decompressed data, we extract the two header components: the
object type and its size. From the type, we determine the actual class
to use. We convert the size to a Python integer, and check if it
matches.

When all is done, we just call the correct constructor for that object’s
format.

``` src src-python
def object_read(repo, sha):
    """Read object sha from Git repository repo.  Return a
    GitObject whose exact type depends on the object."""

    path = repo_file(repo, "objects", sha[0:2], sha[2:])

    if not os.path.isfile(path):
        return None

    with open (path, "rb") as f:
        raw = zlib.decompress(f.read())

        # Read object type
        x = raw.find(b' ')
        fmt = raw[0:x]

        # Read and validate object size
        y = raw.find(b'\x00', x)
        size = int(raw[x:y].decode("ascii"))
        if size != len(raw)-y-1:
            raise Exception("Malformed object {0}: bad length".format(sha))

        # Pick constructor
        match fmt:
            case b'commit' : c=GitCommit
            case b'tree'   : c=GitTree
            case b'tag'    : c=GitTag
            case b'blob'   : c=GitBlob
            case _:
                raise Exception("Unknown type {0} for object {1}".format(fmt.decode("ascii"), sha))

        # Call constructor and return object
        return c(raw[y+1:])
```

### 4.4. Writing objects

Writing an object is reading it in reverse: we compute the hash, insert
the header, zlib-compress everything and write the result in the correct
location. This really shouldn’t require much explanation, just notice
that the hash is computed **after** the header is added (so it’s the
hash of the object itself, uncompressed, not just its contents)

``` src src-python
def object_write(obj, repo=None):
    # Serialize object data
    data = obj.serialize()
    # Add header
    result = obj.fmt + b' ' + str(len(data)).encode() + b'\x00' + data
    # Compute hash
    sha = hashlib.sha1(result).hexdigest()

    if repo:
        # Compute path
        path=repo_file(repo, "objects", sha[0:2], sha[2:], mkdir=True)

        if not os.path.exists(path):
            with open(path, 'wb') as f:
                # Compress and write
                f.write(zlib.compress(result))
    return sha
```

### 4.5. Working with blobs

We said earlier that the type header could be one of four: `blob`,
`commit`, `tag` and `tree` — so git has four object types.

Blobs are the simplest of those four types, because they have no actual
format. Blobs are user data: the content of every file you put in git
(`main.c`, `logo.png`, `README.md`) is stored as a blob. That makes them
easy to manipulate, because they have no actual syntax or constraints
beyond the basic object storage mechanism: they’re just unspecified
data. Creating a `GitBlob` class is thus trivial, the `serialize` and
`deserialize` functions just have to store and return their input
unmodified.

``` src src-python
class GitBlob(GitObject):
    fmt=b'blob'

    def serialize(self):
        return self.blobdata

    def deserialize(self, data):
        self.blobdata = data
```

### 4.6. The cat-file command

We can now create `wyag cat-file`. `git cat-file` simply prints the raw
contents of an object to stdout, uncompressed and without the git
header. In a clone of [wyag’s source
repository](https://github.com/thblt/write-yourself-a-git), `git
cat-file blob e0695f14a412c29e252c998c81de1dde59658e4a` will show a
version of the README.

Our simplified version will just take those two positional arguments: a
type and an object identifier:

``` example
wyag cat-file TYPE OBJECT
```

The subparser is very simple:

``` src src-python
argsp = argsubparsers.add_parser("cat-file",
                                 help="Provide content of repository objects")

argsp.add_argument("type",
                   metavar="type",
                   choices=["blob", "commit", "tag", "tree"],
                   help="Specify the type")

argsp.add_argument("object",
                   metavar="object",
                   help="The object to display")
```

And we can implement the functions, which just call into existing code
we wrote earlier:

``` src src-python
def cmd_cat_file(args):
    repo = repo_find()
    cat_file(repo, args.object, fmt=args.type.encode())

def cat_file(repo, obj, fmt=None):
    obj = object_read(repo, object_find(repo, obj, fmt=fmt))
    sys.stdout.buffer.write(obj.serialize())
```

This function calls an `object_find` function we haven’t introduced yet.
For now, it’s just going to return one of its arguments unmodified, like
this:

``` src src-python
def object_find(repo, name, fmt=None, follow=True):
    return name
```

The reason for this strange small function is that Git has a *lot* of
ways to refer to objects: full hash, short hash, tags… `object_find()`
will be our name resolution function. We’ll only implement it
[later](#object_find), so this is just a temporary placeholder. This
means that until we implement the real thing, the only way we can refer
to an object will be by its full hash.

### 4.7. The hash-object command

We will want to put our *own* data in our repositories, though.
`hash-object` is basically the opposite of `cat-file`: it reads a file,
computes its hash as an object, either storing it in the repository (if
the -w flag is passed) or just printing its hash.

The syntax of `wyag hash-object` is a simplification of `git
hash-object`:

``` example
wyag hash-object [-w] [-t TYPE] FILE
```

Which converts to:

``` src src-python
argsp = argsubparsers.add_parser(
    "hash-object",
    help="Compute object ID and optionally creates a blob from a file")

argsp.add_argument("-t",
                   metavar="type",
                   dest="type",
                   choices=["blob", "commit", "tag", "tree"],
                   default="blob",
                   help="Specify the type")

argsp.add_argument("-w",
                   dest="write",
                   action="store_true",
                   help="Actually write the object into the database")

argsp.add_argument("path",
                   help="Read object from <file>")
```

The actual implementation is very simple. As usual, we create a small
bridge function:

``` src src-python
def cmd_hash_object(args):
    if args.write:
        repo = repo_find()
    else:
        repo = None

    with open(args.path, "rb") as fd:
        sha = object_hash(fd, args.type.encode(), repo)
        print(sha)
```

The actual implementation is also trivial. The `repo` argument is
optional, and the object isn’t written if it is `None` (this is handled
in `object_write()`, above):

``` src src-python
def object_hash(fd, fmt, repo=None):
    """ Hash object, writing it to repo if provided."""
    data = fd.read()

    # Choose constructor according to fmt argument
    match fmt:
        case b'commit' : obj=GitCommit(data)
        case b'tree'   : obj=GitTree(data)
        case b'tag'    : obj=GitTag(data)
        case b'blob'   : obj=GitBlob(data)
        case _: raise Exception("Unknown type %s!" % fmt)

    return object_write(obj, repo)
```

### 4.8. Aside: what about packfiles?

What we’ve just implemented is called “loose objects”. Git has a second
object storage mechanism called packfiles. Packfiles are much more
efficient, but also much more complex, than loose objects. Simply put, a
packfile is a compilation of loose objects (like a `tar`) but some are
stored as deltas (as a transformation of another object). Packfiles are
way too complex to be supported by wyag.

The packfile is stored in `.git/objects/pack/`. It has a `.pack`
extension, and is accompanied by an index file of the same name with the
`.idx` extension. Should you want to convert a packfile to loose objects
format (to play with `wyag` on an existing repo, for example), here’s
the solution.

First, *move* the packfile outside the gitdir (just copying it won’t
work).

``` src src-shell
mv .git/objects/pack/pack-d9ef004d4ca729287f12aaaacf36fee39baa7c9d.pack .
```

You can ignore the `.idx`. Then, from the worktree, just `cat` it and
pipe the result to `git unpack-objects`:

``` src src-shell
cat pack-d9ef004d4ca729287f12aaaacf36fee39baa7c9d.pack | git unpack-objects
```

## 5\. Reading commit history: log

### 5.1. Parsing commits

Now that we can read and write objects, we should consider commits. A
commit object (uncompressed, without headers) looks like this:

``` example
tree 29ff16c9c14e2652b22f8b78bb08a5a07930c147
parent 206941306e8a8af65b66eaaaea388a7ae24d49a0
author Thibault Polge <thibault@thb.lt> 1527025023 +0200
committer Thibault Polge <thibault@thb.lt> 1527025044 +0200
gpgsig -----BEGIN PGP SIGNATURE-----

 iQIzBAABCAAdFiEExwXquOM8bWb4Q2zVGxM2FxoLkGQFAlsEjZQACgkQGxM2FxoL
 kGQdcBAAqPP+ln4nGDd2gETXjvOpOxLzIMEw4A9gU6CzWzm+oB8mEIKyaH0UFIPh
 rNUZ1j7/ZGFNeBDtT55LPdPIQw4KKlcf6kC8MPWP3qSu3xHqx12C5zyai2duFZUU
 wqOt9iCFCscFQYqKs3xsHI+ncQb+PGjVZA8+jPw7nrPIkeSXQV2aZb1E68wa2YIL
 3eYgTUKz34cB6tAq9YwHnZpyPx8UJCZGkshpJmgtZ3mCbtQaO17LoihnqPn4UOMr
 V75R/7FjSuPLS8NaZF4wfi52btXMSxO/u7GuoJkzJscP3p4qtwe6Rl9dc1XC8P7k
 NIbGZ5Yg5cEPcfmhgXFOhQZkD0yxcJqBUcoFpnp2vu5XJl2E5I/quIyVxUXi6O6c
 /obspcvace4wy8uO0bdVhc4nJ+Rla4InVSJaUaBeiHTW8kReSFYyMmDCzLjGIu1q
 doU61OM3Zv1ptsLu3gUE6GU27iWYj2RWN3e3HE4Sbd89IFwLXNdSuM0ifDLZk7AQ
 WBhRhipCCgZhkj9g2NEk7jRVslti1NdN5zoQLaJNqSwO1MtxTmJ15Ksk3QP6kfLB
 Q52UWybBzpaP9HEd4XnR+HuQ4k2K0ns2KgNImsNvIyFwbpMUyUWLMPimaV1DWUXo
 5SBjDB/V/W2JBFR+XKHFJeFwYhj7DD/ocsGr4ZMx/lgc8rjIBkI=
 =lgTX
 -----END PGP SIGNATURE-----

Create first draft
```

The format is a simplified version of mail messages, as specified in
[RFC 2822](https://www.ietf.org/rfc/rfc2822.txt). It begins with a
series of key-value pairs, with space as the key/value separator, and
ends with the commit message, that may span over multiple lines. Values
may continue over multiple lines, subsequent lines start with a space
which the parser must drop (like the `gpgsig` field above, which spans
over 16 lines).

Let’s have a look at those fields:

  - `tree` is a reference to a tree object, a type of object that we’ll
    see just next. A tree maps blobs IDs to filesystem locations, and
    describes a state of the work tree. Put simply, it is the actual
    content of the commit: file contents, and where they go.
  - `parent` is a reference to the parent of this commit. It may be
    repeated: merge commits, for example, have multiple parents. It may
    also be absent: the very first commit in a repository obviously
    doesn’t have a parent.
  - `author` and `committer` are separate, because the author of a
    commit is not necessarily the person who can commit it (This may not
    be obvious for GitHub users, but a lot of projects do Git through
    e-mail)
  - `gpgsig` is the PGP signature of this object.

We’ll start by writing a simple parser for the format. The code is
obvious. The name of the function we’re about to create, `kvlm_parse()`,
may be confusing: it isn’t called `commit_parse()` because tags have the
very same format, so we’ll use it for both objects types. I use KVLM to
mean “Key-Value List with Message”.

``` src src-python
def kvlm_parse(raw, start=0, dct=None):
    if not dct:
        dct = collections.OrderedDict()
        # You CANNOT declare the argument as dct=OrderedDict() or all
        # call to the functions will endlessly grow the same dict.

    # This function is recursive: it reads a key/value pair, then call
    # itself back with the new position.  So we first need to know
    # where we are: at a keyword, or already in the messageQ

    # We search for the next space and the next newline.
    spc = raw.find(b' ', start)
    nl = raw.find(b'\n', start)

    # If space appears before newline, we have a keyword.  Otherwise,
    # it's the final message, which we just read to the end of the file.

    # Base case
    # =========
    # If newline appears first (or there's no space at all, in which
    # case find returns -1), we assume a blank line.  A blank line
    # means the remainder of the data is the message.  We store it in
    # the dictionary, with None as the key, and return.
    if (spc < 0) or (nl < spc):
        assert nl == start
        dct[None] = raw[start+1:]
        return dct

    # Recursive case
    # ==============
    # we read a key-value pair and recurse for the next.
    key = raw[start:spc]

    # Find the end of the value.  Continuation lines begin with a
    # space, so we loop until we find a "\n" not followed by a space.
    end = start
    while True:
        end = raw.find(b'\n', end+1)
        if raw[end+1] != ord(' '): break

    # Grab the value
    # Also, drop the leading space on continuation lines
    value = raw[spc+1:end].replace(b'\n ', b'\n')

    # Don't overwrite existing data contents
    if key in dct:
        if type(dct[key]) == list:
            dct[key].append(value)
        else:
            dct[key] = [ dct[key], value ]
    else:
        dct[key]=value

    return kvlm_parse(raw, start=end+1, dct=dct)
```

**Object identity rules**

We use an `OrderedDict` (a dictionary/hashmap where keys are ordered) so
fields always appear in the same order. This matters, because Git has
**two strong rules about object identity**:

1.  The first rule is that **the same name will always refer to the same
    object**. We’ve seen this one already, it’s just a consequence of
    the fact that an object’s name is a hash of its contents.
2.  The second rule is subtly different: **the same object will always
    be referred by the same name**. This means that there shouldn’t be
    two equivalent objects under different names. This is why fields
    order matter: by modifying the *order* fields appear in a given
    commit, eg by putting the `tree` after the `parent`, we’d modify the
    SHA-1 hash of the commit, and we’d create two equivalent, but
    numerically distinct, commit objects.

For example, when comparing trees, git will assume that two trees with
different names *are* different — this is why we’ll have to make sure
elements of the tree objects are properly sorted, so we don’t produce
distinct but equivalent trees.

We’re also going to need to write similar objects, so let’s add a
`kvlm_serialize()` function to our toolkit. This is very simple: we
write all fields first, then a newline, the message, and a final
newline.

``` src src-python
def kvlm_serialize(kvlm):
    ret = b''

    # Output fields
    for k in kvlm.keys():
        # Skip the message itself
        if k == None: continue
        val = kvlm[k]
        # Normalize to a list
        if type(val) != list:
            val = [ val ]

        for v in val:
            ret += k + b' ' + (v.replace(b'\n', b'\n ')) + b'\n'

    # Append message
    ret += b'\n' + kvlm[None] + b'\n'

    return ret
```

### 5.2. The Commit object

Now we have the parser, we can create the `GitCommit` class:

``` src src-python
class GitCommit(GitObject):
    fmt=b'commit'

    def deserialize(self, data):
        self.kvlm = kvlm_parse(data)

    def serialize(self):
        return kvlm_serialize(self.kvlm)

    def init(self):
        self.kvlm = dict()
```

### 5.3. The log command

We’ll implement a much, much simpler version of `log` than what Git
provides. Most importantly, we won’t deal with representing the log *at
all*. Instead, we’ll dump Graphviz data and let the user use `dot` to
render the actual log. (If you don’t know how to use Graphviz, just
paste the raw output into [this
site](https://dreampuf.github.io/GraphvizOnline/). If the link is dead,
lookup “graphviz online” in your favorite search engine)

``` src src-python
argsp = argsubparsers.add_parser("log", help="Display history of a given commit.")
argsp.add_argument("commit",
                   default="HEAD",
                   nargs="?",
                   help="Commit to start at.")
```

``` src src-python
def cmd_log(args):
    repo = repo_find()

    print("digraph wyaglog{")
    print("  node[shape=rect]")
    log_graphviz(repo, object_find(repo, args.commit), set())
    print("}")

def log_graphviz(repo, sha, seen):

    if sha in seen:
        return
    seen.add(sha)

    commit = object_read(repo, sha)
    short_hash = sha[0:8]
    message = commit.kvlm[None].decode("utf8").strip()
    message = message.replace("\\", "\\\\")
    message = message.replace("\"", "\\\"")

    if "\n" in message: # Keep only the first line
        message = message[:message.index("\n")]

    print("  c_{0} [label=\"{1}: {2}\"]".format(sha, sha[0:7], message))
    assert commit.fmt==b'commit'

    if not b'parent' in commit.kvlm.keys():
        # Base case: the initial commit.
        return

    parents = commit.kvlm[b'parent']

    if type(parents) != list:
        parents = [ parents ]

    for p in parents:
        p = p.decode("ascii")
        print ("  c_{0} -> c_{1};".format(sha, p))
        log_graphviz(repo, p, seen)
```

You can now use our log command like this:

``` src src-shell
wyag log e03158242ecab460f31b0d6ae1642880577ccbe8 > log.dot
dot -O -Tpdf log.dot
```

### 5.4. Anatomy of a commit

You may have noticed a few things right now.

First and foremost, we’ve been playing with commits, browsing and
walking through commit objects, building a graph of commit history,
without ever touching a single file in the worktree or a blob. We’ve
done a lot with commits *without considering their contents*. This is
important: work tree contents are just one part of a commit. But a
commit is made of everything it holds: its contents, its authors, **also
its parents**. If you remember that the ID (the SHA-1 hash) of a commit
is computed from the whole commit object, you’ll understand what it
means that commits are immutable: if you change the author, the parent
commit or a single file, you’ve actually created a new, different
object. Each and every commit is bound to its place and its relationship
to the whole repository up to the very first commit. To put it
otherwise, a given commit ID not only identifies some file contents, but
it also binds the commit to its whole history and to the whole
repository.

It’s also worth noting that from the point of view of a commit, time
somehow runs backwards: we’re used to considering the history of a
project from its humble beginnings as an evening distraction, starting
with a few lines of code, some initial commits, and progressing to its
present state (millions of lines of code, dozens of contributors,
whatever). But each commit is completely unaware of its future, it’s
only linked to the past. Commits have “memory”, but no premonition.

So what makes a commit? To sum it up:

  - A tree object, which we’ll discuss now, that is, the contents of a
    worktree, files and directories;
  - Zero, one or more parents;
  - An author identity (name and email), and a timestamp;
  - A committer identity (name and email), and a timestamp;
  - An optional PGP signature
  - A message;

All this hashed together in a unique SHA-1 identifier.

**Wait, does that make Git a blockchain?**

Because of cryptocurrencies, blockchains are all the hype these days.
And yes, *in a way*, Git is a blockchain: it’s a sequence of blocks
(commits) tied together by cryptographic means in a way that guarantee
that each single element is associated to the whole history of the
structure. Don’t take the comparison too seriously, though: we don’t
need a GitCoin. Really, we don’t.

## 6\. Reading commit data: checkout

It’s all well that commits hold a lot more than files and directories in
a given state, but that doesn’t make them really useful. It’s probably
time to start implementing tree objects as well, so we’ll be able to
checkout commits into the work tree.

### 6.1. What’s in a tree?

Informally, a tree describes the content of the work tree, that it, it
associates blobs to paths. It’s an array of three-element tuples made of
a file mode, a path (relative to the worktree) and a SHA-1. A typical
tree contents may look like this:

<table>
<thead>
<tr class="header">
<th>Mode</th>
<th>SHA-1</th>
<th>Path</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><code>100644</code></td>
<td><code>894a44cc066a027465cd26d634948d56d13af9af</code></td>
<td><code>.gitignore</code></td>
</tr>
<tr class="even">
<td><code>100644</code></td>
<td><code>94a9ed024d3859793618152ea559a168bbcbb5e2</code></td>
<td><code>LICENSE</code></td>
</tr>
<tr class="odd">
<td><code>100644</code></td>
<td><code>bab489c4f4600a38ce6dbfd652b90383a4aa3e45</code></td>
<td><code>README.md</code></td>
</tr>
<tr class="even">
<td><code>100644</code></td>
<td><code>6d208e47659a2a10f5f8640e0155d9276a2130a9</code></td>
<td><code>src</code></td>
</tr>
<tr class="odd">
<td><code>040000</code></td>
<td><code>e7445b03aea61ec801b20d6ab62f076208b7d097</code></td>
<td><code>tests</code></td>
</tr>
<tr class="even">
<td><code>040000</code></td>
<td><code>d5ec863f17f3a2e92aa8f6b66ac18f7b09fd1b38</code></td>
<td><code>main.c</code></td>
</tr>
</tbody>
</table>

Mode is just the file’s
[mode](https://en.wikipedia.org/wiki/File_system_permissions), path is
its location. The SHA-1 refers to either a blob or another tree object.
If a blob, the path is a file, if a tree, it’s directory. To instantiate
this tree in the filesystem, we would begin by loading the object
associated to the first path (`.gitignore`) and check its type. Since
it’s a blob, we’ll just create a file called `.gitignore` with this
blob’s contents; and same for `LICENSE` and `README.md`. But the object
associated with `src` is not a blob, but another tree: we’ll create the
directory `src` and repeat the same operation in that directory with the
new tree.

**A path is a single filesystem entry**

The path identifies exactly one file or directory. Not two, not three.
If you have five levels of nested directories, even if four are empty
save the next directory, you’re going to need five tree objects
recursively referring to one another. You cannot take the shortcut of
putting a full path in a single tree entry, like
`dir1/dir2/dir3/dir4/dir5`.

### 6.2. Parsing trees

Unlike tags and commits, tree objects are binary objects, but their
format is actually quite simple. A tree is the concatenation of records
of the format:

``` example
[mode] space [path] 0x00 [sha-1]
```

  - `[mode]` is up to six bytes and is an octal representation of a file
    **mode**, stored in ASCII. For example, 100644 is encoded with byte
    values 49 (ASCII “1”), 48 (ASCII “0”), 48, 54, 52, 52. The first two
    digits encode the file type (file, directory, symlink or submodule),
    the last four the permissions.
  - It’s followed by 0x20, an ASCII **space**;
  - Followed by the null-terminated (0x00) **path**;
  - Followed by the object’s **SHA-1** in binary encoding, on 20 bytes.

The parser is going to be quite simple. First, create a tiny object
wrapper for a single record (a leaf, a single path):

``` src src-python
class GitTreeLeaf (object):
    def __init__(self, mode, path, sha):
        self.mode = mode
        self.path = path
        self.sha = sha
```

Because a tree object is just the repetition of the same fundamental
data structure, we write the parser in two functions. First, a parser to
extract a single record, which returns parsed data and the position it
reached in input data:

``` src src-python
def tree_parse_one(raw, start=0):
    # Find the space terminator of the mode
    x = raw.find(b' ', start)
    assert x-start == 5 or x-start==6

    # Read the mode
    mode = raw[start:x]
    if len(mode) == 5:
        # Normalize to six bytes.
        mode = b" " + mode

    # Find the NULL terminator of the path
    y = raw.find(b'\x00', x)
    # and read the path
    path = raw[x+1:y]

    # Read the SHA and convert to a hex string
    sha = format(int.from_bytes(raw[y+1:y+21], "big"), "040x")
    return y+21, GitTreeLeaf(mode, path.decode("utf8"), sha)
```

And the “real” parser which just calls the previous one in a loop, until
input data is exhausted.

``` src src-python
def tree_parse(raw):
    pos = 0
    max = len(raw)
    ret = list()
    while pos < max:
        pos, data = tree_parse_one(raw, pos)
        ret.append(data)

    return ret
```

We’ll finally need a serializer to write trees back. Because we may have
added or modified entries, we need to sort them again. Consistently
sorting matters, because we need to respect git’s [identity
rules](#org78b0903), which says that no two equivalent object can have a
different hash — but differently sorted trees with the same contents
*would* be equivalent (describing the same directory structure), and
still numerically distinct (different SHA-1 identifiers). Incorrectly
sorted trees are invalid, but *git doesn’t enforce that*. I created some
invalid trees by accident writing wyag, and all I got was weird bugs in
`git status` (specifically, `status` would report an actually clean
worktree as fully modified). We don’t want that.

The ordering function is quite simple, with an unexpected twist. are
Entries sorted by name, alphabetically, *but* directories (that is, tree
entries) are sorted with a final `/` added. It matters, because it means
that if `whatever` names a regular file, it will sort *before*
`whatever.c`, but if `whatever` is a dir, it will sort *after*, as
`whatever/`. (I’m not sure why git does that. If you’re curious, see the
function `base_name_compare` in `tree.c` in the git source)

``` src src-python
# Notice this isn't a comparison function, but a conversion function.
# Python's default sort doesn't accept a custom comparison function,
# like in most languages, but a `key` arguments that returns a new
# value, which is compared using the default rules.  So we just return
# the leaf name, with an extra / if it's a directory.
def tree_leaf_sort_key(leaf):
    if leaf.mode.startswith(b"10"):
        return leaf.path
    else:
        return leaf.path + "/"
```

Then the serializer itself. This one is very simple: we sort the items
using our newly created function as a transformer, then write them in
order.

``` src src-python
def tree_serialize(obj):
    obj.items.sort(key=tree_leaf_sort_key)
    ret = b''
    for i in obj.items:
        ret += i.mode
        ret += b' '
        ret += i.path.encode("utf8")
        ret += b'\x00'
        sha = int(i.sha, 16)
        ret += sha.to_bytes(20, byteorder="big")
    return ret
```

And now we just have to combine all that into a class:

``` src src-python
class GitTree(GitObject):
    fmt=b'tree'

    def deserialize(self, data):
        self.items = tree_parse(data)

    def serialize(self):
        return tree_serialize(self)

    def init(self):
        self.items = list()
```

### 6.3. Showing trees: ls-tree

While we’re at it, let’s add the `ls-tree` command to wyag. It’s so easy
there’s no reason not to. `git ls-tree [-r] TREE` simply prints the
contents of a tree, recursively with the `-r` flag. In recursive mode,
it doesn’t show subtrees, just final objects with their full paths.

``` src src-python
argsp = argsubparsers.add_parser("ls-tree", help="Pretty-print a tree object.")
argsp.add_argument("-r",
                   dest="recursive",
                   action="store_true",
                   help="Recurse into sub-trees")

argsp.add_argument("tree",
                   help="A tree-ish object.")

def cmd_ls_tree(args):
    repo = repo_find()
    ls_tree(repo, args.tree, args.recursive)

def ls_tree(repo, ref, recursive=None, prefix=""):
    sha = object_find(repo, ref, fmt=b"tree")
    obj = object_read(repo, sha)
    for item in obj.items:
        if len(item.mode) == 5:
            type = item.mode[0:1]
        else:
            type = item.mode[0:2]

        match type: # Determine the type.
            case b'04': type = "tree"
            case b'10': type = "blob" # A regular file.
            case b'12': type = "blob" # A symlink. Blob contents is link target.
            case b'16': type = "commit" # A submodule
            case _: raise Exception("Weird tree leaf mode {}".format(item.mode))

        if not (recursive and type=='tree'): # This is a leaf
            print("{0} {1} {2}\t{3}".format(
                "0" * (6 - len(item.mode)) + item.mode.decode("ascii"),
                # Git's ls-tree displays the type
                # of the object pointed to.  We can do that too :)
                type,
                item.sha,
                os.path.join(prefix, item.path)))
        else: # This is a branch, recurse
            ls_tree(repo, item.sha, recursive, os.path.join(prefix, item.path))
```

### 6.4. The checkout command

`git checkout` simply instantiates a commit in the worktree. We’re going
to oversimplify the actual git command to make our implementation clear
and understandable. We’re also going to add a few safeguards. Here’s how
our version of checkout will work:

  - It will take two arguments: a commit, and a directory. Git checkout
    only needs a commit.
  - It will then instantiate the tree in the directory, **if and only if
    the directory is empty**. Git is full of safeguards to avoid
    deleting data, which would be too complicated and unsafe to try to
    reproduce in wyag. Since the point of wyag is to demonstrate git,
    not to produce a working implementation, this limitation is
    acceptable.

Let’s get started. As usual, we need a subparser:

``` src src-python
argsp = argsubparsers.add_parser("checkout", help="Checkout a commit inside of a directory.")

argsp.add_argument("commit",
                   help="The commit or tree to checkout.")

argsp.add_argument("path",
                   help="The EMPTY directory to checkout on.")
```

A wrapper function:

``` src src-python
def cmd_checkout(args):
    repo = repo_find()

    obj = object_read(repo, object_find(repo, args.commit))

    # If the object is a commit, we grab its tree
    if obj.fmt == b'commit':
        obj = object_read(repo, obj.kvlm[b'tree'].decode("ascii"))

    # Verify that path is an empty directory
    if os.path.exists(args.path):
        if not os.path.isdir(args.path):
            raise Exception("Not a directory {0}!".format(args.path))
        if os.listdir(args.path):
            raise Exception("Not empty {0}!".format(args.path))
    else:
        os.makedirs(args.path)

    tree_checkout(repo, obj, os.path.realpath(args.path))
```

And a function to do the actual work:

``` src src-python
def tree_checkout(repo, tree, path):
    for item in tree.items:
        obj = object_read(repo, item.sha)
        dest = os.path.join(path, item.path)

        if obj.fmt == b'tree':
            os.mkdir(dest)
            tree_checkout(repo, obj, dest)
        elif obj.fmt == b'blob':
            # @TODO Support symlinks (identified by mode 12****)
            with open(dest, 'wb') as f:
                f.write(obj.blobdata)
```

## 7\. Refs, tags and branches

### 7.1. What a ref is, and the show-ref command

As of now, the only way we can refer to objects is by their full
hexadecimal identifier. In git, we actually rarely see those, except to
talk about a specific commit. But in general, we’re talking about HEAD,
about some branch called names like `main` or `feature/more-bombs`, and
so on. This is handled by a simple mechanism called references.

Git references, or refs, are probably the most simple type of things git
holds. They live in subdirectories of `.git/refs`, and are text files
containing a hexadecimal representation of an object’s hash, encoded in
ASCII. They’re actually as simple as this:

``` example
6071c08bcb4757d8c89a30d9755d2466cef8c1de
```

Refs can also refer to another reference, and thus only indirectly to an
object, in which case they look like this:

``` example
ref: refs/remotes/origin/master
```

**Direct and indirect references**

From now on, I will call a reference of the form `ref:
path/to/other/ref` an **indirect** reference, and a ref with a SHA-1
object ID a **direct reference**.

This section will describe the uses of refs. For now, all that matter is
this:

  - they’re text files, in the `.git/refs` hierarchy;
  - they hold the SHA-1 identifier of an object, or a reference to
    another reference, ultimately to a SHA-1 (no loops\!)

To work with refs, we’re first going to need a simple recursive solver
that will take a ref name, follow eventual recursive references (refs
whose content begin with `ref:`, as exemplified above) and return a
SHA-1 identifier:

``` src src-python
def ref_resolve(repo, ref):
    path = repo_file(repo, ref)

    # Sometimes, an indirect reference may be broken.  This is normal
    # in one specific case: we're looking for HEAD on a new repository
    # with no commits.  In that case, .git/HEAD points to "ref:
    # refs/heads/main", but .git/refs/heads/main doesn't exist yet
    # (since there's no commit for it to refer to).
    if not os.path.isfile(path):
        return None

    with open(path, 'r') as fp:
        data = fp.read()[:-1]
        # Drop final \n ^^^^^
    if data.startswith("ref: "):
        return ref_resolve(repo, data[5:])
    else:
        return data
```

Let’s create two small functions, and implement the `show-refs` command
— it just lists all references in a repository. First, a stupid
recursive function to collect refs and return them as a dict:

``` src src-python
def ref_list(repo, path=None):
    if not path:
        path = repo_dir(repo, "refs")
    ret = collections.OrderedDict()
    # Git shows refs sorted.  To do the same, we use
    # an OrderedDict and sort the output of listdir
    for f in sorted(os.listdir(path)):
        can = os.path.join(path, f)
        if os.path.isdir(can):
            ret[f] = ref_list(repo, can)
        else:
            ret[f] = ref_resolve(repo, can)

    return ret
```

And, as usual, a subparser, a bridge, and a (recursive) worker function:

``` src src-python
argsp = argsubparsers.add_parser("show-ref", help="List references.")

def cmd_show_ref(args):
    repo = repo_find()
    refs = ref_list(repo)
    show_ref(repo, refs, prefix="refs")

def show_ref(repo, refs, with_hash=True, prefix=""):
    for k, v in refs.items():
        if type(v) == str:
            print ("{0}{1}{2}".format(
                v + " " if with_hash else "",
                prefix + "/" if prefix else "",
                k))
        else:
            show_ref(repo, v, with_hash=with_hash, prefix="{0}{1}{2}".format(prefix, "/" if prefix else "", k))
```

### 7.2. Tags as references

The most simple use of refs is tags. A tag is just a user-defined name
for an object, often a commit. A very common use of tags is identifying
software releases: You’ve just merged the last commit of, say, version
12.78.52 of your program, so your most recent commit (let’s call it
`6071c08`) *is* your version 12.78.52. To make this association
explicit, all you have to do is:

``` src src-shell
git tag v12.78.52 6071c08
# the object hash ^here^^ is optional and defaults to HEAD.
```

This creates a new tag, called `v12.78.52`, pointing at `6071c08`.
Tagging is like aliasing: a tag introduces a new way to refer to an
existing object. After the tag is created, the name `v12.78.52` refers
to `6071c08`. For example, these two commands are now perfectly
equivalent:

``` src src-shell
git checkout v12.78.52
git checkout 6071c08
```

Versions are a common use of tags, but like almost everything in Git,
tags have no predefined semantics: they mean whatever you want them to
mean, and can point to whichever object you want, you can even tag
*blobs*\!

### 7.3. Lightweight tags and tag objects, and parsing the latter

You’ve probably guessed already that tags are actually refs. They live
in the `.git/refs/tags/` hierarchy. The only point worth noting is that
they come in two flavors: lightweight tags and tags objects.

  - “Lightweight” tags  
    are just regular refs to a commit, a tree or a blob.
  - Tag objects  
    are regular refs pointing to an object of type `tag`. Unlike
    lightweight tags, tag objects have an author, a date, an optional
    PGP signature and an optional annotation. Their format is the same
    as a commit object.

We don’t even need to implement tag objects, we can reuse `GitCommit`
and just change the `fmt` field:

``` src src-python
class GitTag(GitCommit):
    fmt = b'tag'
```

And now we support tags.

### 7.4. The tag command

Let’s add the `tag` command. In Git, it does two things: it creates a
new tag or list existing tags (by default). So you can invoke it with:

``` src src-shell
git tag                  # List all tags
git tag NAME [OBJECT]    # create a new *lightweight* tag NAME, pointing
                         # at HEAD (default) or OBJECT
git tag -a NAME [OBJECT] # create a new tag *object* NAME, pointing at
                         # HEAD (default) or OBJECT
```

This translates to argparse as follows. Notice we ignore the mutual
exclusion between `--list` and `[-a] name [object]`, which seems too
complicated for argparse.

``` src src-python
argsp = argsubparsers.add_parser(
    "tag",
    help="List and create tags")

argsp.add_argument("-a",
                   action="store_true",
                   dest="create_tag_object",
                   help="Whether to create a tag object")

argsp.add_argument("name",
                   nargs="?",
                   help="The new tag's name")

argsp.add_argument("object",
                   default="HEAD",
                   nargs="?",
                   help="The object the new tag will point to")
```

The `cmd_tag` function will dispatch behavior (list or create) depending
on whether or not `name` is provided.

``` src src-python
def cmd_tag(args):
    repo = repo_find()

    if args.name:
        tag_create(repo,
                   args.name,
                   args.object,
                   type="object" if args.create_tag_object else "ref")
    else:
        refs = ref_list(repo)
        show_ref(repo, refs["tags"], with_hash=False)
```

And we just need one more function to actually create the tag:

``` src src-python
def tag_create(repo, name, ref, create_tag_object=False):
    # get the GitObject from the object reference
    sha = object_find(repo, ref)

    if create_tag_object:
        # create tag object (commit)
        tag = GitTag(repo)
        tag.kvlm = collections.OrderedDict()
        tag.kvlm[b'object'] = sha.encode()
        tag.kvlm[b'type'] = b'commit'
        tag.kvlm[b'tag'] = name.encode()
        # Feel free to let the user give their name!
        # Notice you can fix this after commit, read on!
        tag.kvlm[b'tagger'] = b'Wyag <wyag@example.com>'
        # …and a tag message!
        tag.kvlm[None] = b"A tag generated by wyag, which won't let you customize the message!"
        tag_sha = object_write(tag)
        # create reference
        ref_create(repo, "tags/" + name, tag_sha)
    else:
        # create lightweight tag (ref)
        ref_create(repo, "tags/" + name, sha)

def ref_create(repo, ref_name, sha):
    with open(repo_file(repo, "refs/" + ref_name), 'w') as fp:
        fp.write(sha + "\n")
```

### 7.5. What’s a branch?

Tags are done. Now for another big chunk: branches.

It’s time to address the elephant in the room: like most Git users, wyag
still doesn’t have any idea what a branch is. It currently treats a
repository as a bunch of disorganized objects, some of them commits, and
has no representation whatsoever of the fact that commits are grouped in
branches, and that at every point in time there’s a commit that’s
`HEAD`, *ie*, the **head** commit (or “tip”) of the **active** branch.

So, what’s a branch? The answer is actually surprisingly simple, but it
may also end up being simply surprising: **a branch is a reference to a
commit**. You could even say that a branch is a kind of a name for a
commit. In this regard, a branch is exactly the same thing as a tag.
Tags are refs that live in `.git/refs/tags`, branches are refs that live
in `.git/refs/heads`.

There are, of course, differences between a branch and a tag:

1.  Branches are references to a *commit*, tags can refer to any object;
2.  Most importantly, the branch ref is updated at each commit. This
    means that whenever you commit, Git actually does this:
    1.  a new commit object is created, with the current branch’s
        (commit\!) ID as its parent;
    2.  the commit object is hashed and stored;
    3.  the branch ref is updated to refer to the new commit’s hash.

That’s all.

But what about the **current** branch? It’s actually even easier. It’s a
ref file outside of the `refs` hierarchy, in `.git/HEAD`, which is an
**indirect** ref (that is, it is of the form `ref: path/to/other/ref`,
and not a simple hash).

**Detached HEAD**

When you just checkout a random commit, git will warn you it’s in
“detached HEAD state”. This means you’re not on any branch anymore. In
this case, `.git/HEAD` is a **direct** reference: it contains a SHA-1.

### 7.6. Referring to objects: the `object_find` function

#### 7.6.1. Resolving names

Remember when we’ve created [the stupid `object_find`
function](#org9618193) that would take four arguments, return the second
unmodified and ignore the other three? It’s time to replace it by
something more useful. We’re going to implement a small, but usable,
subset of the actual Git name resolution algorithm. The new
`object_find()` will work in two steps: first, given a name, it will
return a complete sha-1 hash. For example, with `HEAD`, it will return
the hash of the head commit of the current branch, etc. More precisely,
this name resolution function will work like this:

  - If `name` is `HEAD`, it will just resolve `.git/HEAD`;
  - If `name` is a full hash, this hash is returned unmodified.
  - If `name` looks like a short hash, it will collect objects whose
    full hash begin with this short hash.
  - At last, it will resolve tags and branches matching name.

Notice how the last two steps *collect* values: the first two are
absolute references, so we can safely return a result. But short hashes
or branch names can be ambiguous, we want to enumerate all possible
meanings of the name and raise an error if we’ve found more than 1.

**Short hashes**

For convenience, Git allows to refer to hashes by a prefix of their
name. For example, `5bd254aa973646fa16f66d702a5826ea14a3eb45` can be
referred to as `5bd254`. This is called a “short hash”.

``` src src-python
def object_resolve(repo, name):
    """Resolve name to an object hash in repo.

This function is aware of:

 - the HEAD literal
    - short and long hashes
    - tags
    - branches
    - remote branches"""
    candidates = list()
    hashRE = re.compile(r"^[0-9A-Fa-f]{4,40}$")

    # Empty string?  Abort.
    if not name.strip():
        return None

    # Head is nonambiguous
    if name == "HEAD":
        return [ ref_resolve(repo, "HEAD") ]

    # If it's a hex string, try for a hash.
    if hashRE.match(name):
        # This may be a hash, either small or full.  4 seems to be the
        # minimal length for git to consider something a short hash.
        # This limit is documented in man git-rev-parse
        name = name.lower()
        prefix = name[0:2]
        path = repo_dir(repo, "objects", prefix, mkdir=False)
        if path:
            rem = name[2:]
            for f in os.listdir(path):
                if f.startswith(rem):
                    # Notice a string startswith() itself, so this
                    # works for full hashes.
                    candidates.append(prefix + f)

    # Try for references.
    as_tag = ref_resolve(repo, "refs/tags/" + name)
    if as_tag: # Did we find a tag?
        candidates.append(as_tag)

    as_branch = ref_resolve(repo, "refs/heads/" + name)
    if as_branch: # Did we find a branch?
        candidates.append(as_branch)

    return candidates
```

The second step is to follow the object we found to an object of the
required type, if a type argument was provided. Since we only need to
handle trivial cases, this is a very simple iterative process:

  - If we have a tag and `fmt` is anything else, we follow the tag.
  - If we have a commit and `fmt` is tree, we return this commit’s tree
    object
  - In all other situations, we bail out: nothing else makes sense.

(The process is iterative because it may take an undefined number of
steps, since tags themselves can be tagged)

``` src src-python
def object_find(repo, name, fmt=None, follow=True):
      sha = object_resolve(repo, name)

      if not sha:
          raise Exception("No such reference {0}.".format(name))

      if len(sha) > 1:
          raise Exception("Ambiguous reference {0}: Candidates are:\n - {1}.".format(name,  "\n - ".join(sha)))

      sha = sha[0]

      if not fmt:
          return sha

      while True:
          obj = object_read(repo, sha)
          #     ^^^^^^^^^^^ < this is a bit agressive: we're reading
          # the full object just to get its type.  And we're doing
          # that in a loop, albeit normally short.  Don't expect
          # high performance here.

          if obj.fmt == fmt:
              return sha

          if not follow:
              return None

          # Follow tags
          if obj.fmt == b'tag':
                sha = obj.kvlm[b'object'].decode("ascii")
          elif obj.fmt == b'commit' and fmt == b'tree':
                sha = obj.kvlm[b'tree'].decode("ascii")
          else:
              return None
```

With the new `object_find()`, the CLI wyag becomes a bit more usable.
You can now do things like:

``` example
$ wyag checkout v3.11 # A tag
$ wyag checkout feature/explosions # A branch
$ wyag ls-tree -r HEAD # The active branch or commit.  There's also a
                       # follow here: HEAD is actually a commit.
$ wyag cat-file blob e0695f # A short hash
$ wyag cat-file tree master # A branch, as a tree (another "follow")
```

#### 7.6.2. The rev-parse command

Let’s implement `wyag rev-parse`. The `git rev-parse` commands does a
lot, but one of its use cases, the one we’re going to clone, is solving
references. For the purpose of further testing the “follow” feature of
`object_find`, we’ll add an optional `wyag-type` argument to its
interface.

``` src src-python
argsp = argsubparsers.add_parser(
    "rev-parse",
    help="Parse revision (or other objects) identifiers")

argsp.add_argument("--wyag-type",
                   metavar="type",
                   dest="type",
                   choices=["blob", "commit", "tag", "tree"],
                   default=None,
                   help="Specify the expected type")

argsp.add_argument("name",
                   help="The name to parse")
```

The bridge does all the job:

``` src src-python
def cmd_rev_parse(args):
    if args.type:
        fmt = args.type.encode()
    else:
        fmt = None

    repo = repo_find()

    print (object_find(repo, args.name, fmt, follow=True))
```

And it works:

``` example
$ wyag rev-parse --wyag-type commit HEAD
6c22393f5e3830d15395fd8d2f8b0cf8eb40dd58
$ wyag rev-parse --wyag-type tree HEAD
11d33fad71dbac72840aff1447e0d080c7484361
$ wyag rev-parse --wyag-type tree HEAD
None
```

## 8\. Working with the staging area and the index file

### 8.1. What’s the index file?

This final step will bring us to where commits happen (although actually
creating them is for the next section\!)

You probably know that to commit in Git, you first “stage” some changes,
using `git add` and `git rm`, and only *then* do you commit those
changes. This intermediate stage between the last and the next commit is
called the **staging area**.

It would seem natural to use a commit or tree object to represent the
staging area, but Git actually and uses a completely different
mechanism, in the form of what it calls the **index file**.

After a commit, the index file is a sort of copy of that commit: it
holds the same path/blob association than the corresponding tree. But it
also holds extra information about files in the worktree, like their
creation/modification time, so `git status` doesn’t often need to
actually compare files: it just checks that their modification time is
the same as the one stored in the index file, and only if it isn’t does
it perform an actual comparison.

You can thus consider the index file as a three-way association list:
not only paths with blobs, but also paths with actual filesystem
entries.

Another important characteristic of the **index file** is that unlike a
tree, it can represent inconsistent states, like a merge conflict,
whereas a tree is always a complete, unambiguous representation.

When you commit, what git actually does is turn the index file into a
new tree object. To summarize:

1.  When the repository is “clean”, the index file holds the exact same
    contents as the HEAD commit, plus metadata about the corresponding
    filesystem entries. For instance, it may contain something like:
    
    > There’s a file called `src/disp.c` whose contents are blob
    > 797441c76e59e28794458b39b0f1eff4c85f4fa0. The real `src/disp.c`
    > file, in the worktree, was created on 2023-07-15
    > 15:28:29.168572151, and last modified 2023-07-15
    > 15:28:29.1689427709. It is stored on device 65026, inode 8922881.

2.  When you `git add` or `git rm`, the index file is modified
    accordingly. In the example above, if you modify `src/disp.c`, and
    `add` your changes, the index file will be updated with a new blob
    ID (the blob itself will also be created in the process, of course),
    and the various file metadata will be updated as well so `git
    status` knows when not to compare file contents.

3.  When you `git commit` those changes, a new tree is produced from the
    index file, a new commit object is generated with that tree,
    branches are updated and we’re done.

**A note on words**

The staging area and the index are thus the same thing, but the name
“staging area” is more the name of the git user-exposed feature (that
could have been implemented otherwise), the abstraction if you will;
while “index file” refers specifically to the way this abstract feature
is actually implemented in git.

### 8.2. Parsing the index

The index file is by far the most complicated piece of data a Git
repository can hold. Its complete documentation can be found in Git
source tree at `Documentation/gitformat-index.txt`; you can browse it
[on the Github
mirror](https://github.com/git/git/blob/master/Documentation/gitformat-index.txt).
It’s made of three parts:

  - An header with the format version number and the number of entries
    the index holds;
  - A series of entries, sorted, each representing a file; padded to
    multiple of 8 bytes.
  - A series of optional extensions, which we’ll ignore.

The first thing we need to represent is a single entry. It actually
holds quite a lot of stuff, I’m leaving the details in comments. It’s
worth observing that an entry stores **both** the SHA-1 of the
associated blob in the object store *and* a ton of metadata about the
actual file on the actual filesystem. Again, this is because `git/wyag
status` will need to determine which files in the index were modified:
it is much more efficient to begin by checking the last-modified
timestamp and comparing it with a known values, before comparing actual
files.

``` src src-python
class GitIndexEntry (object):
    def __init__(self, ctime=None, mtime=None, dev=None, ino=None,
                 mode_type=None, mode_perms=None, uid=None, gid=None,
                 fsize=None, sha=None, flag_assume_valid=None,
                 flag_stage=None, name=None):
      # The last time a file's metadata changed.  This is a pair
      # (timestamp in seconds, nanoseconds)
      self.ctime = ctime
      # The last time a file's data changed.  This is a pair
      # (timestamp in seconds, nanoseconds)
      self.mtime = mtime
      # The ID of device containing this file
      self.dev = dev
      # The file's inode number
      self.ino = ino
      # The object type, either b1000 (regular), b1010 (symlink),
      # b1110 (gitlink).
      self.mode_type = mode_type
      # The object permissions, an integer.
      self.mode_perms = mode_perms
      # User ID of owner
      self.uid = uid
      # Group ID of ownner
      self.gid = gid
      # Size of this object, in bytes
      self.fsize = fsize
      # The object's SHA
      self.sha = sha
      self.flag_assume_valid = flag_assume_valid
      self.flag_stage = flag_stage
      # Name of the object (full path this time!)
      self.name = name
```

The index file is a binary file, likely for performance reasons. The
format is reasonably simple, though. It begins with a header with the
`DIRC` magic bytes, a version number and the total number of entries in
that index file. We create the `GitIndex` class to hold them:

``` src src-python
class GitIndex (object):
    version = None
    entries = []
    # ext = None
    # sha = None

    def __init__(self, version=2, entries=None):
        if not entries:
            entries = list()

        self.version = version
        self.entries = entries
```

And a parser to read index files into those objects. After reading the
12-bytes header, we just parse entries in the order they appear. An
entry begins with a set of fixed-length data, followed by a
variable-length name.

The code is quite straightforward, but as it’s reading a binary format,
it feels more messy than what we did so far. We use the
`int.from_bytes(bytes, endianness)` a lot to read raw bytes into an
integer, and just a few bitwise operations to separate data that share
the same byte.

(This format was probably designed so index files could just be
`mmapp()ed` to memory, and read directly as C structs, with an index
built in O(n) time in most cases. This kind of approach tends to produce
more elegant code in C than in Python…)

``` src src-python
def index_read(repo):
    index_file = repo_file(repo, "index")

    # New repositories have no index!
    if not os.path.exists(index_file):
        return GitIndex()

    with open(index_file, 'rb') as f:
        raw = f.read()

    header = raw[:12]
    signature = header[:4]
    assert signature == b"DIRC" # Stands for "DirCache"
    version = int.from_bytes(header[4:8], "big")
    assert version == 2, "wyag only supports index file version 2"
    count = int.from_bytes(header[8:12], "big")

    entries = list()

    content = raw[12:]
    idx = 0
    for i in range(0, count):
        # Read creation time, as a unix timestamp (seconds since
        # 1970-01-01 00:00:00, the "epoch")
        ctime_s =  int.from_bytes(content[idx: idx+4], "big")
        # Read creation time, as nanoseconds after that timestamps,
        # for extra precision.
        ctime_ns = int.from_bytes(content[idx+4: idx+8], "big")
        # Same for modification time: first seconds from epoch.
        mtime_s = int.from_bytes(content[idx+8: idx+12], "big")
        # Then extra nanoseconds
        mtime_ns = int.from_bytes(content[idx+12: idx+16], "big")
        # Device ID
        dev = int.from_bytes(content[idx+16: idx+20], "big")
        # Inode
        ino = int.from_bytes(content[idx+20: idx+24], "big")
        # Ignored.
        unused = int.from_bytes(content[idx+24: idx+26], "big")
        assert 0 == unused
        mode = int.from_bytes(content[idx+26: idx+28], "big")
        mode_type = mode >> 12
        assert mode_type in [0b1000, 0b1010, 0b1110]
        mode_perms = mode & 0b0000000111111111
        # User ID
        uid = int.from_bytes(content[idx+28: idx+32], "big")
        # Group ID
        gid = int.from_bytes(content[idx+32: idx+36], "big")
        # Size
        fsize = int.from_bytes(content[idx+36: idx+40], "big")
        # SHA (object ID).  We'll store it as a lowercase hex string
        # for consistency.
        sha = format(int.from_bytes(content[idx+40: idx+60], "big"), "040x")
        # Flags we're going to ignore
        flags = int.from_bytes(content[idx+60: idx+62], "big")
        # Parse flags
        flag_assume_valid = (flags & 0b1000000000000000) != 0
        flag_extended = (flags & 0b0100000000000000) != 0
        assert not flag_extended
        flag_stage =  flags & 0b0011000000000000
        # Length of the name.  This is stored on 12 bits, some max
        # value is 0xFFF, 4095.  Since names can occasionally go
        # beyond that length, git treats 0xFFF as meaning at least
        # 0xFFF, and looks for the final 0x00 to find the end of the
        # name --- at a small, and probably very rare, performance
        # cost.
        name_length = flags & 0b0000111111111111

        # We've read 62 bytes so far.
        idx += 62

        if name_length < 0xFFF:
            assert content[idx + name_length] == 0x00
            raw_name = content[idx:idx+name_length]
            idx += name_length + 1
        else:
            print("Notice: Name is 0x{:X} bytes long.".format(name_length))
            # This probably wasn't tested enough.  It works with a
            # path of exactly 0xFFF bytes.  Any extra bytes broke
            # something between git, my shell and my filesystem.
            null_idx = content.find(b'\x00', idx + 0xFFF)
            raw_name = content[idx: null_idx]
            idx = null_idx + 1

        # Just parse the name as utf8.
        name = raw_name.decode("utf8")

        # Data is padded on multiples of eight bytes for pointer
        # alignment, so we skip as many bytes as we need for the next
        # read to start at the right position.

        idx = 8 * ceil(idx / 8)

        # And we add this entry to our list.
        entries.append(GitIndexEntry(ctime=(ctime_s, ctime_ns),
                                     mtime=(mtime_s,  mtime_ns),
                                     dev=dev,
                                     ino=ino,
                                     mode_type=mode_type,
                                     mode_perms=mode_perms,
                                     uid=uid,
                                     gid=gid,
                                     fsize=fsize,
                                     sha=sha,
                                     flag_assume_valid=flag_assume_valid,
                                     flag_stage=flag_stage,
                                     name=name))

    return GitIndex(version=version, entries=entries)
```

### 8.3. The ls-files command

`git ls-files` displays the names of files in the staging area, with, as
usual, a ton of options. Our `ls-files` will be much simpler, *but*
we’ll add a `--verbose` option that doesn’t exist in git, just so we
can display every single bit of info in the index file.

``` src src-python
argsp = argsubparsers.add_parser("ls-files", help = "List all the stage files")
argsp.add_argument("--verbose", action="store_true", help="Show everything.")

def cmd_ls_files(args):
  repo = repo_find()
  index = index_read(repo)
  if args.verbose:
    print("Index file format v{}, containing {} entries.".format(index.version, len(index.entries)))

  for e in index.entries:
    print(e.name)
    if args.verbose:
      print("  {} with perms: {:o}".format(
        { 0b1000: "regular file",
          0b1010: "symlink",
          0b1110: "git link" }[e.mode_type],
        e.mode_perms))
      print("  on blob: {}".format(e.sha))
      print("  created: {}.{}, modified: {}.{}".format(
        datetime.fromtimestamp(e.ctime[0])
        , e.ctime[1]
        , datetime.fromtimestamp(e.mtime[0])
        , e.mtime[1]))
      print("  device: {}, inode: {}".format(e.dev, e.ino))
      print("  user: {} ({})  group: {} ({})".format(
        pwd.getpwuid(e.uid).pw_name,
        e.uid,
        grp.getgrgid(e.gid).gr_name,
        e.gid))
      print("  flags: stage={} assume_valid={}".format(
        e.flag_stage,
        e.flag_assume_valid))
```

If you run ls-files, you’ll notice that on a “clean” worktree (an
unmodified checkout of `HEAD`), it lists all files on `HEAD`. Again, the
index is not a *delta* (a set of differences) from the `HEAD` commit,
but starts as a copy of it, in a different format.

### 8.4. A detour: the check-ignore command

We want to write `status`, but `status` needs to know about ignore
rules, that are stored in the various `.gitignore` files. So we first
need to add some rudimentary support for ignore files in `wyag`. We’ll
expose this support as the `check-ignore` command, which takes a list of
paths and outputs back those of those paths that should be ignored.

Again, the command parser is trivial:

``` src src-python
argsp = argsubparsers.add_parser("check-ignore", help = "Check path(s) against ignore rules.")
argsp.add_argument("path", nargs="+", help="Paths to check")
```

And the function is just as simple:

``` src src-python
def cmd_check_ignore(args):
  repo = repo_find()
  rules = gitignore_read(repo)
  for path in args.path:
      if check_ignore(rules, path):
        print(path)
```

But of course, most of the function we call don’t exist yet in wyag.
We’ll begin by writing a reader for rules in ignore files,
`gitignore_read()`. The syntax of those rules is quite simple: each line
in an ignore file is an exclusion pattern, so files that match this
pattern are ignored by `status`, `add -A` and so on. There are three
special cases, though:

1.  Lines that begin with an exclamation mark `!` *negate* the pattern
    (files that match this pattern are *included*, even they were
    ignored by an earlier pattern)
2.  Lines that begin with a dash `#` are comments, and are skipped.
3.  A backslash `\` at the beginning treats `!` and `#` as literal
    characters.

First, a parser for a single pattern. This parser returns a pair: the
pattern itself, and a boolean to indicate if files matching the pattern
*should* be excluded (`True`) or included (`False`). In other words,
`False` if the pattern did start with `!`, `True` otherwise.

``` src src-python
def gitignore_parse1(raw):
    raw = raw.strip() # Remove leading/trailing spaces

    if not raw or raw[0] == "#":
        return None
    elif raw[0] == "!":
        return (raw[1:], False)
    elif raw[0] == "\\":
        return (raw[1:], True)
    else:
        return (raw, True)
```

Parsing a file is just collecting all rules in that file. Notice this
function doesn’t parse *files*, but just lists of lines: that’s because
we’ll need to read rules from git blobs as well, and not just regular
files.

``` src src-python
def gitignore_parse(lines):
    ret = list()

    for line in lines:
        parsed = gitignore_parse1(line)
        if parsed:
            ret.append(parsed)

    return ret
```

Last thing we need to do is collect the various ignore files. They come
in two kinds:

  - Some of these files **live in the index**: they’re the various
    `gitignore` files. Emphasis on the plural; although there often is
    only one such file, at the root, there can be one in each directory,
    and it applies to this directory and its subdirectories. I’ll call
    those **scoped**, because they only apply to paths under their
    directory.
  - The others live **outside the index**. They’re the global ignore
    file (usually in `~/.config/git/ignore`) and the repository-specific
    `.git/info/exclude`. I call those **absolute**, because they apply
    everywhere, but at a lower priority.

Again, a class to hold that: a list of absolute rules, a dict (hashmap)
of relative rules. The keys to this hashmap are **directories**,
relative to the root of a worktree.

``` src src-python
class GitIgnore(object):
    absolute = None
    scoped = None

    def __init__(self, absolute, scoped):
        self.absolute = absolute
        self.scoped = scoped
```

And finally our function to collect all gitignore rules in a repository,
and return a `GitIgnore` object. Notice how it reads scoped files from
the index, and not the worktree: only *staged* `.gitignore` files matter
(also remember: HEAD is *already* staged — the staging area is a copy,
not a delta).

``` src src-python
def gitignore_read(repo):
    ret = GitIgnore(absolute=list(), scoped=dict())

    # Read local configuration in .git/info/exclude
    repo_file = os.path.join(repo.gitdir, "info/exclude")
    if os.path.exists(repo_file):
        with open(repo_file, "r") as f:
            ret.absolute.append(gitignore_parse(f.readlines()))

    # Global configuration
    if "XDG_CONFIG_HOME" in os.environ:
        config_home = os.environ["XDG_CONFIG_HOME"]
    else:
        config_home = os.path.expanduser("~/.config")
    global_file = os.path.join(config_home, "git/ignore")

    if os.path.exists(global_file):
        with open(global_file, "r") as f:
            ret.absolute.append(gitignore_parse(f.readlines()))

    # .gitignore files in the index
    index = index_read(repo)

    for entry in index.entries:
        if entry.name == ".gitignore" or entry.name.endswith("/.gitignore"):
            dir_name = os.path.dirname(entry.name)
            contents = object_read(repo, entry.sha)
            lines = contents.blobdata.decode("utf8").splitlines()
            ret.scoped[dir_name] = gitignore_parse(lines)
    return ret
```

We’re almost there. To tie everything together, we need the
`check_ignore` function that matches a path, relative to the root of a
worktree, against a set of rules. This is how this function will work:

  - It will first try to match this path against the **scoped** rules.
    It will do this from the deepest parent of the path to the farthest.
    That is, if the path is `src/support/w32/legacy/sound.c~`, it will
    first look for rules in `src/support/w32/legacy/.gitignore`, then
    `src/support/w32/.gitignore`, `src/support/.gitignore`, and so on up
    to simply `.gitignore"` at the root.
  - If nothing matches, it will continue with the **absolute** rules.

We write three small support functions. One to match a path against a
set of rules, and return the result of the last matching rule. Notice
how it’s not a real boolean functions, since it has **three** possible
return values: `True`, `False` but also `None`. It returns `None` if
nothing matched, so the caller knows it should continue trying with more
general ignore files (eg, go one directory level up).

``` src src-python
def check_ignore1(rules, path):
    result = None
    for (pattern, value) in rules:
        if fnmatch(path, pattern):
            result = value
    return result
```

A function to match against the dictionary of **scoped** rules (the
various `.gitignore` files). It just starts at the path’s directory then
moves up to the parent directory, recursively, until it has tested root.
Notice that this function (and the next two as well), never breaks
**inside** a given `.gitignore` file. Even if a rule matches, they keep
going through the file, because another rule there may negate reverse
the effect (rules are processed in order, so if you want to exclude
`*.c` but not `generator.c`, the general rule must come before the
specific one). But as soon as at least one rule matched in a file, we
drop the remaining files, because a more general file never cancels the
effect of a more specific one (this is why `check_ignore1` is ternary
and not boolean)

``` src src-python
def check_ignore_scoped(rules, path):
    parent = os.path.dirname(path)
    while True:
        if parent in rules:
            result = check_ignore1(rules[parent], path)
            if result != None:
                return result
        if parent == "":
            break
        parent = os.path.dirname(parent)
    return None
```

A much simpler function to match against the list of absolute rules.
Notice that the order we push those rules to the list matters (we *did*
read the repository rules before the global ones\!)

``` src src-python
def check_ignore_absolute(rules, path):
    parent = os.path.dirname(path)
    for ruleset in rules:
        result = check_ignore1(ruleset, path)
        if result != None:
            return result
    return False # This is a reasonable default at this point.
```

And finally, a function to bind them all.

``` src src-python
def check_ignore(rules, path):
    if os.path.isabs(path):
        raise Exception("This function requires path to be relative to the repository's root")

    result = check_ignore_scoped(rules.scoped, path)
    if result != None:
        return result

    return check_ignore_absolute(rules.absolute, path)
```

You can now call `wyag check-ignore`. On its own source tree:

``` example
$ wyag check-ignore hello.el hello.elc hello.html wyag.zip wyag.tar
hello.elc
hello.html
wyag.zip
```

**This is only an approximation**

This isn’t a perfect reimplementation. In particular, excluding whole
directories with a rule that’s only the directory name (eg
`__pycache__`) won’t work, because `fnmatch` would want the pattern as
`__pycache__/**`. If you really want to play with ignore rules, [this
may be a good starting
point](https://github.com/mherrmann/gitignore_parser).

### 8.5. The status command

`status` is more complex than `ls-files`, because it needs to compare
the index with both HEAD *and* the actual filesystem. You call `git
status` to know which files were added, removed or modified since the
last commit, and which of these changes are actually staged, and will
make it to the next commit. So `status` actually compares the `HEAD`
with the staging area, and the staging area with the worktree. This is
what its output looks like:

``` example
On branch master

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
    modified:   write-yourself-a-git.org

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
    modified:   write-yourself-a-git.org

Untracked files:
  (use "git add <file>..." to include in what will be committed)
    org-html-themes/
    wl-copy
```

We’ll implement `status` in three parts: first the active branch or
“detached HEAD”, then the difference between the index and the
worktree (“Changes not staged for commit”), then the difference between
HEAD and the index (“Changes to be committed” and “Untracked files”).

The public interface is dead simple, our status will take no argument:

``` src src-python
argsp = argsubparsers.add_parser("status", help = "Show the working tree status.")
```

And the bridge function just calls the three component functions in
order:

``` src src-python
def cmd_status(_):
    repo = repo_find()
    index = index_read(repo)

    cmd_status_branch(repo)
    cmd_status_head_index(repo, index)
    print()
    cmd_status_index_worktree(repo, index)
```

#### 8.5.1. Finding the active branch

First we need to know if we’re on a branch, and if so which one. We do
this by just looking at `.git/HEAD`. It should contain either an
hexadecimal ID (a ref to a commit, in detached HEAD state), or an
indirect reference to something in `refs/heads/`: the active branch. We
either return its name, or `False`.

``` src src-python
def branch_get_active(repo):
    with open(repo_file(repo, "HEAD"), "r") as f:
        head = f.read()

    if head.startswith("ref: refs/heads/"):
        return(head[16:-1])
    else:
        return False
```

Based on this, we can write the first of the three `cmd_status_*`
functions the bridge calls. This one prints the name of the active
branch, or the hash of the detached HEAD:

``` src src-python
def cmd_status_branch(repo):
    branch = branch_get_active(repo)
    if branch:
        print("On branch {}.".format(branch))
    else:
        print("HEAD detached at {}".format (object_find(repo, "HEAD")))
```

#### 8.5.2. Finding changes between HEAD and index

The second block of the status output is the “changes to be committed”,
that is, how the staging area differs from HEAD. To do this, we’re going
first to read the `HEAD` tree, and flatten it as a single dict (hashmap)
with full paths as keys, so it’s closer to the (flat) index associating
paths to blobs. Then we’ll just compare them and output their
differences.

First, a function to convert a tree (recursive, remember) to a (flat)
dict. And since trees are recursive, so the function itself is, again —
sorry about that :)

``` src src-python
def tree_to_dict(repo, ref, prefix=""):
  ret = dict()
  tree_sha = object_find(repo, ref, fmt=b"tree")
  tree = object_read(repo, tree_sha)

  for leaf in tree.items:
      full_path = os.path.join(prefix, leaf.path)

      # We read the object to extract its type (this is uselessly
      # expensive: we could just open it as a file and read the
      # first few bytes)
      is_subtree = leaf.mode.startswith(b'04')

      # Depending on the type, we either store the path (if it's a
      # blob, so a regular file), or recurse (if it's another tree,
      # so a subdir)
      if is_subtree:
        ret.update(tree_to_dict(repo, leaf.sha, full_path))
      else:
        ret[full_path] = leaf.sha

  return ret
```

And the command itself:

``` src src-python
def cmd_status_head_index(repo, index):
    print("Changes to be committed:")

    head = tree_to_dict(repo, "HEAD")
    for entry in index.entries:
        if entry.name in head:
            if head[entry.name] != entry.sha:
                print("  modified:", entry.name)
            del head[entry.name] # Delete the key
        else:
            print("  added:   ", entry.name)

    # Keys still in HEAD are files that we haven't met in the index,
    # and thus have been deleted.
    for entry in head.keys():
        print("  deleted: ", entry)
```

#### 8.5.3. Finding changes between index and worktree

``` src src-python
def cmd_status_index_worktree(repo, index):
    print("Changes not staged for commit:")

    ignore = gitignore_read(repo)

    gitdir_prefix = repo.gitdir + os.path.sep

    all_files = list()

    # We begin by walking the filesystem
    for (root, _, files) in os.walk(repo.worktree, True):
        if root==repo.gitdir or root.startswith(gitdir_prefix):
            continue
        for f in files:
            full_path = os.path.join(root, f)
            rel_path = os.path.relpath(full_path, repo.worktree)
            all_files.append(rel_path)

    # We now traverse the index, and compare real files with the cached
    # versions.

    for entry in index.entries:
        full_path = os.path.join(repo.worktree, entry.name)

        # That file *name* is in the index

        if not os.path.exists(full_path):
            print("  deleted: ", entry.name)
        else:
            stat = os.stat(full_path)

            # Compare metadata
            ctime_ns = entry.ctime[0] * 10**9 + entry.ctime[1]
            mtime_ns = entry.mtime[0] * 10**9 + entry.mtime[1]
            if (stat.st_ctime_ns != ctime_ns) or (stat.st_mtime_ns != mtime_ns):
                # If different, deep compare.
                # @FIXME This *will* crash on symlinks to dir.
                with open(full_path, "rb") as fd:
                    new_sha = object_hash(fd, b"blob", None)
                    # If the hashes are the same, the files are actually the same.
                    same = entry.sha == new_sha

                    if not same:
                        print("  modified:", entry.name)

        if entry.name in all_files:
            all_files.remove(entry.name)

    print()
    print("Untracked files:")

    for f in all_files:
        # @TODO If a full directory is untracked, we should display
        # its name without its contents.
        if not check_ignore(ignore, f):
            print(" ", f)
```

Our status function is done. It should output something like:

``` example
$ wyag status
On branch main.
Changes to be committed:
  added:    src/main.c

Changes not staged for commit:
  modified: build.py
  deleted:  README.org

Untracked files:
  src/cli.c
```

The real `status` is a lot smarter: it can detect renames, for example,
where ours cannot. Another significant difference worth mentioning is
that `git status` actually *writes* the index back if a file metadata
was modified, but not its content. You can see it with our special
ls-files:

``` example
$ wyag ls-files --verbose
Index file format v2, containing 1 entries.
file
  regular file with perms: 644
  on blob: f2f279981ce01b095c42ee7162aadf60185c8f67
  created: 2023-07-18 18:26:15.771460869, modified: 2023-07-18 18:26:15.771460869
  ...
$ touch file
$ git status > /dev/null
$ wyag ls-files --verbose
Index file format v2, containing 1 entries.
file
  regular file with perms: 644
  on blob: f2f279981ce01b095c42ee7162aadf60185c8f67
  created: 2023-07-18 18:26:41.421743098, modified: 2023-07-18 18:26:41.421743098
  ...
```

Notice how both timestamps, from the *index file*, were updated by `git
status` to reflect the changes in the real file’s metadata.

## 9\. Staging area and index, part 2: staging and committing

OK. Let’s create commits.

We have *almost* everything we need for that, except for three last
things:

1.  We need commands to modify the index, so our commits aren’t just a
    copy of their parent. Those commands are `add` and `rm`.
2.  These commands need to write the modified index back, since we
    commit *from the index*.
3.  And obviously, we’ll need the `commit` function and its associated
    `wyag commit` command.

### 9.1. Writing the index

We’ll start by writing the index. Roughly, we’re just serializing
everything back to binary. This is a bit tedious, but the code should be
straightforward. I’m leaving the gory details for the comments, but it’s
really just `index_read` in reverse — refer to it if needed, and the
`GitIndexEntry` class.

``` src src-python
def index_write(repo, index):
    with open(repo_file(repo, "index"), "wb") as f:

        # HEADER

        # Write the magic bytes.
        f.write(b"DIRC")
        # Write version number.
        f.write(index.version.to_bytes(4, "big"))
        # Write the number of entries.
        f.write(len(index.entries).to_bytes(4, "big"))

        # ENTRIES

        idx = 0
        for e in index.entries:
            f.write(e.ctime[0].to_bytes(4, "big"))
            f.write(e.ctime[1].to_bytes(4, "big"))
            f.write(e.mtime[0].to_bytes(4, "big"))
            f.write(e.mtime[1].to_bytes(4, "big"))
            f.write(e.dev.to_bytes(4, "big"))
            f.write(e.ino.to_bytes(4, "big"))

            # Mode
            mode = (e.mode_type << 12) | e.mode_perms
            f.write(mode.to_bytes(4, "big"))

            f.write(e.uid.to_bytes(4, "big"))
            f.write(e.gid.to_bytes(4, "big"))

            f.write(e.fsize.to_bytes(4, "big"))
            # @FIXME Convert back to int.
            f.write(int(e.sha, 16).to_bytes(20, "big"))

            flag_assume_valid = 0x1 << 15 if e.flag_assume_valid else 0

            name_bytes = e.name.encode("utf8")
            bytes_len = len(name_bytes)
            if bytes_len >= 0xFFF:
                name_length = 0xFFF
            else:
                name_length = bytes_len

            # We merge back three pieces of data (two flags and the
            # length of the name) on the same two bytes.
            f.write((flag_assume_valid | e.flag_stage | name_length).to_bytes(2, "big"))

            # Write back the name, and a final 0x00.
            f.write(name_bytes)
            f.write((0).to_bytes(1, "big"))

            idx += 62 + len(name_bytes) + 1

            # Add padding if necessary.
            if idx % 8 != 0:
                pad = 8 - (idx % 8)
                f.write((0).to_bytes(pad, "big"))
                idx += pad
```

### 9.2. The rm command

The easiest change we can do to an index is to remove an entry from it,
which mean that the next commit **won’t include** this file. This is
what the `git rm` command does.

`git rm` is **destructive**, and so is `wyag rm`. The command not only
modifies the index, it also removes file(s) from the worktree. Unlike
git, `wyag rm` doesn’t care if the file it removes isn’t saved. Proceed
with caution.

`rm` takes a single argument, a list of paths to remove:

``` src src-python
argsp = argsubparsers.add_parser("rm", help="Remove files from the working tree and the index.")
argsp.add_argument("path", nargs="+", help="Files to remove")

def cmd_rm(args):
  repo = repo_find()
  rm(repo, args.path)
```

The `rm` function is a bit long, but it’s very simple. It takes a
repository and a list of paths, reads that repository index, and removes
entries in the index that match this list. The optional arguments
control whether the function should actually delete the files, and
whether it should abort if some paths aren’t present on the index (both
those arguments are for the use of `add`, they’re not exposed in the
`wyag rm` command).

``` src src-python
def rm(repo, paths, delete=True, skip_missing=False):
  # Find and read the index
  index = index_read(repo)

  worktree = repo.worktree + os.sep

  # Make paths absolute
  abspaths = list()
  for path in paths:
    abspath = os.path.abspath(path)
    if abspath.startswith(worktree):
      abspaths.append(abspath)
    else:
      raise Exception("Cannot remove paths outside of worktree: {}".format(paths))

  kept_entries = list()
  remove = list()

  for e in index.entries:
    full_path = os.path.join(repo.worktree, e.name)

    if full_path in abspaths:
      remove.append(full_path)
      abspaths.remove(full_path)
    else:
      kept_entries.append(e) # Preserve entry

  if len(abspaths) > 0 and not skip_missing:
    raise Exception("Cannot remove paths not in the index: {}".format(abspaths))

  if delete:
    for path in remove:
      os.unlink(path)

  index.entries = kept_entries
  index_write(repo, index)
```

And we can now delete files with `wyag rm`.

### 9.3. The add command

Adding is just a bit more complex than removing, but nothing we don’t
already know. Staging a file to a three-steps operation:

1.  We begin by removing the existing index entry, if there’s one,
    without removing the file itself (this is why the `rm` function we
    just wrote has those optional arguments).
2.  We then hash the file into a glob object,
3.  create its entry,
4.  And of course, finally write the modified index back.

First, the interface. Nothing surprising, `wyag add PATH ...` where PATH
is one or more file(s) to stage. The bridge is as boring as can be.

``` src src-python
argsp = argsubparsers.add_parser("add", help = "Add files contents to the index.")
argsp.add_argument("path", nargs="+", help="Files to add")

def cmd_add(args):
  repo = repo_find()
  add(repo, args.path)
```

The main difference with `rm` is that `add` needs to create an index
entry. This isn’t hard: we just `stat()` the file and copy the metadata
in the index’s field (`stat()` returns those metadata the index stores:
creation/modification time, and so on)

``` src src-python
def add(repo, paths, delete=True, skip_missing=False):

  # First remove all paths from the index, if they exist.
  rm (repo, paths, delete=False, skip_missing=True)

  worktree = repo.worktree + os.sep

  # Convert the paths to pairs: (absolute, relative_to_worktree).
  # Also delete them from the index if they're present.
  clean_paths = list()
  for path in paths:
    abspath = os.path.abspath(path)
    if not (abspath.startswith(worktree) and os.path.isfile(abspath)):
      raise Exception("Not a file, or outside the worktree: {}".format(paths))
    relpath = os.path.relpath(abspath, repo.worktree)
    clean_paths.append((abspath,  relpath))

    # Find and read the index.  It was modified by rm.  (This isn't
    # optimal, good enough for wyag!)
    #
    # @FIXME, though: we could just
    # move the index through commands instead of reading and writing
    # it over again.
    index = index_read(repo)

    for (abspath, relpath) in clean_paths:
      with open(abspath, "rb") as fd:
        sha = object_hash(fd, b"blob", repo)

      stat = os.stat(abspath)

      ctime_s = int(stat.st_ctime)
      ctime_ns = stat.st_ctime_ns % 10**9
      mtime_s = int(stat.st_mtime)
      mtime_ns = stat.st_mtime_ns % 10**9

      entry = GitIndexEntry(ctime=(ctime_s, ctime_ns), mtime=(mtime_s, mtime_ns), dev=stat.st_dev, ino=stat.st_ino,
                            mode_type=0b1000, mode_perms=0o644, uid=stat.st_uid, gid=stat.st_gid,
                            fsize=stat.st_size, sha=sha, flag_assume_valid=False,
                            flag_stage=False, name=relpath)
      index.entries.append(entry)

    # Write the index back
    index_write(repo, index)
```

### 9.4. The commit command

Now that we have modified the index, so actually *staged changes*, we
only need to turn those changes into a commit. That’s what `commit`
does.

``` src src-python
argsp = argsubparsers.add_parser("commit", help="Record changes to the repository.")

argsp.add_argument("-m",
                   metavar="message",
                   dest="message",
                   help="Message to associate with this commit.")
```

To do so, we first need to convert the index into a tree object,
generate and store the corresponding commit object, and update the HEAD
branch to the new commit (remember: a branch is just a ref to a commit).

Before we get to the interesting details, we will need to read git’s
config to get the name of the user, which we’ll use as the author and
committer. We’ll use the same `configparser` library we’ve used to read
repo’s config.

``` src src-python
def gitconfig_read():
    xdg_config_home = os.environ["XDG_CONFIG_HOME"] if "XDG_CONFIG_HOME" in os.environ else "~/.config"
    configfiles = [
        os.path.expanduser(os.path.join(xdg_config_home, "git/config")),
        os.path.expanduser("~/.gitconfig")
    ]

    config = configparser.ConfigParser()
    config.read(configfiles)
    return config
```

And just a simple function to grab, and format, the user identity:

``` src src-python
def gitconfig_user_get(config):
    if "user" in config:
        if "name" in config["user"] and "email" in config["user"]:
            return "{} <{}>".format(config["user"]["name"], config["user"]["email"])
    return None
```

Now for the interesting part. We first need to build a tree from the
index. This isn’t hard, but notice that while the index is flat (it
stores full paths for the whole worktree), a tree is a recursive
structure: it lists files, or other trees. To “unflatten” the index into
a tree, we’re going to:

1.  Build a dictionary (hashmap) of directories. Keys are full paths
    from worktree root (like `assets/sprites/monsters/`), values are
    list of `GitIndexEntry` — files in the directory. At this point, our
    dictionary only contains *files*: directories are only its keys.
2.  Traverse this list, going bottom-up, that is, from the deepest
    directories up to root (depth doesn’t really matter: we just want to
    see each directory *before* its parent. To do that, we just sort
    them by *full* path length, from longest to shortest — parents are
    obviously always shorter). As an example, imagine we start at
    `assets/sprites/monsters/`
3.  At each directory, we build a tree with its contents, say
    `cacodemon.png`, `imp.png` and `baron-of-hell.png`.
4.  We write the new tree to the repository.
5.  We then add this tree to this directory’s parent. Meaning that at
    this point, `assets/sprites/` now contains our new tree object’s
    SHA-1 id under the name `monsters`.
6.  And we iterate over the next directory, let’s say
    `assets/sprites/keys` where we find `red.png`, `blue.png` and
    `yellow.png`, create a tree, store the tree, add the tree’s SHA-1
    under the name `keys` under `assets/sprites/`, and so on.

And since trees are recursive? So the last tree we’ll build, which is
necessarily the one for root (since its key’s length is 0), will
ultimately refer to all others, and thus will be only one we’ll need.
We’ll simply return its SHA-1, and be done.

Since this may seem a bit complex, let’s work this example in full
details — feel free to skip. At the beginning, the dictionary we built
from the index looks like this:

``` example
contents["assets/sprites/monsters"] =
  [ cacodemon.png : GitIndexEntry
  , imp.png : GitIndexEntry
  , baron-of-hell.png : GitIndexEntry ]
contents["assets/sprites/keys"] =
  [ red.png : GitIndexEntry
  , blue.png : GitIndexEntry
  , yellow.png : GitIndexEntry ]
contents["assets/sprites/"] =
  [ hero.png : GitIndexEntry ]
contents["assets/"] = [] # No files in here
contents[""] = # Root!
  [ README: GitIndexEntry ]
```

We iterate over it, by order of descending key length. The first key we
meet is the longest, so `assets/sprites/monsters`. We build a new tree
object from its contents, which associates the three file names
(`cacodemon.png`, `imp.png`, `baron-of-hell.png`) with their
corresponding blobs (A tree leaf stores *less* data than the index —
just path, mode and blob. So converting entries that way is easy)

Notice we don’t need to concern ourselves with storing the **contents**
of those files: `wyag add` did create the corresponding blobs as needed.
We need to store the *trees* we create to the object store, but we can
assume the blobs are there already.

Let’s say that our new tree hashes, made from the index entries that
lived directly in `assets/sprites/monsters`, hashes down to
`426f894781bc3c38f1d26f8fd2c7f38ab8d21763`. We **modify our dictionary**
to add that new tree object to the directory’s parent, like this, so
what remains to traverse now looks like this:

``` example
contents["assets/sprites/keys"] = # <- unmodified.
  [ red.png : GitIndexEntry
  , blue.png : GitIndexEntry
  , yellow.png : GitIndexEntry ]
contents["assets/sprites/"] =
  [ hero.png : GitIndexEntry
  , monsters : Tree 426f894781bc3c38f1d26f8fd2c7f38ab8d21763 ] <- look here
contents["assets/"] = [] # empty
contents[""] = # Root!
  [ README: GitIndexEntry ]
```

We do the same for the next longest key, `assets/sprites/keys`,
producing a tree of hash `b42788e087b1e94a0e69dcb7a4a243eaab802bb2`, so:

``` example
contents["assets/sprites/"] =
  [ hero.png : GitIndexEntry
  ,  monsters : Tree 426f894781bc3c38f1d26f8fd2c7f38ab8d21763
  , keys : Tree b42788e087b1e94a0e69dcb7a4a243eaab802bb2 ]
contents["assets/"] = [] # empty
contents[""] = # Root!
  [ README: GitIndexEntry ]
```

We then generate tree `6364113557ed681d775ccbd3c90895ed276956a2` from
assets/sprites, which now contains our two trees and `hero.png`.

``` example
contents["assets/"] = [
  sprites: Tree 6364113557ed681d775ccbd3c90895ed276956a2 ]
contents[""] = # Root!
  [ README: GitIndexEntry ]
```

Assets in turn becomes tree `4d35513cb6d2a816bc00505be926624440ebbddd`,
so:

``` example
contents[""] = # Root!
  [ README: GitIndexEntry,
    assets: 4d35513cb6d2a816bc00505be926624440ebbddd]
```

We make a tree from that last key (with the `README` blob and the
`assets` subtree), it hashes to
`9352e52ff58fa9bf5a750f090af64c09fa6a3d93`. That’s our return value: the
tree whose contents are the same as the index’s.

Here’s the actual function:

``` src src-python
def tree_from_index(repo, index):
    contents = dict()
    contents[""] = list()

    # Enumerate entries, and turn them into a dictionary where keys
    # are directories, and values are lists of directory contents.
    for entry in index.entries:
        dirname = os.path.dirname(entry.name)

        # We create all dictonary entries up to root ("").  We need
        # them *all*, because even if a directory holds no files it
        # will contain at least a tree.
        key = dirname
        while key != "":
            if not key in contents:
                contents[key] = list()
            key = os.path.dirname(key)

        # For now, simply store the entry in the list.
        contents[dirname].append(entry)

    # Get keys (= directories) and sort them by length, descending.
    # This means that we'll always encounter a given path before its
    # parent, which is all we need, since for each directory D we'll
    # need to modify its parent P to add D's tree.
    sorted_paths = sorted(contents.keys(), key=len, reverse=True)

    # This variable will store the current tree's SHA-1.  After we're
    # done iterating over our dict, it will contain the hash for the
    # root tree.
    sha = None

    # We ge through the sorted list of paths (dict keys)
    for path in sorted_paths:
        # Prepare a new, empty tree object
        tree = GitTree()

        # Add each entry to our new tree, in turn
        for entry in contents[path]:
            # An entry can be a normal GitIndexEntry read from the
            # index, or a tree we've created.
            if isinstance(entry, GitIndexEntry): # Regular entry (a file)

                # We transcode the mode: the entry stores it as integers,
                # we need an octal ASCII representation for the tree.
                leaf_mode = "{:02o}{:04o}".format(entry.mode_type, entry.mode_perms).encode("ascii")
                leaf = GitTreeLeaf(mode = leaf_mode, path=os.path.basename(entry.name), sha=entry.sha)
            else: # Tree.  We've stored it as a pair: (basename, SHA)
                leaf = GitTreeLeaf(mode = b"040000", path=entry[0], sha=entry[1])

            tree.items.append(leaf)

        # Write the new tree object to the store.
        sha = object_write(tree, repo)

        # Add the new tree hash to the current dictionary's parent, as
        # a pair (basename, SHA)
        parent = os.path.dirname(path)
        base = os.path.basename(path) # The name without the path, eg main.go for src/main.go
        contents[parent].append((base, sha))

    return sha
```

This was the hard part; I hope it’s clear enough. From this, creating
the commit object and updating HEAD will be way easier. Just remember
that what this function *does* is built and store as many tree objects
as needed to represent the index, and return the root tree’s SHA-1.

The function to create a commit object is simple enough, it just takes
some arguments: the hash of the tree, the hash of the parent commit, the
author’s identity (a string), the timestamp and timezone delta, and the
message:

``` src src-python
def commit_create(repo, tree, parent, author, timestamp, message):
    commit = GitCommit() # Create the new commit object.
    commit.kvlm[b"tree"] = tree.encode("ascii")
    if parent:
        commit.kvlm[b"parent"] = parent.encode("ascii")

    # Format timezone
    offset = int(timestamp.astimezone().utcoffset().total_seconds())
    hours = offset // 3600
    minutes = (offset % 3600) // 60
    tz = "{}{:02}{:02}".format("+" if offset > 0 else "-", hours, minutes)

    author = author + timestamp.strftime(" %s ") + tz

    commit.kvlm[b"author"] = author.encode("utf8")
    commit.kvlm[b"committer"] = author.encode("utf8")
    commit.kvlm[None] = message.encode("utf8")

    return object_write(commit, repo)
```

All what remains to write is `cmd_commit`, the bridge function to the
`wyag commit` command:

``` src src-python
def cmd_commit(args):
    repo = repo_find()
    index = index_read(repo)
    # Create trees, grab back SHA for the root tree.
    tree = tree_from_index(repo, index)

    # Create the commit object itself
    commit = commit_create(repo,
                           tree,
                           object_find(repo, "HEAD"),
                           gitconfig_user_get(gitconfig_read()),
                           datetime.now(),
                           args.message)

    # Update HEAD so our commit is now the tip of the active branch.
    active_branch = branch_get_active(repo)
    if active_branch: # If we're on a branch, we update refs/heads/BRANCH
        with open(repo_file(repo, os.path.join("refs/heads", active_branch)), "w") as fd:
            fd.write(commit + "\n")
    else: # Otherwise, we update HEAD itself.
        with open(repo_file(repo, "HEAD"), "w") as fd:
            fd.write("\n")
```

And we’re done\!

## 10\. Final words

### 10.1. Comments, feedback and issues

This page has no comment system :) I can be reached by e-mail at
<thibault@thb.lt>. I can also be found [on Mastodon as
@thblt@toad.social](https://toad.social/@thblt) and [on Twitter as
@ThbPlg](https://twitter.com/ThbPlg), and on IRC (sometimes) as `thblt`
on Libera.

The source for this article is hosted [on
Github](https://github.com/thblt/write-yourself-a-git). Issue reports
and pull requests are welcome, either directly on GitHub or through
e-mail if you prefer.

### 10.2. License

This article is distributed under the terms of the [Creative Commons
BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/). The
[program itself](./wyag.zip) is also licensed under the terms of the GNU
General Public License 3.0, or, at your option, any later version of the
same licence.

## Footnotes:

^([1](#fnr.1))

You may know that [collisions have been discovered in
SHA-1](https://shattered.io/). Git actually doesn’t use SHA-1 anymore:
it uses a [hardened
variant](https://github.com/git/git/blob/26e47e261e969491ad4e3b6c298450c061749c9e/Documentation/technical/hash-function-transition.txt#L34-L36)
which is not SHA, but which applies the same hash to every known input
but the two PDF files known to collide.

Author: [Thibault Polge](mailto:thibault@thb.lt)

Created: 2024-06-08 sam. 10:48

[Validate](https://validator.w3.org/check?uri=referer)