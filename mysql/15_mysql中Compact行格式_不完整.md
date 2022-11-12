> 参考来源：
>
> 康师傅：https://www.bilibili.com/video/BV1iq4y1u7vj?p=116
>
> 爱编程的大李子：https://blog.csdn.net/LXYDSF/article/details/125873790

ROW_FORMAT = Compact;

-  record_type：记录头信息的一项属性，表示记录的类型， 0 表示普通记录、 2 表示最小记 录、 3 表示最大记录、 1 表示目录项的记录，暂时还没用过，下面讲。
- next_record：记录头信息的一项属性，表示下一条地址相对于本条记录的地址偏移量，我们用箭头来表明下一条记录是谁。（用来保证数据逻辑上的连续）
- 各个列的值：这里只记录在 index_demo 表中的三个列，分别是 c1 、 c2 和 c3。
-  其他信息：除了上述 3 种信息以外的所有信息，包括其他隐藏列的值以及记录的额外信息。