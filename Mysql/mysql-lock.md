## InnoDB引擎的行锁是怎么实现的?

InnoDB行锁是通过给索引上的索引项加锁来实现的，这一点MySQL与Oracle不同，Oracle是通过在数据块中对相应数据行加锁来实现的。InnoDB这种行锁实现特点意味着：只有通过索引条件检索数据，InnoDB才使用行级锁，否则，InnoDB将使用表锁！

