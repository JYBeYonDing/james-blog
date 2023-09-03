---
title: The basics of InnoDB space file layout
categories: []
tags:
  - 转载
  - MySQL
halo:
  site: http://blog.jamesyoung94.top:8081
  name: a3d9c806-91f1-4648-af97-d8bfb19dfe60
  publish: true
---
# The basics of InnoDB space file layout
#### by Jeremy Cole, [blog.jcole.us](https://blog.jcole.us/2013/01/03/the-basics-of-innodb-space-file-layout/) ▪ 2013年1月4日星期五 15:17

In [On learning InnoDB: A journey to the core](https://blog.jcole.us/2013/01/02/on-learning-innodb-a-journey-to-the-core/), I introduced the [innodb\_diagrams](http://github.com/jeremycole/innodb_diagrams) project to document the InnoDB internals, which provides the diagrams used in this post.

InnoDB’s data storage model uses “spaces”, often called “tablespaces” in the context of MySQL, and sometimes called “file spaces” in InnoDB itself. A space may consist of multiple actual files at the operating system level (e.g. ibdata1, ibdata2, etc.) but it is just a single logical file — multiple physical files are just treated as though they were physically concatenated together.

Each space in InnoDB is assigned a 32-bit integer space ID, which is used in many different places to refer to the space. InnoDB always has a “system space”, which is always assigned the space ID of 0. The system space is used for various special bookkeeping that InnoDB requires. Through MySQL, InnoDB currently only supports additional spaces in the form of “file per table” spaces, which create an .ibd file for each MySQL table. Internally, this .ibd file is actually a fully functional space which could contain multiple tables, but in the implementation with MySQL, they will only contain a single table.

Pages
-----

Each space is divided into pages, normally 16 KiB each (this can differ for two reasons: if the compile-time define UNIV\_PAGE\_SIZE is changed, or if InnoDB compression is used). Each page within a space is assigned a 32-bit integer page number, often called “offset”, which is actually just the page’s offset from the beginning of the space (not necessarily the file, for multi-file spaces). So, page 0 is located at file offset 0, page 1 at file offset 16384, and so on. (The astute may remember that InnoDB has a limit of 64TiB of data; this is actually a limit per space, and is due primarily to the page number being a 32-bit integer combined with the default page size: 232 x 16 KiB = 64 TiB.)

A page is laid out as follows:
![Alt text](<https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/The basics of InnoDB space file layout/image.png>)

Every page has a 38-byte FIL header and 8-byte FIL trailer (FIL is a shortened form of “file”). The header contains a field which is used to indicate the page type, which determines the structure of the rest of the page. The structure of the FIL header and trailer are:
![Alt text](<https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/The basics of InnoDB space file layout/image-1.png>)

The FIL header and trailer contain the following structures (not in order):

*   The page type is stored in the header. This is necessary in order to parse the rest of the page data. Pages are allocated for file space management, extent management, the transaction system, the data dictionary, undo logs, blobs, and of course indexes (table data).
*   The space ID is stored in the header.
*   The page number is stored in the header once the page has been initialized. Checking that the page number read from that field matches what it should be based on the offset into the file is helpful to indicate that reading is correct, and this field being initialized indicates that the page has been initialized.
*   A 32-bit checksum is stored in the header, and an older format (and broken) 32-bit checksum is stored in the trailer. The older checksum could be deprecated and that space reclaimed at some point.
*   Pointers to the logical previous and next page for this page type are stored in the header. This allows doubly-linked lists of pages to be built, and this is used for INDEX pages to link all pages at the same level, which allows for e.g. full index scans to be efficient. Many page types do not use these fields.
*   The 64-bit log sequence number (LSN) of the last modification of the page is stored in the header, and the low 32-bits of the same LSN are stored in the trailer.
*   A 64-bit “flush LSN” field is stored in the header, which is actually only populated for a single page in the entire system, page 0 of space 0. This stores the highest LSN flushed to any page in the entire system (all spaces). This field is a great candidate for re-use in the rest of the space.

Space files
-----------

A space file is just a concatenation of many (up to 232) pages. For more efficient management, pages are grouped into blocks of 1 MiB (64 contiguous pages with the default page size of 16 KiB), and called an “extent”. Many structures then refer only to extents to allocate pages within a space.

InnoDB needs to do some bookkeeping to keep track of all of the pages, extents, and the space itself, so a space file has some mandatory super-structure:
![Alt text](<https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/The basics of InnoDB space file layout/image-2.png>)

The first page (page 0) in a space is always an FSP\_HDR or “file space header” page. The FSP\_HDR page contains (confusingly) an FSP header structure, which tracks things like the size of the space and lists of free, fragmented, and full extents. (A more detailed discussion of free space management is reserved for a future post.)

An FSP\_HDR page only has enough space internally to store bookkeeping information for 256 extents (or 16,384 pages, 256 MiB), so additional space must be reserved every 16,384 pages for bookkeeping information in the form of an XDES page. The structure of XDES and FSP\_HDR pages is identical, except that the FSP header structure is zeroed-out in XDES pages. These additional pages are allocated automatically as a space file grows.

The third page in each space (page 2) will be an INODE page, which is used to store lists related to file segments (groupings of extents plus an array of singly-allocated “fragment” pages). Each INODE page can store 85 INODE entries, and each index requires two INODE entries. (A more detailed discussion of INODE entries and file segments is reserved for a future post.)

Alongside each FSP\_HDR or XDES page will also be an IBUF\_BITMAP page, which is used for bookkeeping information related to insert buffering, and is outside the scope of this post.

The system space
----------------

The system space (space 0) is special in InnoDB, and contains quite a few pages allocated at fixed page numbers to store a wide range of information critical to InnoDB’s operation. Since the system space is a space like any other, it has the required FSP\_HDR, IBUF\_BITMAP, and INODE pages allocated as its first three pages. After that, it is a bit special:
![Alt text](<https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/The basics of InnoDB space file layout/image-3.png>)

The following pages are allocated:

*   Page 3, type SYS: Headers and bookkeeping information related to insert buffering.
*   Page 4, type INDEX: The root page of the index structure used for insert buffering.
*   Page 5, type TRX\_SYS: Information related to the operation of InnoDB’s transaction system, such as the latest transaction ID, MySQL binary log information, and the location of the double write buffer extents.
*   Page 6, type SYS: The first rollback segment page. Additional pages (or whole extents) are allocated as needed to store rollback segment data.
*   Page 7, type SYS: Headers related to the data dictionary, containing root page numbers for the indexes that make up the data dictionary. This information is required to be able to find any other indexes (tables), as their root page numbers are stored in the data dictionary itself.
*   Pages 64-127: The first block of 64 pages (an extent) in the double write buffer. The double write buffer is used as part of InnoDB’s recovery mechanism.
*   Pages 128-191: The second block of the double write buffer.

All other pages are allocated on an as-needed basis to indexes, rollback segments, undo logs, etc.

Per-table space files
---------------------

InnoDB offers a “file per table” mode, which will create a file (which as explained above is actually a space) for each MySQL table created. A better name for this feature may be “space per table” rather than “file per table”. The .ibd file created for each table has the typical space file structure:
![Alt text](<https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/The basics of InnoDB space file layout/image-4.png>)

Ignoring “fast index creation” which adds indexes at runtime, after the requisite 3 initial pages, the next pages allocated in the space will be the root pages of each index in the table, in the order they were defined in the table creation. Page 3 will be the root of the clustered index, Page 4 will be the root of the first secondary key, etc.

Since most of InnoDB’s bookkeeping structures are stored in the system space, most pages allocated in a per-table space will be of type INDEX and store table data.

What’s next?
------------

Next we’ll look at free space management within InnoDB: extent descriptors, file segments (inodes), and lists.

---

Clipped on 2023年9月3日星期日 14:30
