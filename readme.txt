mariadb-5.5.57-linux-x86_64.tar.gz
官方下载地址: 

https://mirrors.tuna.tsinghua.edu.cn/mariadb//mariadb-10.1.22/source/mariadb-10.1.22.tar.gz 

1、首先查询下是否安装了mysql或者旧版本mariadb
rpm -qa | grep mysql
删除rm -rf /etc/my.cnf
2、安装依赖包
yum install cmake
yum groupinstall -y Development Tools
yum install -y ncurses-devel
openssl-devel openssl yum install  -y  libevent 
3、准备数据存放的文件系统

新建一个逻辑卷，并将其挂载至特定目录即可。这里不再给出过程。

这里假设其逻辑卷的挂载目录为/mydata，而后需要创建/mydata/data目录做为mysql数据的存放目录。

4、新建用户以安全方式运行进程：

# groupadd -r mysql
# useradd -g mysql -r -s /sbin/nologin -M -d /mydata/data mysql
# chown -R mysql:mysql /mydata/data
一、编译安装开始
1、解压
#tar zxf mariadb-10.1.22.tar.gz
#cd mariadb-10.1.22
#cmake . -DCMAKE_INSTALL_PREFIX=/appliction/mysql \      //安装目录
          -DMYSQL_DATADIR=/appliction/mydata \      //数据库存放目录
          -DWITH_INNOBASE_STORAGE_ENGINE=1 \      //支持数据库innobase引擎
          -DWITH_ARCHIVE_STORAGE_ENGINE=1 \      //支持数据库archive引擎
          -DWITH_BLACKHOLE_STORAGE_ENGINE=1 \    //支持数据库blackhole存储引擎
          -DWITH_READLINE=1 \                                    
          -DWITH_SSL=system \                                    
          -DWITH_ZLIB=system \
          -DWITH_LIBWRAP=0 \
          -DMYSQL_UNIX_ADDR=/tmp/mysql.sock \                  
          -DDEFAULT_CHARSET=utf8 \            //字符集utf8
          -DDEFAULT_COLLATION=utf8_general_ci \    //校验字符
          -DENABLED_LOCAL_INFILE=1            //允许本地导入数据
执行编译安装：
cmake . -DCMAKE_INSTALL_PREFIX=/usr/local/mysql -DMYSQL_DATADIR=/mydata/data -DSYSCONFDIR=/etc -DWITHOUT_TOKUDB=1 -DWITH_INNOBASE_STORAGE_ENGINE=1 -DWITH_ARCHIVE_STPRAGE_ENGINE=1 -DWITH_BLACKHOLE_STORAGE_ENGINE=1 -DWIYH_READLINE=1 -DWIYH_SSL=system -DVITH_ZLIB=system -DWITH_LOBWRAP=0 -DMYSQL_UNIX_ADDR=/tmp/mysql.sock -DDEFAULT_CHARSET=utf8 -DDEFAULT_COLLATION=utf8_general_ci
这里说明一下：-DCMAKE_INSTALL_PREFIX是指定安装的位置，这里是/usr/local/mysql，-DMYSQL_DATADIR是指定MySQL的数据目录，这里是/data1/mysql，安装目录和数据目录都可以自定义设置，-DSYSCONFDIR是指定配置文件所在的目录，一般都是/etc ，具体的配置文件是/etc/my.cnf，-DWITHOUT_TOKUDB=1这个参数一般都要设置上，表示不安装tokudb引擎，tokudb是MySQL中一款开源的存储引擎，可以管理大量数据并且有一些新的特性，这些是Innodb所不具备的，这里之所以不安装，是因为一般计算机默认是没有Percona Server的，并且加载tokudb还要依赖jemalloc内存优化，一般开发中也是不用tokudb的，所以暂时屏蔽掉，否则在系统中找不到依赖会出现：CMake Error at storage/tokudb/PerconaFT/cmake_modules/TokuSetupCompilerNaNake:179 (message)这样的错误，然后后面那些参数都是可选的，可以加也可以不加，最后的编码建议设置一下，所以编译指令也可以简化成下面这样：
cmake . -DCMAKE_INSTALL_PREFIX=/usr/local/mysql -DMYSQL_DATADIR=/mydata/data -DSYSCONFDIR=/etc -DWITHOUT_TOKUDB=1 -DMYSQL_UNIX_ADDR=/tmp/mysql.sock -DDEFAULT_CHARSET=utf8 -DDEFAULT_COLLATION=utf8_general_ci
注意：如果万一执行中有了错误，可以执行： rm -f CMakeCache.txt 删除编译缓存，让指令重新执行，否则每次读取这个文件，命令修改正确也是报错
#make -j4
#make install
cmake没问题，可以编译并且安装了： make && make install 时间有点长，耐心等待
执行完成也就是安装完成了，现在执行 cd /usr/local/mysql/ 进入mysql安装目录分别执行下面命令：


2、初始化mysql-5.5.33

# cd /usr/local/mysql 

# chown -R mysql:mysql  .
# scripts/mysql_install_db --user=mysql --datadir=/mydata/data
# chown -R root  .

3、为mysql提供主配置文件：

# cp support-files/my-large.cnf  /etc/my.cnf

并修改此文件中thread_concurrency的值为你的CPU个数乘以2，比如这里使用如下行：
thread_concurrency = 2

另外还需要添加如下行指定mysql数据文件的存放位置：
datadir = /mydata/data


4、为mysql提供sysv服务脚本：

# cd /usr/local/mysql
# cp support-files/mysql.server  /etc/rc.d/init.d/mysqld
# chmod +x /etc/rc.d/init.d/mysqld

添加至服务列表：
# chkconfig --add mysqld
# chkconfig mysqld on

添加PATH变量路径
# vim /etc/profile.d/

mysqld.sh 

 
export PATH=/usr/local/mysql/bin:$PATH 
# . /etc/profile.d/

mysqld.sh 


启动服务
systemctl start mysqld




为了使用mysql的安装符合系统使用规范，并将其开发组件导出给系统使用，这里还需要进行如下步骤：
5、输出mysql的man手册至man命令的查找路径：

c6 编辑/etc/man.config，添加如下行即可：
c6 MANPATH  /usr/local/mysql/man
c7 编辑/etc/man_db.conf
c7 MANPATH_MAP /usr/local/mysql    /usr/local/mysql/man

6、输出mysql的头文件至系统头文件路径/usr/include：

这可以通过简单的创建链接实现：
# ln -sv /usr/local/mysql/include  /usr/include/mysql

7、输出mysql的库文件给系统库查找路径：

# echo '/usr/local/mysql/lib' > /etc/

ld.so 

.conf.d/mysql.conf

而后让系统重新载入系统库：
# ldconfig
完成
