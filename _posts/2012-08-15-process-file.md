进程中的文件表关系
======================

在进程的描述符中，关于文件描述符的定义如下:

    struct task_struct {
        ....
	    struct files_struct *files;
	    ....
	}

这里的file_struct 里面就是文件打开表的一些定义:

    struct files_struct {
 	  atomic_t count;
	  struct fdtable __rcu *fdt;
	  struct fdtable fdtab;
	  
	  spinlock_t file_lock __cacheline__aligned_in_smp;
	  int next_fd;//文件打开时，下一个fd，也就是打开fd的数量
	  .......
	  struct file __rcu *fd_array[NR_OPEN_DEFAULT];//为了加快访问速度
    }

默认，内核只缓存每个进程的文件描述符为:NR\_OPEN\_DEFUALT
          
    #define NR_OPEN_DEFAULT BITS_PER_LONG

在32位系统中，BITS\_PER\_LONG为32，64位系统则64,所有文件打开表的信息都存放在数据结构fdtable中:
     
	struct fdtable {
		unsigned int max_fds; //支持的最大fd个数
		struct file __rcu **fd;//这里就存放了文件打开时的文件了，索引即文件的fd
		fd_set *close_on_exec;
		fd_set *open_fds;//每一位暗示对应的fd是否已经被使用
		struct rcu_head rcu;
		struct fdtable *next;
	}

    typedef struct {
	   unsigned long fds_bits[__FDSET_LONGS];
	} __kernel_fd_set;
	
	typedef __kernel_fd_set fd_set;
	
在每个进程打开文件表的背后就是具体的文件file结构体了，该数据结构定义如下:

    <linux/fs.h>
	
    struct file {
	    union {
		    struct list_head fu_list; 
			struct rcu_head fu_rcuhead;
		} f_u;
		
		struct path f_path;
		const struct file_operations *f_op;//这里就是由文件系统开发者实现了
		struct fown_struct f_owner;//异步读写SIGIO支持
		struct file_ra_state f_ra;//预读取支持
		spinlock_t f_lock;
		atomic_long_t f_count;
		unsigned int f_flags;//open 系统调用标记
		fmode_t f_mode;
		loff_t f_pos; //文件的偏移量
		....
		void   *private_data;//不同文件系统的私有数据.
		struct address_space *f_mapping;//支持mmap 内存映射
		....
	}

    struct path {
	    struct vfsmount *mnt;
		struct dentry *dentry;
	}
	
每个文件的背后管理都是内核态由具体文件系统来实现，而在用户态则体现为路径管理，这里的*struct path*就是这样的桥梁作用.这样来讲具体的文件系统的调用关系图如下:

                         -------> file ----> vfsmnt --> super_block
	file_struct ---->fdtable           ----> dentry --> inode
		                 -------> file 
						 
						 -------> file
						 
当打开文件表的fd多于默认打开的fd个数，这时就需要进行扩展fdtable了，具体实现如下:

    <fs/file.c>
	
    int expand_files(struct files_struct *files,int nr)
	    ----> static int expand_fdtable(struct files_struct *files,int nr)
		
    static int expand_fdtable(struct files_struct *files,int nr)
	{
	     struct fdtable *new_fdt,*cur_fdt;
		 
		 ...
		 new_fdt = alloc_fdtable(nr);
		 ...
		 
		 cur_fdt = files_table(files);
		 if(nr >= cur_fdt->max_fds) {
		    copy_fdtable(new_fdt,cur_fdt);
			rcu_assign_pointer(files->fdt,new_fdt);
			if(cur_fdt->max_fds > NR_OPEN_DEFAULT)
			    free_fdtable(cur_fdt);
		 } else {
		    __free_fdtable(new_fdt);
		 }
	
	     return 1;
	}
	
当进行文件打开时，调用流程如下:

    <fs/open.c>
	open -->
	         do_sys_open
			    ---> do_filp_open
				  ---> finish_open
				    ----> nameidata_to_filp
					   ---> __dentry_open
	
\_\_dentry\_open 函数就是在经过对路径解析(路径到inode节点的翻译)及权限判断，打开标志检查之后，开始进行初始化file数据结构:

    <fs/open.c>
	
    static struct file * __dentry_open(struct dentry *dentry,struct vfsmount *mnt,struct file *f,
	                                   int (*open)(struct inode *,struct file *),
									   const struct cred *cred)
	{
	      struct inode *inode;
		  int error;
		  
		  f->f_mode = OPEN_FMODE(f->f_flags) | FMODE_LSEEK | FMODE_PREAD | FMODE_PWRITE;
		  inode = dentry->d_inode;
		  
		  ....
		  
		  f->f_mapping = inode->i_mapping;
		  f->f_path.dentry = dentry;
		  f->f_path.mnt = mnt;
		  f->f_pos = 0;
		  f->f_op = fops_get(inode->i_fop);
		  file_sb_list_add(f,inode->i_sb);
		  .....
		  if(!open && f->f_op)
		      open = f->f_op->open;
		  if(open) {
		     error = open(inode,f);
			 if(error)
			    goto cleanup_all;
		  }
		  
		  ima_counts_get(f);
		  f->f_flags &= ~(O_CREAT | O_EXCL | O_NOCTTY | O_TRUNC);
		  
		  file_ra_state_init(&f->f_ra,f->f_mapping->host->i_mapping);
		  .....
	}

在dentry缓存中，所有的文件夹和文件均是一个目录项,每一个文件(struct file)都有一个目录项(struct dentry)和inode节点(struct inode),最重要的就是安装上与文件对应的操作函数，并调用对应的回调函数,返回给用户.
在fcntl中有open函数的声明，如下:

    __nonnull:gcc 扩展，规定__file不为NULL
    extern int open(__const char *__file,int __oflags, ...) __nonnull ((1));

当文件关闭时，则调用流程如下:
   
    <fs/file_table.c>
    sys_close -->
	            filp_close --->
				              fput --> 
							         __fput
	
	int filp_close(struct file *filp,fl_owner_t id)
	{
		int retval = 0;
		
		//正在使用的文件对象为0,明显出错了
		if(!file_count(filp)) {
		    printk(KERN_ERR "VFS: Close: file count is 0\n");
			return 0;
		}
		
		if(filp->f_op && filp->f_op->flush)
		    retval = filp->f_op->flush(filp,id);
			
		dnotify_flush(filp,id);
		locks_remove_posix(filp,id);
		//减少引用技术，当计数为0时，调用注册的f_op release/faysnc 方法,释放系统资源
		fput(filp);
		
		return retval;
	}
