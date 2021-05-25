# java io   

## IO简介

### 概念

数据流是一组有序、有起点有终点的字节数据序列，包括输入流和输出流。Java中的流分为两种：1）字节流：数据流中最小的数据单位事字节；2）字符流：数据流中最小的数据单位事字符，Java中字符是Unicode编码，一个字符占两个字节。  Java IO框架如下图所示： ![](C:\Users\27253\AppData\Roaming\Typora\typora-user-images\image-20210507163328491.png)

IO流嵌套使用了装饰模式，比如：

```Java
DataOutputStream data = new DataOutputStream(
	new BufferOutputStream(
    	new FileOutputStream(
        	new File(file)
        )
    )
) 
```

分析上面代码：写文件首先需要创建一个FileOutputStream，此时写文件是每写一个byte都会访问一次磁盘磁头，所以效率很低。为了提高速度，可以包裹一层具备缓存功能的BufferOutputStream。而为了可以直接输出java的基本数据类型，可以将缓存数据流传递给DataOutputStream。从上面的关系可以看出，其目的都是给OutputStream穿了一层又一层的“衣服”，这就是装饰者模式。下面三张图是源码的截图：

<center class="half">
    <img src="C:\Users\27253\AppData\Roaming\Typora\typora-user-images\image-20210507162856493.png" alt="image-20210507162856493" style="zoom: 80%;" />
    <img src="C:\Users\27253\AppData\Roaming\Typora\typora-user-images\image-20210507163033072.png" alt="image-20210507163033072" style="zoom: 80%;" />
    <img src="C:\Users\27253\AppData\Roaming\Typora\typora-user-images\image-20210507162941453.png" alt="image-20210507162941453" style="zoom:80%;" />
</center>

其中只有BufferOutputStream有flush()，因为在使用BufferOutputStream的时候，需要使用指定大小的byte数组，每次数组读满，都会自动向磁盘写入，最后一次不一定能够写满数组，所以需要使用flush()强制写入磁盘。

