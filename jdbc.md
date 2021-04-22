# UXDB实例

## Windows下JDBC连接UXDB

### 1 安装配置JDK软件

### 下载安装JDK

转到https://www.oracle.com/java/technologies/javase/javase-jdk8-downloads.html。选择适当的JDK软件，然后单击“下载”。双击按一步安装。JDK软件已安装在您的计算机上，如在C:\Program Files\Java\jdk1.8.0_281目录中。如果需要，您可以把JDK安装到其它个位置。

#### 设置JAVA_HOME：

右键单击“我的电脑”，然后选择“属性”。
在“高级”选项卡上，选择“环境变量”，然后编辑以下三个变量，例如。

```
JAVA_HOME
C:\Program Files\Java\jdk1.8.0_281
```

```
CLASSPATH
.;%JAVA_HOME%\lib;%JAVA_HOME%\lib\tools.jar
```

```
PATH
%JAVA_HOME%\bin
%JAVA_HOME%\jre\bin 
```

### 测试是否成功

打开cmd窗口，输入java -version查看是否安装配置成功

```shell
C:\java -version
java version "1.8.0_281"
Java(TM) SE Runtime Environment (build 1.8.0_281-b09)
Java HotSpot(TM) 64-Bit Server VM (build 25.281-b09, mixed mode)
```

### 2 配置eclipse环境

新建java project
打开IntelliJ IDEA，新建名为uxdb-jdbc 的 java 项目

```
new —>java —>project
Project SDK 选JDK安装目录
addtional libraries and_frameworks选SQL Support
Default Dialect：选postgresql
```

在右上角“database”中增加“data sources and drivers”
正确填写 name general中的相关信息（略），Test Connection，到Successful
至此，数据库连接完成。

### 3 程序测试

把以下两个文件复制到新建的项目中(/src目录下)，修改IP和PWD，City表为历史表，您也可以换成您库中已有的表。

ConnTest.java

```java
import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;

public class ConnTest {
    public static void main(String[] args) {
        Connection conn = ConnUtil.getConn();
        String sql = "select * from city";
        Statement stmt;
        stmt = null;
        ResultSet rs;
        rs = null;
        try {
            stmt = conn.createStatement();
            rs = stmt.executeQuery(sql);
            System.out.println("city");
            while (rs.next()) {
                System.out.print(rs.getString(1));
                System.out.print("   " + rs.getString(2));
                System.out.print("\n");
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

ConnUtil.java

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class ConnUtil {
    public static Connection getConn() {
        Connection conn;
        conn = null;
        try {
            Class.forName("com.uxsino.uxdb.Driver");
            String url = "jdbc:uxdb://192.168.138.131:5432/uxdb";
            try {
                return DriverManager.getConnection(url, "uxdb", "mypassword");
            } catch (SQLException e) {
                e.printStackTrace();
            }
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return conn;
    }
}
```

RUN—>Eidt configuration

VM options参考配置如下：

```
-server
-Xms128m
-Xmx512m
-XX:ReservedCodeCacheSize=240m
-XX:+IgnoreUnrecognizedVMOptions
-XX:+UseConcMarkSweepGC
-XX:SoftRefLRUPolicyMSPerMB=50
-ea
-Dsun.io.useCanonCaches=false
-Djava.net.preferIPv4Stack=true
-XX:+HeapDumpOnOutOfMemoryError
-XX:-OmitStackTraceInFastThrow
-XX:MaxJavaStackTraceDepth=-1
```

RUN，结果如下：

```shell
C:\Program Files\Java\jdk1.8.0_281\bin\java.exe" -server -Xms128m -Xmx512m -XX:ReservedCodeCacheSize=240m -XX:+IgnoreUnrecognizedVMOptions -XX:+UseConcMarkSweepGC -XX:SoftRefLRUPolicyMSPerMB=50 -ea -Dsun.io.useCanonCaches=false -Djava.net.preferIPv4Stack=true -XX:+HeapDumpOnOutOfMemoryError -XX:-OmitStackTraceInFastThrow -XX:MaxJavaStackTraceDepth=-1 "-javaagent:C:\Program Files\JetBrains\IntelliJ IDEA 2018.2.3\lib\idea_rt.jar=1917:C:\Program Files\JetBrains\IntelliJ IDEA 2018.2.3\bin" -Dfile.encoding=UTF-8 -classpath C:\Users\tyler\AppData\Local\Temp\classpath1384363653.jar ConnTest
city
中国   西安
中国   襄阳
中国   喀什
蒙古   乌兰巴托
蒙古   苏赫巴托尔
```

如果您在此过程里遇到

- Error: 找不到或者无法加载主类

- Error: 无法加载JVM DLL C：\ Program Files \ Java \ jdk1.8.0_112
- Error: Could not create the Java Virtual Machine. 

等问题，请仔细检查上述步骤，正确配置JDK以及IDEA中有与JDK的参数。