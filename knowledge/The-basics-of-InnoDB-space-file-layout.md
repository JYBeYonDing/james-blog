---
title: The-basics-of-InnoDB-space-file-layout
categories: []
tags:
  - 转载
  - MySQL
halo:
  site: http://blog.jamesyoung94.top:8081
  name: 04d63a33-b808-473b-b28a-1e8a4f0b75d5
  publish: true
---
# The basics of InnoDB space file layout
#### by Jeremy Cole, [blog.jcole.us](https://blog.jcole.us/2013/01/03/the-basics-of-innodb-space-file-layout/) ▪ 2013年1月4日星期五 15:17

In [On learning InnoDB: A journey to the core](https://blog.jcole.us/2013/01/02/on-learning-innodb-a-journey-to-the-core/), I introduced the [innodb\_diagrams](http://github.com/jeremycole/innodb_diagrams) project to document the InnoDB internals, which provides the diagrams used in this post.

InnoDB’s data storage model uses “spaces”, often called “tablespaces” in the context of MySQL, and sometimes called “file spaces” in InnoDB itself. A space may consist of multiple actual files at the operating system level (e.g. ibdata1, ibdata2, etc.) but it is just a single logical file — multiple physical files are just treated as though they were physically concatenated together.


翻译：InnoDB的数据存储模型使用“空间”，在MySQL上下文中通常称为“表空间”，在InnoDB本身中有时称为“文件空间”。空间可以由操作系统级别的多个实际文件组成（例如ibdata1，ibdata2等），但它只是单个逻辑文件 - 多个物理文件只是被视为物理连接在一起。 


Each space in InnoDB is assigned a 32-bit integer space ID, which is used in many different places to refer to the space. InnoDB always has a “system space”, which is always assigned the space ID of 0. The system space is used for various special bookkeeping that InnoDB requires. Through MySQL, InnoDB currently only supports additional spaces in the form of “file per table” spaces, which create an .ibd file for each MySQL table. Internally, this .ibd file is actually a fully functional space which could contain multiple tables, but in the implementation with MySQL, they will only contain a single table.

翻译：InnoDB中的每个空间都被分配一个32位整数空间ID，该ID用于许多不同的位置引用该空间。 InnoDB始终拥有一个“系统空间”，该空间的空间ID为 0。系统空间用于InnoDB所需的各种特殊标记。通过MySQL，InnoDB目前仅支持以“每个表一个文件”空间的形式创建其他空间，每个MySQL表创建一个.ibd文件。在内部，此.ibd文件实际上是一个完全功能的空间，可以包含多个表，但在MySQL的实现中，它们只包含一个表。

Pages
-----

Each space is divided into pages, normally 16 KiB each (this can differ for two reasons: if the compile-time define UNIV\_PAGE\_SIZE is changed, or if InnoDB compression is used). Each page within a space is assigned a 32-bit integer page number, often called “offset”, which is actually just the page’s offset from the beginning of the space (not necessarily the file, for multi-file spaces). So, page 0 is located at file offset 0, page 1 at file offset 16384, and so on. (The astute may remember that InnoDB has a limit of 64TiB of data; this is actually a limit per space, and is due primarily to the page number being a 32-bit integer combined with the default page size: 232 x 16 KiB = 64 TiB.)


翻译：一个空间可以分为多个页，每页大小为16KiB（16*1024Byte）。空间中的每个页面都被分配了一个32位整数页码，通常称为“偏移量”。实际上，这只是页面相对于空间开头的偏移量（对于多文件空间，不一定是文件）。因此，页面0位于文件偏移量0处，页面1位于文件偏移量16384处，依此类推。（敏锐的人可能会记得InnoDB有64TiB数据的限制;这实际上是每个空间的限制，主要是由于页码是32位整数与默认页大小相结合：2^32 x 16 KiB = 64 TiB。）


A page is laid out as follows:

![Alt text](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/The-basics-of-InnoDB-space-file-layout/image.png)

Every page has a 38-byte FIL header and 8-byte FIL trailer (FIL is a shortened form of “file”). The header contains a field which is used to indicate the page type, which determines the structure of the rest of the page. The structure of the FIL header and trailer are:


翻译：每个页面都有一个38字节的FIL头和8字节的FIL尾（FIL是“文件”的缩写）。头包含一个字段，用于指示页面类型，该字段确定其余页面的结构。FIL头和尾的结构如下：


![Alt text](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/The-basics-of-InnoDB-space-file-layout/image-1.png)

The FIL header and trailer contain the following structures (not in order):


翻译：FIL头和尾包含以下结构（不按顺序）：


*   The page type is stored in the header. This is necessary in order to parse the rest of the page data. Pages are allocated for file space management, extent management, the transaction system, the data dictionary, undo logs, blobs, and of course indexes (table data).
*   The space ID is stored in the header.
*   The page number is stored in the header once the page has been initialized. Checking that the page number read from that field matches what it should be based on the offset into the file is helpful to indicate that reading is correct, and this field being initialized indicates that the page has been initialized.
*   A 32-bit checksum is stored in the header, and an older format (and broken) 32-bit checksum is stored in the trailer. The older checksum could be deprecated and that space reclaimed at some point.
*   Pointers to the logical previous and next page for this page type are stored in the header. This allows doubly-linked lists of pages to be built, and this is used for INDEX pages to link all pages at the same level, which allows for e.g. full index scans to be efficient. Many page types do not use these fields.
*   The 64-bit log sequence number (LSN) of the last modification of the page is stored in the header, and the low 32-bits of the same LSN are stored in the trailer.
*   A 64-bit “flush LSN” field is stored in the header, which is actually only populated for a single page in the entire system, page 0 of space 0. This stores the highest LSN flushed to any page in the entire system (all spaces). This field is a great candidate for re-use in the rest of the space.

Space files
-----------

A space file is just a concatenation of many (up to 2^32) pages. For more efficient management, pages are grouped into blocks of 1 MiB (64 contiguous pages with the default page size of 16 KiB), and called an “extent”. Many structures then refer only to extents to allocate pages within a space.

翻译：空间文件只是许多（最多2^32）页面的连接。为了更有效地管理，页面被分组为1 MiB的块（默认页面大小为16 KiB的64个连续页面），称为“extent”。然后，许多结构仅引用extent以在空间内分配页面。

InnoDB needs to do some bookkeeping to keep track of all of the pages, extents, and the space itself, so a space file has some mandatory super-structure:

翻译：InnoDB需要做一些簿记来跟踪所有页面，extent和空间本身，因此空间文件具有一些强制性的超级结构：

![Alt text](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/The-basics-of-InnoDB-space-file-layout/image-2.png)

The first page (page 0) in a space is always an FSP\_HDR or “file space header” page. The FSP\_HDR page contains (confusingly) an FSP header structure, which tracks things like the size of the space and lists of free, fragmented, and full extents. (A more detailed discussion of free space management is reserved for a future post.)


翻译：空间中的第一页（页面0）始终是FSP_HDR或“文件空间头”页面。 FSP_HDR页面包含（令人困惑的是）FSP头结构，该结构跟踪空间的大小以及空闲，碎片和完整extent的列表。 （有关空闲空间管理的更详细讨论将保留到未来的帖子。）


An FSP\_HDR page only has enough space internally to store bookkeeping information for 256 extents (or 16,384 pages, 256 MiB), so additional space must be reserved every 16,384 pages for bookkeeping information in the form of an XDES page. The structure of XDES and FSP\_HDR pages is identical, except that the FSP header structure is zeroed-out in XDES pages. These additional pages are allocated automatically as a space file grows.


翻译：  FSP_HDR页面内部仅有足够的空间来存储256个extent（或16,384个页面，256 MiB）的簿记信息，因此必须在每16,384个页面中保留额外的空间，以便以XDES页面的形式存储簿记信息。 XDES和FSP_HDR页面的结构相同，只是在XDES页面中将FSP头结构清零。这些额外的页面将随着空间文件的增长而自动分配。


The third page in each space (page 2) will be an INODE page, which is used to store lists related to file segments (groupings of extents plus an array of singly-allocated “fragment” pages). Each INODE page can store 85 INODE entries, and each index requires two INODE entries. (A more detailed discussion of INODE entries and file segments is reserved for a future post.)


翻译：每个空间中的第三页（页面2）将是INODE页面，用于存储与文件段相关的列表（extent的分组加上单独分配的“fragment”页面数组）。每个INODE页面可以存储85个INODE条目，每个索引需要两个INODE条目。 （有关INODE条目和文件段的更详细讨论将保留到未来的帖子。）


Alongside each FSP\_HDR or XDES page will also be an IBUF\_BITMAP page, which is used for bookkeeping information related to insert buffering, and is outside the scope of this post.


翻译：每个FSP_HDR或XDES页面旁边还将有一个IBUF_BITMAP页面，用于与插入缓冲区相关的簿记信息，超出了本帖子的范围。


The system space
----------------

The system space (space 0) is special in InnoDB, and contains quite a few pages allocated at fixed page numbers to store a wide range of information critical to InnoDB’s operation. Since the system space is a space like any other, it has the required FSP\_HDR, IBUF\_BITMAP, and INODE pages allocated as its first three pages. After that, it is a bit special:

翻译：InnoDB中的系统空间（空间0）是特殊的，并且包含分配给存储InnoDB操作所必需的各种关键信息的固定页面号的许多页面。由于系统空间与其他任何空间一样，因此它具有所需的FSP_HDR，IBUF_BITMAP和INODE页面，这些页面被分配为其前三个页面。之后，它有点特别：

![Alt text](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/The-basics-of-InnoDB-space-file-layout/image-3.png)

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


翻译：InnoDB提供了“每个表一个文件”的模式，该模式将为创建的每个MySQL表创建一个文件（如上所述，实际上是一个空间）。此功能的更好名称可能是“每个表一个空间”，而不是“每个表一个文件”。为每个表创建的.ibd文件具有典型的空间文件结构：


![Alt text](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/The-basics-of-InnoDB-space-file-layout/image-4.png)

Ignoring “fast index creation” which adds indexes at runtime, after the requisite 3 initial pages, the next pages allocated in the space will be the root pages of each index in the table, in the order they were defined in the table creation. Page 3 will be the root of the clustered index, Page 4 will be the root of the first secondary key, etc.


翻译：忽略在运行时添加索引的“快速索引创建”，在必要的3个初始页面之后，空间中分配的下一个页面将是表中每个索引的根页面，按照它们在表创建中定义的顺序。页面3将是聚集索引的根，页面4将是第一个辅助键的根，依此类推。


Since most of InnoDB’s bookkeeping structures are stored in the system space, most pages allocated in a per-table space will be of type INDEX and store table data.


翻译：由于InnoDB的大多数簿记结构都存储在系统空间中，因此在每个表空间中分配的大多数页面都是INDEX类型并存储表数据。


What’s next?
------------

Next we’ll look at free space management within InnoDB: extent descriptors, file segments (inodes), and lists.


翻译：接下来，我们将查看InnoDB中的空闲空间管理：extent描述符，文件段（inode）和列表。


---

Clipped on 2023年9月3日星期日 14:51
