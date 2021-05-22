# shell 综合案例

1. 每天凌晨2:10分备份数据库shelltest到/logs/backup/db/下
2. 备份开始和结束能够给出相应的提示信息
3. 备份后的文件已备份时间为文件名，并打包成.tar.gz的形式
4. 在备份的同时，检查是否由10天前的数据文件，有的话就删除

```shell
#!/bin/bash
#目录
BACKUP=/logs/backup/db
#当前时间作为文件名
DATETIME=$(date +%Y_%m_%d_%H%M%S)
#echo $DATETIME
echo "=====开始备份====="
echo "====备份的路径是 $BACKUP/$DATETIME.tar.gz"

#主机
HOST=localhost
#用户名
DB_USER=root
#密码
DB_PWD=root
#备份数据库名
DATABASE=shelltest
#创建备份的路径
#如果备份的路径文件夹存在，就使用，否则就创建
[ ! -d "$BACKUP/$DATETIME" ] && mkdir -p "$BACKUP/$DATETIME"
#执行mysql的备份数据库命令
mysqldump -u${DB_USER} -p${DB_PWD} --host=$HOST $DATABASE | gzip  > $BACKUP/$DATETIME/$DATETIME.sql.gz
#打包备份文件
cd $BACKUP
tar -zcvf $DATETIME.tar.gz  $DATETIME
#删除临时目录
rm -rf $BACKUP/$DATETIME

#删除10天前的备份文件
find $BACKUP -mtime +10 -name "*.tar.gz" -exec rm -rf {} \;
echo "==备份文件成功===;"
```

之后执行crontab -e 

`10 2 * * * /usr/bin/mysql_db_backup.sh`



