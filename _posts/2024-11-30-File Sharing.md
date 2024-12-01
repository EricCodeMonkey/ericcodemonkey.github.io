 Today, when I read the book 《Advanced Programming in Unix Environment》Chatper 3 File/IO, I came across the below statement in the section "File Sharing":

    "Each process that opens the file gets its own file table entry, but only a single v-node table entry is required for a given file. One reason each process gets its own file table entry is so that each process has its own current offset for the file"

I need some clarification about this statement. Does it mean even when two processes open the same file, the kernel will create two entries in the file table for those two processes respectively. With this question in mind, I did some research and this post is a summary of it.

Key Concepts in the Kernel's File System

1. File Descriptor Table(Per process)
    Each process has its own file descriptor table, which maps file descriptors (e.g., 0,1,2...) to entries in the kernel's system-wide file table.
    Example: If a process opens a file, the file descriptor(e.g., fd=3) is mapped to an entry in the kernel file table.

2. File Table (System-wide)
    The kernel maintains a file table that stores information about all open files in the stem
    Each entry in this table includes details like:
                The file offset(current read/write position in the file)
                File access mode(read, write, or both)
                pointer to the v-node table entry for the file.

3. V-Node table (System-wide)
    The v-node table stores metadata about files, such as:
        File type (regular file, directory, etc.).
        File permissions and attributes.
        Pointers to disk blocks or inodes for accessing the file's content.
    
    Only one v-node entry exists for each unique file, regardless of how many processes open it.

What Happens When Two Processes Open the Same File?


1. Two File Table Entries:
    
    When two processes open the same file, the kernel creates two separate entries in the file table(one for each process).

    Each entry contains:
        Independent file offsets (since each process might read/write different parts of the file).
        The access mode for the specific open operation(read-only, write-only, etc.).
        A pointer to the same v-node entry for the file.

2. Single V-Node Entry:
    
    Both file table entries for the two processes point to the same v-node entry because the file being opened is the same.

    The v-node contains file-specific metadata and a reference to the file's inode on disk.


Why Two File Table Entries?


Each Process can manipulate the file independently:
    Different file offsets: Process A might be reading from offset 100, while process B is writing at offset 200.
    Different access modes: Process A might have opened the file in read-only mode, while process B opened it in write-only mode.

The Kernel needs separate file table entries to maintain this independent state for each process.

Summary of the Structure

When two processes open the same file:

    1. File Descriptor Table (Per process):
        Each process gets its own file descriptor pointing to a unique file table entry.
    
    2. File Table(System-wide)
        Two separate entries are created in the file table(one for each process), allowing independent tracking of file offsets and access modes.

    3. V-Node Table(System-Wide)
        Only a single v-node table entry is created for the file, as it represents the underlying file on the disk.


Visualization


Process       |          File Descriptor Table        |      File Table Entry             |        V-Node Entry

Process A   |  fd=3  -> File Table Entry 1       |   Offset: 100, Mode: Read  |  Points to V-Node for the file

Process B   | fd=4 -> File Table Entry 2         |  Offset: 200, Mode: Write  |  Points to V-Node for the file
