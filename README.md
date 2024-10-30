# Про swap и не только

Источники:

* https://chrisdown.name/2018/01/02/in-defence-of-swap.html
* https://man7.org/linux/man-pages/man5/proc_meminfo.5.html
* https://unix.stackexchange.com/questions/677006/what-is-anon-pages-in-memory
* https://www.kernel.org/doc/gorman/html/understand/understand014.html
* https://docs.kernel.org/admin-guide/mm/zswap.html#zswap

*Обычное описание swap'a выглядит следующим образом:*

>**Swap** is essentially emergency memory; a space set aside for times when your system temporarily needs more physical memory than you have available in RAM. It's considered "bad" in the sense that it's slow and inefficient, and if your system constantly needs to use **swap** then it obviously doesn't have enough memory. […] If you have enough RAM to handle all of your needs, and don't expect to ever max it out, then you should be perfectly safe running without a **swap** space.

> По сути, **swap** — это аварийная память; пространство, отведенное на случай, если вашей системе временно потребуется больше физической памяти, чем имеется в оперативной памяти. Это считается «плохим» в том смысле, что это медленно и неэффективно, и если вашей системе постоянно приходится использовать **swap**, то ей явно не хватает памяти. […] Если у вас достаточно оперативной памяти для удовлетворения всех ваших потребностей, и вы не ожидаете, что она когда-либо будет исчерпана, то вы можете абсолютно спокойно отказаться от использования **swap**

Это не так. **Точнее, не совсем так?**. Но, для начала, требуется рассказать немного про память в Linux.

#### Типы памяти в Linux

В Linux существует множество различных типов памяти, и у каждого типа есть свои свойства. Понимание их нюансов является ключом к пониманию важности подкачки.

Например, есть страницы («блоки» памяти, обычно 4 КБ), отвечающие за хранение кода для каждого процесса, запущенного на вашем компьютере. Есть также страницы, отвечающие за кэширование данных и метаданных, связанных с файлами, к которым обращаются эти программы, для ускорения будущего доступа. Они являются частью кэша страниц, назовем их **файловой памятью**.

Есть также страницы, отвечающие за выделение памяти внутри этого кода, например, когда записывается новая память, выделенная с помощью malloc, или при использовании флага MAP_ANONYMOUS mmap. Это «анонимные» страницы — так называемые, потому что они ничем не подкреплены — назовем их **анонимной памятью**.

Существуют и другие типы памяти — разделяемая память, slab-память, память стека ядра, буферы и т. п. — но *анонимная память* и *файловая память* являются наиболее известными и простыми для понимания, поэтому сосредоточимся на них.

---

Техническое отступление:

**Из man'ов**:

`man mmap`
```
NAME
       mmap, munmap - map or unmap files or devices into memory
...
MAP_ANONYMOUS
        The mapping is not backed by any file; its contents are
        initialized to zero.  The fd argument is ignored; however,
        some implementations require fd to be -1 if MAP_ANONYMOUS
        (or MAP_ANON) is specified, and portable applications
        should ensure this.  The offset argument should be zero.
        Support for MAP_ANONYMOUS in conjunction with MAP_SHARED
        was added in Linux 2.4.
```

`man proc_meminfo 5`
```man
AnonPages %lu (since Linux 2.6.18)
        Non-file backed pages mapped into user-space page
        tables.
```

Процесс отображает память в Linux'e с помощью системного вызова **mmap(2)**. Данная память может быть как подкреплена реальным файлом на диске, так и нет. Во втором случае получится пустой раздел памяти, называемый анонимным.
Многие знакомы с функцией **malloc(3)**, которая используется для аллокации динамической памяти. В Linux'e, в большинстве случаем, под капотом **malloc(3)** использует **mmap(2)**

---

**Не уверен в корректности перевода reclaim в данном случае**

#### Освобождаемая/неосвобождаемая память (Reclaimable/unreclaimable memory)

Один из самых фундаментальных вопросов при рассмотрении конкретного типа памяти — можно ли ее освободить или нет. «Освобождение» (reclaim) здесь означает, что система может, не теряя данных, очистить страницы этого типа из физической памяти.

Для некоторых типов страниц это обычно довольно тривиально. Например, в случае чистой (clean) (немодифицированной) кэш-памяти страниц мы просто кэшируем что-то, что у нас есть на диске, для производительности, поэтому мы можем удалить страницу без необходимости выполнять какие-либо дополнительные действия.

Для некоторых типов страниц это возможно, но не тривиально. Например, в случае грязной (dirty) (модифицированной) кэш-памяти страниц мы не можем просто удалить страницу, потому что на диске еще нет наших изменений. Таким образом, нам нужно либо отклонить освобождение, либо сначала сбросить наши изменения на диск, прежде чем мы сможем удалить эту память.

Для некоторых типов страниц это невозможно. Например, в случае с анонимными страницами, они существуют только в памяти и не имеют другого резервного хранилища.

Отсюда возникает вопрос - как быть с такими страницами? Ответ прост - это то, для чего нужен **swap**

*Swap - это прежде всего механизм для равенства освобождения памяти*

**Добавить больше примеров**
Простой пример, почему может потребоваться сбросить анонимную страницу в swap:

* Во время нормальной работы программы мы можем выделить память, которая используется достаточно редко. Для общей производительности системы имеет смысл сбросить ее на диск и дождаться page fault'a, чтобы подгружать страницы по требованию, а освободившуюся память использовать для чего-то другого, что более важно.

#### Примеры при наличии/отсутствии swap'a

**При отсутствии/низкой конкуренции за память**

* **swap**: мы можем сбросить в swap редко используемую анонимную память, которая может использоваться только в течение небольшой части жизненного цикла процесса, что позволит нам использовать эту память, например, для повышения частоты попаданий в кэш.

* **no swap**: у нас нет такой возможности, так как страница заблокирована в памяти. Хотя это может не сразу проявиться как проблема, при некоторых нагрузках это может представлять собой серьезное падение производительности из-за "старых" анонимных страниц, отнимающих место, которое можно использовать для более важных вещей.

**При умеренной/высокой конкуренции за память**

* **swap**: все типы памяти имеют одинаковую вероятность быть сброшенными в swap. Это означает, что у нас больше шансов успешно освободить страницу, то есть мы можем освободить страницы, которые не будут промахом по кэшу (page fault) в ближайшее время (эффект **пробуксовки**).

*Wiki*
>Пробуксовка (thrashing) - состояние, когда подсистема виртуальной памяти компьютера находится в состоянии постоянного свопинга, часто обменивая данные в памяти и данные на диске, в ущерб выполнению приложений. Это вызывает замедление или даже остановку работы компьютера. Такое состояние может продолжаться неограниченно долго, пока вызвавшие его причины не будут устранены. Когда объём предоставленной процессам памяти превышает объём имеющейся оперативной памяти, часть страниц может быть выгружена на внешний носитель. Поскольку за заданный интервал времени процесс обычно не использует всю доступную ему память, а только её часть, называемую рабочим множеством, это не сказывается на производительности. Однако если сумма рабочих множеств всех процессов превышает объём оперативной памяти, резко возрастает вероятность отказа страницы, то есть отсутствия требуемой страницы в оперативной памяти. Происходит постоянная загрузка страниц рабочих множеств активных процессов и выгрузка страниц неактивных процессов. Поскольку загрузка страницы с внешнего носителя на несколько порядков медленнее обращения к оперативной памяти, производительность компьютера резко падает. Загрузка процессора при этом невысока. Такое состояние и называется **пробуксовкой**.

* **no swap**: анонимные страницы заблокированы в памяти, так как им некуда идти. Вероятность успешного долгосрочного возвращения страниц ниже, так как у нас есть только некоторые типы памяти, которые могут быть возвращены вообще. Риск переполнения страницы выше. Случайный читатель может подумать, что это все равно будет лучше, так как это может избежать необходимости выполнять дисковый ввод-вывод, но это не так — мы просто переносим дисковый ввод-вывод подкачки на удаление горячих кэшей страниц и удаление сегментов кода, которые нам скоро понадобятся.

**При временных всплесках использования памяти**

* **swap**: мы более устойчивы к временным всплескам, но в случаях серьезного голодания памяти период от начала переполнения памяти до OOM killer может быть продлен. Мы лучше видим зачинщиков давления памяти и можем действовать на них более разумно, а также можем выполнять контролируемое вмешательство.

* **no swap**: OOM killer срабатывает быстрее, так как анонимные страницы заблокированы в памяти и не могут быть восстановлены. Мы с большей вероятностью будем перегружать память, но время между переполнением и OOMing сокращается. В зависимости от вашего приложения это может быть лучше или хуже. Например, приложение на основе очередей может желать такого быстрого перехода от переполнения к уничтожению. Тем не менее, это все еще слишком поздно, чтобы быть действительно полезным — OOM killer вызывается только в моменты серьезного голодания, и полагаться на этот метод для такого поведения было бы лучше на более оппортунистическое завершение процессов, как только в первую очередь достигается конкуренция за память.

---

#### In depths

kernel 2.6

```c
 struct swap_info_struct {
     unsigned int flags;
     kdev_t swap_device;
     spinlock_t sdev_lock;
     struct dentry * swap_file;
     struct vfsmount *swap_vfsmnt;
     unsigned short * swap_map;
     unsigned int lowest_bit;
     unsigned int highest_bit;
     unsigned int cluster_next;
     unsigned int cluster_nr;
     int prio;
     int pages;
     unsigned long max;
     int next;
 };
```

kernel 6.x.x
```c
struct swap_info_struct {
	struct percpu_ref users;	/* indicate and keep swap device valid. */
	unsigned long	flags;		/* SWP_USED etc: see above */
	signed short	prio;		/* swap priority of this type */
	struct plist_node list;		/* entry in swap_active_head */
	signed char	type;		/* strange name for an index */
	unsigned int	max;		/* extent of the swap_map */
	unsigned char *swap_map;	/* vmalloc'ed array of usage counts */
	unsigned long *zeromap;		/* kvmalloc'ed bitmap to track zero pages */
	struct swap_cluster_info *cluster_info; /* cluster info. Only for SSD */
	struct list_head free_clusters; /* free clusters list */
	struct list_head full_clusters; /* full clusters list */
	struct list_head nonfull_clusters[SWAP_NR_ORDERS];
					/* list of cluster that contains at least one free slot */
	struct list_head frag_clusters[SWAP_NR_ORDERS];
					/* list of cluster that are fragmented or contented */
	unsigned int frag_cluster_nr[SWAP_NR_ORDERS];
	unsigned int lowest_bit;	/* index of first free in swap_map */
	unsigned int highest_bit;	/* index of last free in swap_map */
	unsigned int pages;		/* total of usable pages of swap */
	unsigned int inuse_pages;	/* number of those currently in use */
	unsigned int cluster_next;	/* likely index for next allocation */
	unsigned int cluster_nr;	/* countdown to next cluster search */
	unsigned int __percpu *cluster_next_cpu; /*percpu index for next allocation */
	struct percpu_cluster __percpu *percpu_cluster; /* per cpu's swap location */
	struct rb_root swap_extent_root;/* root of the swap extent rbtree */
	struct block_device *bdev;	/* swap device or bdev of swap file */
	struct file *swap_file;		/* seldom referenced */
	struct completion comp;		/* seldom referenced */
	spinlock_t lock;		/*
					 * protect map scan related fields like
					 * swap_map, lowest_bit, highest_bit,
					 * inuse_pages, cluster_next,
					 * cluster_nr, lowest_alloc,
					 * highest_alloc, free/discard cluster
					 * list. other fields are only changed
					 * at swapon/swapoff, so are protected
					 * by swap_lock. changing flags need
					 * hold this lock and swap_lock. If
					 * both locks need hold, hold swap_lock
					 * first.
					 */
	spinlock_t cont_lock;		/*
					 * protect swap count continuation page
					 * list.
					 */
	struct work_struct discard_work; /* discard worker */
	struct list_head discard_clusters; /* discard clusters list */
	struct plist_node avail_lists[]; /*
					   * entries in swap_avail_heads, one
					   * entry per node.
					   * Must be last as the number of the
	include/linux/swap.h				   * array is nr_node_ids, which is not
					   * a fixed value so have to allocate
					   * dynamically.
					   * And it has to be an array so that
					   * plist_for_each_* can work.
					   */
};
```

Изменилось в `adfab836f4908deb049a5128082719e689eed964`

```c
struct swap_list_t {
    int head;    /* head of priority-ordered swapfile list */
    int next;    /* swapfile to be used next */
};
```