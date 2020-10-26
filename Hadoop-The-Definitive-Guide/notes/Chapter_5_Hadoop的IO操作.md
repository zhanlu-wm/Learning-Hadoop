# Chapter_5 Hadoop的I/O操作

&emsp;&emsp;Hadoop自带一套原子操作用于数据的I/O操作。

## 5.1 数据完整性

&emsp;&emsp;检测数据是否损坏的常见措施是，在数据第一次引入系统时，计算校验和（checkusm），并在数据通过一个不可靠的通道进行传输时，再次计算校验和，这样就能检测数据是否损坏。如果新计算的校验和与原来的校验和不匹配，就认为数据已损坏。该技术只能检测数据错误，并不能修复数据。——另外，校验和也可能是损坏的，但由于校验和比数据小得多，所以损坏的可能性非常小。

&emsp;&emsp;常用的错误检测码是CRC-32(32位循环冗余校验)，任何大小的数据输入均计算得到一个32位的整数校验和。Hadoop ChecksumFileSystem使用CRC-32计算校验和，HDFS则使用一个更有效的变体CRC-32C计算校验和。

### 5.1.1 HDFS的数据完整性

&emsp;&emsp;HDFS会对写入的所有数据计算校验和，并在读取数据时验证校验和。它针对每个由dfs.bytes-per-checksum指定的字节数大小的数据计算校验和，默认情况下为512个字节，由于CRC-32校验和是4个字节，因而存储校验和的额外开销低于1%.

1. 向一个datanode写入数据时验证校验和：

    datanode负责在收到数据后，在存储该数据及其校验和之前对该数据进行校验，它在收到客户端的数据或者复制其他datanode的数据时执行该操作。正在写数据的客户端将数据及其校验和发送到由一系列datanode组成的管线，管线中的最后一个datanode负责验证校验和。如果datanode检测到错误，客户端便会收到一个IOException异常的子类，对该异常，应以应用程序特定的方式进行处理，比如重试这个操作。

2. 从一个datanode读取数据时验证校验和：

    客户端从datanode读取数据时也会验证校验和，将他们与datanode中存储的校验和进行比较。每个datanode均保存有一个用户验证的校验和日志，因而它知道每个数据块最后一次的验证时间。客户端读取并成功验证一个数据块后，会告诉该datanode，datanode会更新相应日志。

3. datanode定期验证：

    除了客户端在读取数据块儿时会验证校验和，每个datanode也会在后台运行一个DataBlockScanner线程，从而定期验证存储在这个datanode上的所有数据块。该措施是解决物理存储媒介上位损坏的有力措施。

4. 数据块损坏后的修复：

    由于HDFS存储着每个数据块的一定量的副本，因而它可以通过数据副本修复损坏的数据块，进而得到一个新的、完好无损的副本：基本思路是，客户端在读取数据时，如果检测到错误，首先向namenode报告已损坏的数据块及其正在尝试进行数据读取操作的datanode，再抛出ChecksumException。namenode会将这个数据块副本标记为已损坏，这样它将不会再把客户端针对这个数据块的处理请求发送到这个节点。之后，它会安排将这个数据块的另一个副本复制到另一个datanode，这样一来，该数据块的副本因子又会回到期望水平，此后，损坏的数据块副本会被删除。

5. 禁止校验和验证：

    在使用open()方法读取文件之前，将false值传给FileSystem对象的setVerifyChecksum()方法，即可禁止校验和验证。在命令行解释器中使用带-ignoreCrc选项的-get命令，或者使用等价的-copyToLocal命令，也可以达到同样的效果。

    可以用hadoop命令的fs -checksum来检查一个文件的校验和。

### 5.1.2 LocalFileSystem

    TODO

### 5.1.3 ChecksumFileSystem

    TODO


## 5.2 压缩

&emsp;&emsp;文件压缩有两大好处：减少存储文件所需要的磁盘空间，并加速数据在网络和磁盘上的传输。这两大好处在处理大量数据时相当重要。

>有很多种不同的压缩格式、工具和算法，它们各有千秋。下边列出了与Hadoop结合使用的常见压缩方法:
>
>**表5-1 压缩格式总结**：
>|压缩格式|工具|算法|文件扩展名|是否可切分|
>|------------|------|------|--------------|---------------|
>|DEFLATE|无|DEFLATE|.deflate|否|
>|gzip|gzip|DEFLATE|.gz|否|
>|bzip2|bzip2|bzip2|.bz2|是|
>|LZO|lzop|LZO|.lzo|否|
>|LZ4|无|LZ4|.lz4|否|
>|Snappy|无|Snappy|.snappy|否|
>
>**注：**
>
>1.DEFLATE是一个标准压缩算法，该算法的标准实现是zlib。没有可用于生成DEFLATE文件的命令行工具，因为通常都是用gzip格式。gzip格式只是在DEFLATE格式上增加了文件头和文件尾。.deflate文件扩展名是Hadoop约定的。
>
>2.对于lzo文件，如果已经在预处理过程中被添加了索引，那么该lzo文件是可切分的。

&emsp;&emsp;所有压缩算法都要权衡空间和时间，更快的压缩和解压缩速度，通常意味这更低的压缩比率。上表列出的所有压缩工具都提供9个不同的选项来控制压缩时必须考虑的权衡，选项-1为尽可能优化压缩速度，选项-9为尽可能优化压缩比率。

### 5.2.1 codec

&emsp;&emsp;Codec是压缩-解压缩算法的一种实现。在Hadoop中，一个对CompressionCodec接口的实现，代表一个codec，例如GzipCodec包装了gzip的压缩和解压缩算法。

>**表5-2 Hadoop的压缩codec**
>|压缩格式|HadoopCompressionCodec|
>|------------|-------------------------------------|
>|DEFLATE|org.apache.hadoop.io.compress.DefaultCodec|
>|gzip|org.apache.hadoop.io.compress.GzipCodec|
>|bzip2|org.apache.hadoop.io.compress.Bzip2Codec|
>|LZO|com.hadoop.compression.lzo.lzopCodec|
>|LZ4|org.apache.hadoop.io.compress.Lz4Codec|
>|Snappy|org.apache.hadoop.io.compress.SnappyCodec|
>
>**注：**
>
>LzopCodec与lzop工具兼容，LzopCodec基本上是LZO格式的，但它还包含额外的文件头。此外，也有针对纯LZO格式的LzoCodec，并使用.lzo_deflate作为文件扩展名。










    




















