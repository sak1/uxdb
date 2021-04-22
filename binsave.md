# UXDB实例

## 二进制数据存在哪儿好？

在大数据业务场景中经常被问到的一个问题是，二进制数据存在库外好，还是库内好？还有库内存储二进制数据有两种方法，哪种更好？

我决定对可用选项进行基准测试，给这一问题一些数据和参考答案

### 1 将数据存储在数据库外部

为此，您将二进制数据存储在数据库外部的文件中，并将文件的路径存储在数据库中。

缺点是不能以这种方式保证一致性。利用数据库中的所有数据，UXDB可以保证事务的原子性。因此，将数据存储在数据库外部会使体系结构和实现更加复杂。

一种一致性的方法是始终在将文件元数据存储到数据库之前添加文件。删除文件的元数据后，您只需将文件保留在文件系统上即可。这样，数据库中就不会存在文件系统中不存在的路径。您可以使用脱机重组运行来清理文件系统，以摆脱孤立的文件。

这种方法有两个主要优点：

它使数据库较小，从而使维护更加容易。例如，执行文件系统的增量备份比执行数据库更容易。
直接从文件系统读取文件的性能必须更好。毕竟，数据库也存储在文件中，并且必须有一定的开销。

### 2 将数据存储在大对象中

UXDB大对象是在UXDB中存储二进制数据的“老方法”。系统为oid大型对象分配一个（4字节无符号整数），将其拆分为2kB的块并将其存储在ux_largeobject目录表中。

您通过引用大对象oid，但是oid表中存储的存储和关联的大对象之间没有依赖关系。如果删除表行，则必须显式删除大对象（或使用触发器）。

大对象很麻烦，因为使用大对象的代码必须使用特殊的大对象API。在SQL标准不涵盖，而不是所有的客户端API都支持它。

大对象有两个优点：您可以使用大对象存储任意大数据，大对象API支持流传输，即以块的形式读取和写入大对象

### 3 将数据存储为 bytea

bytea（“ byte a rray”的缩写）是在UXDB中存储二进制数据的“新方法”。它使用TOAST（超大属性存储技术，被UXDB社区自豪地称为“自切成薄片以来最好的东西”）来透明地存储脱机数据。

Abytea直接存储在数据库表中，并且在删除表行时消失。无需特殊维护。

主要缺点bytea是：

像所有TOASTed数据类型一样，绝对长度限制为1GB
当您读写时bytea，所有数据都必须存储在内存中（不支持流传输）

如果选择bytea，则应该了解TOAST的工作原理：

对于可能超过2000字节的新表行，如果可能，将压缩可变长度的数据类型
如果压缩后的数据仍然超过2000个字节，则UXDB将可变长度的数据类型拆分为多个块，并将它们不成行地存储在特殊的“ TOAST表”中
现在，对于已经压缩的数据，第一步是不必要的，甚至是有害的。压缩数据后，UXDB将意识到压缩的数据实际上已经增长了（因为UXDB使用了快速压缩算法）并丢弃了它们。这是不必要的CPU时间浪费。

而且，如果仅检索TOASTed值的子字符串，则UXDB仍必须检索解压缩该值所需的所有块。

幸运的是，UXDB允许您指定TOAST应该如何处理列。默认EXTENDED存储类型如上所述。如果我们选择EXTERNAL改为，则值将存储在行外，但不会压缩。这样可以节省CPU时间。它还允许仅需要数据子字符串的操作仅访问包含实际数据的那些块。

因此，您应始终将压缩二进制数据的存储类型更改为EXTERNAL。这也使我们能够使用substr函数（至少在读取操作中）实现流传输（请参见下文）。

bytea我在此基准测试中使用的表的定义如下

```
CREATE TABLE bins (
   id bigint PRIMARY KEY,
   data bytea NOT NULL
);
ALTER TABLE bins ALTER COLUMN data SET STORAGE EXTERNAL;
```

### 对不同方法进行基准测试

选择用Java编写小基准，为读取二进制数据的代码编写一个接口，因此可以轻松地用相同的代码测试不同的实现。这也使比较实现变得更加容易：

```
import java.io.EOFException;
import java.io.IOException;
import java.sql.SQLException;

public interface LOBStreamer {
    public final static int CHUNK_SIZE = 1048576;
    public int getNextBytes(byte[] buf)
            throws EOFException, IOException, SQLException;
    public void close() throws IOException, SQLException;
}
```


CHUNK_SIZE 是读取数据的单位。

#### 从文件系统读取二进制数据的代码

在构造函数中，查询数据库以获取文件的路径。该文件已打开以供阅读；块被读入getNextBytes。

```
import java.io.IOException;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.io.EOFException;
import java.io.File;
import java.io.FileInputStream;

public class FileStreamer implements LOBStreamer {
    private FileInputStream file;
     
```


    public FileStreamer(java.sql.Connection conn, long objectID)
            throws IOException, SQLException {
        PreparedStatement stmt = conn.prepareStatement(
                "SELECT path FROM lobs WHERE id = ?");
        stmt.setLong(1, objectID);
        ResultSet rs = stmt.executeQuery();
        rs.next();
        String path = rs.getString(1);
     
        this.file = new FileInputStream(new File(path));
     
        rs.close();
        stmt.close();
    }
     
    @Override
    public int getNextBytes(byte[] buf)
            throws EOFException, IOException {
        int result = file.read(buf);
     
        if (result == -1)
            throw new EOFException();
     
        return result;
    }
     
    @Override
    public void close() throws IOException {
        file.close();
    }
}

#### 从大对象读取二进制数据的代码

大对象在构造函数中打开。请注意，所有读取操作必须在打开大对象的同一数据库事务中进行。

由于SQL或JDBC标准未涵盖大对象，因此我们必须使用JDBC驱动程序的UXDB特定扩展。这使得代码无法移植到其他数据库系统。

但是，由于大型对象专门支持流传输，因此代码比其他选项更简单。

```

```

    import java.io.EOFException;
    import java.io.IOException;
    import java.sql.Connection;
    import java.sql.SQLException;
    
    import org.postgresql.PGConnection;
    import org.postgresql.largeobject.LargeObject;
    import org.postgresql.largeobject.LargeObjectManager;
    
    public class LargeObjectStreamer implements LOBStreamer {
        private LargeObject lob;
    public LargeObjectStreamer(Connection conn, long objectID)
            throws SQLException {
        PGConnection pgconn = conn.unwrap(PGConnection.class);
        this.lob = pgconn.getLargeObjectAPI().open(
                        objectID, LargeObjectManager.READ);
    }
     
    @Override
    public int getNextBytes(byte[] buf)
            throws EOFException, SQLException {
        int result = lob.read(buf, 0, buf.length);
     
        if (result == 0)
            throw new EOFException();
     
        return result;
    }
     
    @Override
    public void close() throws IOException, SQLException {
        lob.close();
    }
}
从读取二进制数据的代码 bytea
构造函数检索值的长度，并准备一条语句以获取二进制数据的大块。

请注意，该代码比其他示例更复杂，因为我必须自己实现流式传输。

使用这种方法，我不需要在单个事务中读取所有块，但是我这样做是为了使示例尽可能相似。





```java
import java.io.EOFException;
import java.io.IOException;
import java.io.InputStream;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

public class ByteaStreamer implements LOBStreamer {
    private PreparedStatement stmt;
    private Connection conn;
    private int position = 1, size;public ByteaStreamer(Connection conn, long objectID)
        throws SQLException {
    PreparedStatement len_stmt = conn.prepareStatement(
            "SELECT length(data) FROM bins WHERE id = ?");
    len_stmt.setLong(1, objectID);
    ResultSet rs = len_stmt.executeQuery();
 
    if (!rs.next())
        throw new SQLException("no data found", "P0002");
 
    size = rs.getInt(1);
 
    rs.close();
    len_stmt.close();
 
    this.conn = conn;
    this.stmt = conn.prepareStatement(
            "SELECT substr(data, ?, ?) FROM bins WHERE id = ?");
    this.stmt.setLong(3, objectID);
}
 
@Override
public int getNextBytes(byte[] buf)
        throws EOFException, IOException, SQLException {
    int result = (position > size + 1 - buf.length) ?
                    (size - position + 1) : buf.length;
 
    if (result == 0)
        throw new EOFException();
 
    this.stmt.setInt(1, position);
    this.stmt.setInt(2, result);
 
    ResultSet rs = this.stmt.executeQuery();
 
    rs.next();
 
    InputStream is = rs.getBinaryStream(1);
    is.read(buf);
 
    is.close();
    rs.close();
 
    position += result;
 
    return result;
}
 
@Override
public void close() throws SQLException {
    this.stmt.close();
}
}
```
#### 读取二进制数据的基准结果

我执行的基准测试的代码上一台笔记本电脑和英特尔®酷睿™i5-8565U CPU和SSD存储。使用的UXDB版本是2.1.0.12。数据缓存在RAM中，因此结果不反映磁盘I / O开销。数据库连接使用回送接口将网络影响降至最低。

此代码用于运行测试：

```java
Class.forName("org.postgresql.Driver");

Connection conn = DriverManager.getConnection(
        "jdbc:postgresql:test?user=laurenz&password=...");

// important for large objects
conn.setAutoCommit(false);

byte[] buf = new byte[LOBStreamer.CHUNK_SIZE];

long start = System.currentTimeMillis();

for (int i = 0; i < LOBStreamTester.ITERATIONS; ++i) {
    // set LOBStreamer implementation and object ID as appropriate
    LOBStreamer s = new LargeObjectStreamer(conn, 62409);
try {
    while (true)
        s.getNextBytes(buf);
} catch (EOFException e) {
    s.close();
}
 
conn.commit();
}

System.out.println(
        "Average duration: " +
        (double)(System.currentTimeMillis() - start)
            / LOBStreamTester.ITERATIONS);
         
conn.close();
```

我在一个紧密的循环中多次运行了每个测试，都使用一个大文件（350 MB）和一个小文件（4.5 MB）。这些文件是压缩的二进制数据。

350 MB数据	4.5 MB数据
文件系统	46毫秒	1毫秒
大物件	950毫秒	8毫秒
bytea	590毫秒	6毫秒

### 总结一下

在我的基准测试中，从数据库中检索二进制对象的速度大约比从文件系统中的文件读取二进制对象的速度慢十倍。出人意料的是，从流bytea与EXTERNAL存储是可测量比从大对象流更快。由于大型对象专门支持流式传输，因此我料想相反。

综上所述，以下是每种方法的优缺点：

|                                | 优点                                                   | 缺点                                                         |
| ------------------------------ | ------------------------------------------------------ | ------------------------------------------------------------ |
| 文件系统中的二进制数据（外存） | 迄今为止最快的方法<br />小型数据库，易于维护，例如备份 | 难以保证一致性<br/>架构复杂                                  |
| 二进制数据作为大对象(lo)       | 自动保证一致性<br/>API支持流                           | 表现差<br/>非标准API<br/>需要特殊维护，例如DELETE数据库中的触发器<br/>库越来大而笨拙 |
| 二进制数据为bytea              | 自动保证一致性<br />与标准SQL一起使用                  | 表现不佳<br/>写入无法流式传输（需要大量RAM）<br/>库越来大而笨拙 |

当简单的体系结构和易于编码是主要目标，而高性能并不重要时，在小规模上将大型二进制数据存储在数据库中只是个好主意。然后，增加数据库大小也不会是大问题。

仅当数据超过1GB或对数据库进行流写入很重要时，才应使用大对象。

https://www.cybertec-postgresql.com/en/binary-data-performance-in-postgresql/
©劳伦兹·阿尔贝2020