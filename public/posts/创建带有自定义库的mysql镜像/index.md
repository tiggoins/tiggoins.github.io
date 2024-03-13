# 创建带有自定义库的Mysql镜像

默认情况下，在mysql容器中创建新库时需要先运行mysql容器，把需要的sql文件通过docker cp的方式拷贝至容器内，再通过mysql的子命令将sql文件导入。过程比较繁琐，如果是公司的项目部署，可以创建带有公司的项目sql的自定义mysql镜像，避免繁琐的流程。
<!--more-->



### 背景分析

首先拉取官方镜像：

`docker pull mysql:5.7.30`

查看镜像的构建历史

```
[root@bochs docker]# docker history mysql:5.7.30 
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
...               
<missing>           29 hours ago        /bin/sh -c #(nop)  ENTRYPOINT ["docker-entrypoint.sh"]   0B                              
...    
```

可以发现，默认的entrypoint为`docker-entrypoint.sh`

进入容器内，可以发现`docker-entrypoint.sh`其实是个软连接

```
lrwxrwxrwx   1 root root   34 May 15 20:11 entrypoint.sh -> usr/local/bin/docker-entrypoint.sh
```

查看此脚本，发现脚本中已经定义了初始化的代码：

1、`docker_process_init_file()`函数定义了初始文件的格式，其中调用了`docker_process_sql`来完成新库创建及数据导入。

初始化文件可以是.sh  .sql  .sql.gz .sql.xz格式中的任一种

```
  docker_process_init_files() {
          # mysql here for backwards compatibility "${mysql[@]}"
          mysql=( docker_process_sql )
  
          echo
          local f
          for f; do
                  case "$f" in
                          *.sh)
                                  # https://github.com/docker-library/postgres/issues/450#issuecomment-393167936
                                  # https://github.com/docker-library/postgres/pull/452
                                  if [ -x "$f" ]; then
                                          mysql_note "$0: running $f"
                                          "$f"
                                  else
                                          mysql_note "$0: sourcing $f"
                                          . "$f"
                                  fi
                                  ;;
                          *.sql)    mysql_note "$0: running $f"; docker_process_sql < "$f"; echo ;;
                          *.sql.gz) mysql_note "$0: running $f"; gunzip -c "$f" | docker_process_sql; echo ;;
                          *.sql.xz) mysql_note "$0: running $f"; xzcat "$f" | docker_process_sql; echo ;;
                          *)        mysql_warn "$0: ignoring $f" ;;
                  esac
                  echo
          done
}
```

2、`docker_process_sql`使用mysql命令导入数据库文件

```
docker_process_sql() {
         passfileArgs=()
         if [ '--dont-use-mysql-root-password' = "$1" ]; then
                 passfileArgs+=( "$1" )
                 shift
         fi
        
         if [ -n "$MYSQL_DATABASE" ]; then
                 set -- --database="$MYSQL_DATABASE" "$@"
         fi
 
         mysql --defaults-extra-file=<( _mysql_passfile "${passfileArgs[@]}") --protocol=socket -uroot -hlocalhost --socket="${SOCKET}" "$@"
}

```

3、在主函数调用了docker_process_init_file()函数,初始化的文件全部位于`/docker-entrypoint-initdb.d/`中

```
365         docker_process_init_files /docker-entrypoint-initdb.d/*
```

只要把sql文件放入该目录我们就可以通过docker build命令来创建带有自定义库的mysql镜像了。



### 实现过程

创建新的Dockerfile文件，将所需要的用到的sql文件拷贝到/docker-entrypoint-initdb.d/目录中

```
 FROM mysql:5.7.30
 COPY ./mysql/initsql/*.sql /docker-entrypoint-initdb.d/
```

docker-compose文件如下

```
version: "3.3"
   
services:
  mysql:
    build:
      context: .
      dockerfile: Dockerfile
    image: mysql_modified:v1.0
    container_name: mysql_modified
    ports:
     - target: 3306
       published: 3306
       protocol: tcp
       mode: host
    volumes:
       - /home/docker/mysql/conf/:/etc/mysql/conf.d/
       - /home/docker/mysql/data/:/var/lib/mysql/
    environment:
       - MYSQL_ROOT_PASSWORD=Zxczxc@123
```

> 如果是用mysqldump导出的sql文件，必须要加上-B参数保留创库语句。



新建测试用的test.sql文件  

```
CREATE DATABASE IF NOT EXISTS DB_TEST;

USE DB_TEST;
 
CREATE TABLE employees (
    emp_no       INT             NOT NULL COMMENT '主键',
    birth_date    DATE           NOT NULL COMMENT '生日',
    first_name   VARCHAR(14)     NOT NULL COMMENT '用户-姓',
    last_name    VARCHAR(16)     NOT NULL COMMENT '用户-名',
    gender        ENUM ('M','F') NOT NULL COMMENT '性别',
    hire_date     DATE           NOT NULL COMMENT '入职时间',
    PRIMARY KEY (emp_no)
);

INSERT INTO `employees` VALUES 
(10001,'1953-09-02','Georgi','Facello','M','1986-06-26'),
(10002,'1964-06-02','Bezalel','Simmel','F','1985-11-21'),
(10003,'1959-12-03','Parto','Bamford','M','1986-08-28'),
(10004,'1954-05-01','Chirstian','Koblick','M','1986-12-01');
```



整个构建目录结构如下：

```
[root@bochs /]# tree home
home
`-- docker
    |-- docker-compose.yml
    |-- Dockerfile
    `-- mysql
        |-- conf
        |   `-- my.cnf
        |-- data
        `-- initsql
            `-- test.sql

5 directories, 4 files
```

使用`docker-compose build`命令构建新的带有默认库mysql镜像

```
[root@bochs docker]# docker-compose build
Building mysql
Step 1/2 : FROM mysql:5.7.30
 ---> b84d68d0a7db
Step 2/2 : COPY ./mysql/initsql/*.sql /docker-entrypoint-initdb.d/
 ---> c64103df17a2

Successfully built c64103df17a2
Successfully tagged mysql_modified:v1.0

[root@bochs docker]# docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
mysql_modified      v1.0                c64103df17a2        32 seconds ago      448MB
mysql               5.7.30              b84d68d0a7db        30 hours ago        448MB
```

完成镜像创建后使用`docker-compose up -d`启动该容器并查看数据，新库已成功创建，数据也正常。

```
[root@bochs docker]# docker-compose up -d

[root@bochs docker]# docker exec -it  mysql_modified mysql -uroot -p'Zxczxc@123' -e 'show databases;'
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| information_schema |
| DB_TEST            |
| mysql              |
| performance_schema |
| sys                |
+--------------------+

[root@bochs docker]# docker exec -it  mysql_modified mysql -uroot -p'Zxczxc@123' -e 'select * from DB_TEST.employees;\G'
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------+------------+------------+-----------+--------+------------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  |
+--------+------------+------------+-----------+--------+------------+
|  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |
|  10002 | 1964-06-02 | Bezalel    | Simmel    | F      | 1985-11-21 |
|  10003 | 1959-12-03 | Parto      | Bamford   | M      | 1986-08-28 |
|  10004 | 1954-05-01 | Chirstian  | Koblick   | M      | 1986-12-01 |
+--------+------------+------------+-----------+--------+------------+
```



### 总结

放入`/docker-entrypoint-initdb.d/`目录中的文件只会在构建镜像时执行一次

可以将多个数据库放入`/docker-entrypoint-initdb.d/`中达到批量化创建的目的

