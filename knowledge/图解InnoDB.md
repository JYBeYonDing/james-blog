# 图解InnoDB

> 参考：https://blog.jcole.us/innodb/

Space File Overview：
![img](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/The-basics-of-InnoDB-space-file-layout/image-2.png)
Basic Page Overview，每一页page：
![img](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/The-basics-of-InnoDB-space-file-layout/image.png)
16KiB*64=1MiB

即：64个page组成一个数据块（extent）



FIL Header/Trailer：
![Alt text](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/The-basics-of-InnoDB-space-file-layout/image-1.png)

ibdata1 File Overview，InnoDB中的系统空间：
![](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/The-basics-of-InnoDB-space-file-layout/image-3.png)
所有其他页面都会根据需要分配给索引、回滚段、undo logs等。

IBD File Overview，其他的每个表一个空间
![img](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/The-basics-of-InnoDB-space-file-layout/image-4.png)
FSP_HDR/XDES Overview，0~16KiB: 表空间头，也是一页(Page) 16*1024Byte：
![img](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/Page-management-in-InnoDB-space-files/image.png)

XDES Entry：Extent descriptors(数据库描述符)：
![](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/Page-management-in-InnoDB-space-files/image-1.png)
base node节点：
![](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/Page-management-in-InnoDB-space-files/image-2.png)
普通节点：
![](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/Page-management-in-InnoDB-space-files/image-3.png)
FSP Header：space中的表空间头中的信息，表述的是space信息
![](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/Page-management-in-InnoDB-space-files/image-4.png)
* FREE\_FRAG（部分page已经使用）
* FULL\_FRAG（所有page都已经使用）
* FREE（所有page都没有使用）

INODE page:
![](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/Page-management-in-InnoDB-space-files/image-5.png)
对于16KiB页面，每个条目为192个字节，每个INODE页面包含85（16*1024/192）个文件段INODE条目。
文件段INODE条目:
![](https://cdn.jsdelivr.net/gh/JYBeYonDing/james-blog/knowledge/Page-management-in-InnoDB-space-files/image-6.png)

