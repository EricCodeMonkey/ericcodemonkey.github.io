## What is Git
Git is a stupid content tracker. That is probably the best description of it- don't think of it in a 'like (insert favorite SCM system), but...' context, but more like a really interesting file system.

Git tracks content - files and directories. At its heart, it is a collection of simple tools that implement a tree history storage and directory content management system. It is simply used as an SCM, not really designed as one.

`"In many ways, you can just see git as a filesystem - it's content-addressable, and it has a notion of versioning, but I designed it coming at the problem from the viewpoint of a filesystem person (hey, kernels is what I do), and I have absolutely zero interest in creating a traditional SCM system." - Linus`

When most SCMs store a new version of a project, they store the code delta or diff. When Git stores a new version of a project, it stores a new tree - a bunch of blobs of content and a collection of pointers that can be expanded back out into a full directory of files and subdirectories. If you want a diff between two versions, it doesn't add up all the deltas, it simply looks at the two trees and runs a new diff on them. 

This is what fundamentally allows the system to be easily distributed  - it doesn't have issues figuring out how to apply a complex series of deltas, it simply transfers all the directories and content that one user has and another does not have but is requesting. It is efficient - it only stores identical files and directories once and it can compress and transfer its content using delta-compressed pack files - but in concept, it is a very simple beast. Git is at its heart very stupid and simple.



## A Toolkit Design
Git is not really a single binary, but a collection of dozens of small specialized programs which is sometimes annoying to people trying to learn Git, but is pretty cool when you want to do anything non-standard with it. Git is less a program and more a toolkit that can be combined and chained to do new and interesting things.

The tools can be more or less divided into two major camps, often referred to as porcelain and plumbing. The plumbing is not really meant to be used by people on the command line, but rather to do simple things flexibly and are combined by programs and scripts into porcelain programs. 

## Git Object Types
Git objects are the actual data of Git, the main thing that the repository is made up of. There are four main object types in Git. The first three are the most important to really understand the main functions of Git.

All of these types of objects are stored in the Git Object Database, which is kept in the Git Directory. Each object is compressed (with Zlib) and referenced by the SHA-1 value of its contents plus a small header.

### The Blob
In Git, the contents of files are stored as blobs. It is important to note that it is the contents that are stored, not the files. The names and modes of the files are not stored with the blob, just the contents.

That means that if you have two files anywhere in your project that are the same, even if they have different names, Git will only store the blob once. This also means that during repository transfers, such as clones or fetches, Git will only transfer the blob once, then expand it out into multiple files upon checkout.

![Git-Blob](ericcodemonkey/ericcodemonkey.github.io/docs/assets/images/Git-Blob.png)

### The Tree
Directories in Git basically correspond to trees.

![Git-Tree1](ericcodemonkey/ericcodemonkey.github.io/docs/assets/images/Git-Tree1.png)

A tree is a simple list of trees and blobs that the tree contains, along with the names and modes of those trees and blobs. The contents section of a tree object consists of a very simple text file that lists the mode, type, name, and SHA of each entry.
![Git-Blob](ericcodemonkey/ericcodemonkey.github.io/docs/assets/images/Git-Tree2.png)



### The Commit
So, now that we can store arbitrary trees of content in Git, where does the 'history' part of  'tree history storage system' come in? The answer is the commit object.

![Git-Blob](ericcodemonkey/ericcodemonkey.github.io/docs/assets/images/Git-Commit1.png)



The commit is very simple, much like the tree. It simply points to a tree and keeps an author, committer, message and any parent commits that directly preceded it.

![Git-Blob](ericcodemonkey/ericcodemonkey.github.io/docs/assets/images/Git-Commit2.png)

Since it was my first commit, there are no parents. If I commit a second time, the commit object will look more like this:

![Git-Blob](ericcodemonkey/ericcodemonkey.github.io/docs/assets/images/Git-Commit3.png)




### The Tag
The final type of object you will find in a Git database is the tag. This is an object that provides a permanent shorthand name for a particular commit. It contains an object, type, tag, tagger, and a message. Normally the type is commit and the object is the SHA-1 of the commit you're tagging. The tag can also be GPG signed, providing cryptographic integrity to a release or version.

![Git-Blob](ericcodemonkey/ericcodemonkey.github.io/docs/assets/images/Git-Tag1.png)



The Git Data Model
In computer science speak, the Git object data is a directed acyclic graph. That is, starting at any commit you can traverse its parents in one direction and there is no chain that begins and ends with the same object.

All commit objects point to a tree and optionally to previous commits. All trees point to one or many blobs and/or trees. Given this simple model, we can store and retrieve vast histories of complex trees of arbitrarily changing content quickly and efficiently.
