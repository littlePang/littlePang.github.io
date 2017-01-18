 ---
 layout: post
 title: mysql InnoDb Clustered Index
 category: 技术
 tags: mysql
 keywords:
 description:
 ---

 # Clustered Index 是什么
 Clustered Index(聚集索引)是每一个使用InnoDB引擎的表为所有行数据所持有的一个索引,
 当通过 聚集索引来查询某一行数据是非常快的,因为通过聚集索引可以直接找到一行的所有数据,聚集索引的存储结构,相比于将数据和索引分别存储在不同的结构中(例如 MyISAM引擎使用不同的文件来存储数据集和索引记录),能够更加节省磁盘I/O.

 # InnoDB中聚集索引的选取

 * 如果在创建表时,定义了主键索引(primary key),则InnoDB直接使用它作为聚集索引

 * 如果没有定义主键索引, 则mysql会使用第一个所有列都被定义为 not null 的 唯一键索引(unique key)作为聚集所有.

 * 如果记没有定义主键索引,也没有合适的唯一键索引,则InnoDB内部会为表的每一行记录生成一个ID,并且使用它来生成聚集索引,这个ID是一个长度为6个字段的单调递增的域,在每一行插入是生成该行的ID.

 # 二级索引和聚集索引的关联关系
 在InnoDB中,除了聚集索引外的所有索引,都称为Secondary Indexs(二级索引),每一个二级索引除了包含索引指定的列之外,还包含聚集索引所使用的列.通过二级索引查找一行记录时,InnoDB先在二级索引中找到聚集索引所对应的列数据,再通过这些数据去聚集索引中获取该行的全部数据.

 因此,如果聚集索引锁使用的列非常大的话,二级索引就会使用更多的空间,所以聚集索引的key越小越好.

 # 参考

 [http://dev.mysql.com/doc/refman/5.6/en/innodb-index-types.html](http://dev.mysql.com/doc/refman/5.6/en/innodb-index-types.html)
