# Shell 读取控制台输入和函数

## 基本语法

read(选项)（参数）

**选项：**

-p : 读取值时的提示符

-t  : 指定读取值时等待的时间（秒），如果没有在指定的时间内输入，就不用等待了

**参数：**

指定读取值的变量名

## 函数

### 系统函数

* basename

  功能：返回完整路径最后/的部分，常用于获取文件名

  basename [pathname] [suffix]

  选项：

  suffix为后缀，如果suffix被指定了，basename会将pathname或string中的suffix去掉。

* dirname

  功能：返回完整路径最后/的前面的部分，常用于返回路径部分

  dirname 文件绝对路径（从给定的包含绝对路径的文件名中去除文件名（非目录部分），然后返回剩下的路径（目录的部分））

### 自定义函数

* 基本语法

  [function] funname[()] {

  ​    ACTION;

  ​    [return int;]

  }

  调用直接写函数名：funname [值]

* 实例

  ```shell
  #!/bin/bash
  # 计算两个参数之和
  function getSum() {
        SUM=$[$n1+$n2]
       # echo "里面和是=$SUM"
       echo "$SUM"
       return $?
  }
  
  read -p "请输入第一个参数" n1
  read -p "请输入第二个参数" n2
  total=$(getSum $n1 $n2)
  echo "外面和为$total"
  
  ```

  代码中总共执行了两次 echo 命令，但是却只输出一次，**这是因为`$()`捕获了第一个 echo 的输出结果，它并没有真正输出到终端上**。除了`$()`，你也可以使用\`  \`来捕获 echo 的输出结果