---
title: Page-management-in-InnoDB-space-files
categories: []
tags:
  - MySQL
  - 转载
halo:
  site: http://blog.jamesyoung94.top:8081
  name: b0f76ad3-75e6-4e44-b62c-4180097c0a00
  publish: true
---
# Page management in InnoDB space files

#### by Jeremy Cole, [blog.jcole.us](https://blog.jcole.us/2013/01/04/page-management-in-innodb-space-files/) ▪ 2013年1月5日星期六 0:41

In [On learning InnoDB: A journey to the core](https://blog.jcole.us/2013/01/02/on-learning-innodb-a-journey-to-the-core/), I introduced the [innodb\_diagrams](http://github.com/jeremycole/innodb_diagrams) project to document the InnoDB internals, which provides the diagrams used in this post. (Note that each image below is linked to a higher resolution version of the same image.)

The basic structure of the space and each page was described in [The basics of InnoDB space file layout](https://blog.jcole.us/2013/01/03/the-basics-of-innodb-space-file-layout/), so now we’ll expand on that to describe InnoDB’s structures related to management of pages and extents, and “free space” management, and how it tracks pages allocated to the many different purposes for which pages are used.

Extents and extent descriptors
------------------------------

As described previously, InnoDB pages are normally 16 KiB, and are grouped into 1 MiB blocks of 64 contiguous pages, which is called an “extent”. InnoDB allocates FSP\_HDR and XDES pages at fixed locations within the space, to keep track of which extents are in use, and which pages within each extent are in use. These pages have a pretty simple structure:

翻译：如前所述，InnoDB页面（Page）通常为16KiB，并被分组为64个连续页面的1MiB块，这被称为“数据块（extent）”。InnoDB在空间内的固定位置分配FSP_HDR和XDES页面，以跟踪哪些数据块正在使用，以及每个数据块中的哪些页面正在使用。这些页面的结构非常简单：

![Alt text](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/Page-management-in-InnoDB-space-files/image.png)

They contain the usual FIL header and trailer, an FSP header (discussed a bit later), and 256 “extent descriptors” or just “descriptors”. They also contain a sizable amount of unused space.

翻译：它们包含通常的FIL标头和尾部、FSP标头（稍后讨论）和256个“数据块描述符”或就是“描述符”。它们还包含大量未使用的空间。

Extent descriptors have the following structure:

翻译：数据块描述符具有以下结构：

![Alt text](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/Page-management-in-InnoDB-space-files/image-1.png)

The purpose of the various fields in an extent descriptor are:

翻译：数据块描述符中各个字段的用途是：

* File Segment ID: The ID of the file segment to which the extent belongs, if it belongs to a file segment.
* List node for XDES list: Pointers to previous and next extents in a doubly-linked extent descriptor list.
* State: The current state of the extent, for which only four values are currently defined: FREE, FREE\_FRAG, and FULL\_FRAG, meaning this extent belongs to the space’s list with the same name; and FSEG, meaning this extent belongs to the file segment with the ID stored in the File Segment ID field. (More on these lists below.) 数据块的当前状态，当前仅为其定义了四个值：FREE、FREE_FRAG和FULL_FRAG，这意味着此数据块属于具有相同名称的空间列表；以及FSEG，这意味着该区段属于具有存储在文件段ID字段中的ID的文件段。（下面列出了更多信息。）
* Page State Bitmap（每一页的状态）: A bitmap of 2 bits per page in the extent (64 x 2 = 128 bits, or 16 bytes). The first bit indicates whether the page is free. The second bit is reserved to indicate whether the page is clean (has no un-flushed data), but this bit is currently unused and is always set to 1.

Other structures that reference an extent do so using a combination of the page number of the FSP\_HDR or XDES page in which the extent’s descriptor resides, and the byte offset within that page of the descriptor entry itself. For example, the extent referenced by “page 0 offset 150” is the first extent in the space, accounting for pages 0-63, while “page 16384 offset 270” accounts for pages 16576-16639.

List base nodes and list nodes
------------------------------

Lists (or “free lists” as InnoDB calls them) are a fairly generic structure that allows linking multiple related structures together. It consists of two complementary structures, which form a nicely featured on-disk doubly-linked list. The “list base node” has the following structure:

![Alt text](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/Page-management-in-InnoDB-space-files/image-2.png)

The base node is stored only once in some high level structure (such as the FSP header). It contains the length of the list, and pointers to the first and last list node in the list. The actual “list nodes” look very similar:

翻译：基节点仅在某些高级结构（例如FSP标头）中存储一次。它包含列表的长度以及指向列表中第一个和最后一个列表节点的指针。普通的“列表节点”看起来非常相似：

![Alt text](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/Page-management-in-InnoDB-space-files/image-3.png)

Instead of storing first and last pointers, of course, the list node stores previous and next pointers.

翻译：普通列表节点不是存储第一个和最后一个指针，而是存储前一个和后一个指针。

All pointers consist of a page number (which is required to be within the same space), and a byte offset within that page where the list node can be found. All pointers point to the beginning (that is, N+0) of the list node, not necessarily of the _structure_ being linked together. For example, when extent descriptor entries are linked in a list, since the list node is at offset 8 within the XDES entry structure, the code reading a list entry must “know” that the descriptor structure starts 8 bytes _before_ the offset of the list node, and read the structure from there. (It would’ve perhaps been a good idea to ensure that the list node is always first in any structure, but that is not the case.)

翻译：所有指针都由页码（必须在同一空间内）和列表节点可以找到的该页内的字节偏移量组成。所有指针都指向列表节点的开始（即N + 0），而不一定是链接在一起的结构。例如，当数据块描述符条目在列表中链接在一起时，由于列表节点在XDES条目结构的偏移量8处，因此读取列表条目的代码必须“知道”描述符结构从列表节点的偏移量之前8个字节开始，并从那里读取结构。（确保列表节点始终位于任何结构的第一位可能是个好主意，但事实并非如此。）

The file space header and extent lists
--------------------------------------

Aside from storing the extent descriptor entries themselves, the FSP\_HDR page (which is always page 0 in a space) also stores the FSP header, which contains a lot of lists, so couldn’t easily be described earlier. The structure of the FSP header is as follows:

![Alt text](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/Page-management-in-InnoDB-space-files/image-4.png)

The purposes of the non-list related fields in an FSP header are (not in order):

* Space ID: The space ID of the current space.
* Highest page number in the space (size): The “size” is the highest valid page number, and is incremented when the file is grown. However, not all of these pages are initialized (some may be zero-filled), as extending a space is a multi-step process.
* Highest page number initialized (free limit): The “free limit” is the highest page number for which the FIL header has been initialized, storing the page number in the page itself, amongst other things. The free limit will always be less than or equal to the size.
* Flags: Storage of flags related to the space.
* Next Unused Segment ID: The file segment ID that will be used for the next allocated file segment. (This is essentially an auto-increment integer.)
* Number of pages used in the FREE\_FRAG list: This is stored as an optimization to be able to quickly calculate the number of free pages in the FREE\_FRAG list, without iterating through all extents in the list and summing the free pages available in each. （用来快速计算FREE\_FRAG list 中的剩余页数）

List base nodes for the following extent descriptor lists are also stored in the FSP header:

存放一些列表的base节点信息

* FREE\_FRAG（部分page已经使用）: Extents with free pages remaining that are allocated to be used in “fragments”, having individual pages allocated to different purposes rather than allocating the entire extent. For example, every extent with an FSP\_HDR or XDES page will be placed on the FREE\_FRAG list so that the remaining free pages in the extent can be allocated for other uses.
* FULL\_FRAG（所有page都已经使用）: Exactly like FREE\_FRAG but for extents with no free pages remaining. Extents are moved from FREE\_FRAG to FULL\_FRAG when they become full, and moved back to FREE\_FRAG if a page is released so that they are no longer full.
* FREE（所有page都没有使用）: Extents that are completely unused and available to be allocated in whole to some purpose. A FREE extent could be allocated to a file segment (and placed on the appropriate INODE list), or moved to the FREE\_FRAG list for individual page use.

File segments and inodes
------------------------

File segments and inodes are perhaps where InnoDB terminology and documentation are murkiest. InnoDB overloads the term “inode”, which is in [common use in filesystems](http://en.wikipedia.org/wiki/Inode), and uses it for both INODE _entries_ (a single small structure) and INODE _pages_ (a page type holding many INODE entries). Ignoring the naming confusion for a moment, an INODE entry in InnoDB merely describes a file segment (frequently called an FSEG), and will be referred to as a “file segment INODE” from now on. The INODE pages that contain them have the following structure:

InnoDB中的INODE条目仅描述一个文件段（通常称为FSEG），从现在起将被称为“文件段INODE”。包含它们的INODE页面具有以下结构：

![Alt text](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/Page-management-in-InnoDB-space-files/image-5.png)

Each INODE page contains 85 file segment INODE entries (for a 16 KiB page), each of which are 192 bytes. In addition, they contain a list node which is used in the following lists of INODE pages in the FSP\_HDR‘s FSP header structure as described above:

对于16KiB页面，每个条目为192个字节，每个INODE页面包含85（16*1024/192）个文件段INODE条目。

* FREE\_INODES: A list of INODE pages which have at least one free file segment INODE entry.
* FULL\_INODES: A list of INODE pages which have zero free file segment INODE entries. When using “file per table” spaces, this list in each per-table space will be empty unless the table has more than 42 indexes, as each index consumes exactly two file segment INODE entries.

A file segment INODE entry has the following structure:

文件段INODE条目具有以下结构：

![Alt text](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/Page-management-in-InnoDB-space-files/image-6.png)

The purpose of each of the non-list fields in each INODE entry are:

* File Segment ID: The ID of the file segment (FSEG) described by this file segment INODE entry. If the ID is 0, the entry is unused.
* Magic Number: The value 97937874 is stored as a marker that this file segment INODE entry has been properly initialized.
* Number of used pages in the NOT\_FULL list: Exactly like the space’s FREE\_FRAG list (in the FSP header), this field stores the number of pages used in the NOT\_FULL list as an optimization to be able to quickly calculate the number of free pages in the list without iterating through all extents in the list.
* Fragment Array: An array of 32 page numbers of pages allocated individually from extents in the space’s FREE\_FRAG or FULL\_FRAG list of “fragment” extents. Once this array becomes full, only full extents can be allocated to the file segment.

As a table grows it will allocate individual pages in each file segment until the fragment array becomes full, and then switch to allocating 1 extent at a time, and eventually to allocating 4 extents at a time.

List base nodes of extent descriptors are also present in each file segment INODE entry:

* FREE: Extents that are completely unused and are allocated to this file segment.
* NOT\_FULL: Extents with at least one used page allocated to this file segment. When the last free page is used, the extent is moved to the FULL list.
* FULL: Extents with no free pages allocated to this file segment. If a page becomes free, the extent is moved to the NOT\_FULL list.

If the last used page is freed from an extent in the NOT\_FULL list, the extent could be moved to the file segment’s FREE list, but in practice is actually moved directly back to the space’s FREE list instead.

How indexes use file segments
-----------------------------

Although INDEX pages haven’t been described yet, one small aspect can be looked at now. The root page of each index’s FSEG header contains pointers to the file segment INODE entries which describe the file segments used by the index. Each index uses one file segment for leaf pages and one for non-leaf (internal) pages. This information is stored in the FSEG header structure (in the INDEX page):

虽然INDEX页面还没有被描述，但现在可以看一个小方面。每个索引的FSEG标头的根页包含指向文件段INODE条目的指针，这些条目描述了索引使用的文件段。每个索引对叶页使用一个文件段，对非叶页（内部）使用一个。此信息存储在FSEG标头结构中（在INDEX页面中）：

![Alt text](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/Page-management-in-InnoDB-space-files/image-7.png)

The space IDs present are somewhat superfluous — they will always be the same as the current space. The page number and offset point to a file segment INODE entry in an INODE page. Both file segments will always be present, even though they may be completely empty.

For example in a newly created table, the only page that exists will be the root page, which is also a leaf page, but is present in the “internal” file segment (so that it doesn’t have to be moved later). The “leaf” file segment INODE lists and fragment array will all be empty. The “internal” file segment INODE lists will all be empty, and the single root page will be in the fragment array.

Tying it all together
---------------------

The following diagram attempts to illustrate the entire multi-level structure for an index:

![Alt text](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/Page-management-in-InnoDB-space-files/image-9.png)
The index root page points to two inodes (file segments), each of which have a fragment array (pointing to up to 32 individual pages from a fragment list), as well as several lists of whole extents, which are linked together using list pointers in the extent descriptors. The extent descriptors are used both to reference an extent as well as to keep track of free pages within an extent. Easy!

索引根页面指向两个索引节点（文件段），每个索引节点都有一个片段数组（指向片段列表中最多32个单独的页面），以及几个完整扩展数据块的列表，这些列表使用扩展数据块描述符中的列表指针链接在一起。数据块描述符既用于引用区段，也用于跟踪区段内的空闲页面。

What’s next?
-------------

Next we’ll look at the structure of one of the most important page types from a user’s perspective, INDEX pages. We’ll then look at how InnoDB structures its indexes at a high level.

---

Clipped on 2023年9月3日星期日 16:59
