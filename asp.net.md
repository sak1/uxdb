# UXDB实例

## ASP.NET驱动

### 准备

- 安装Visual Studio官网下载并安装Visual Studio 2013。
- 在UXDB官网下载 uxdb-nuxsql-2.1.zip，解压得到Nuxsql.dll和Mono.Security.dll

### 新建网站

点击文件-->新建网站-->ASP.NET空网站（自定义路径和项目名称，如：D: \vs2010\WebSite1）。点击确定。

### 网站中添加库类

1. 右键项目名称，添加ASP.NET文件夹-->Bin；
2. 将Nuxsql.dll和Mono.Security.dll拷贝到项目的bin目录文件夹下（Nuxsql.dll和 Mono.Security.dll在uxdb安装包中）；
3. 在项目中，右键Bin-->添加现有项，选择Nuxsql.dll和Mono.Security.dll，点击添加；
4. 在项目中，右键Bin-->添加引用,选择浏览页签，选中Nuxsql.dll和Mono.Security.dll，点击确定。

### 添加UXDB.cs接口类

1.	右键项目名称，添加ASP.NET文件夹-->App_Code；
2.	App_Code下新建UXDB.cs，右键App_Code-->添加新项，选择Visual C#的类，名称输入 UXDB.cs，点击添加。

UXDB.cs的代码如下：

```csharp
using System;
using System.Collections.Generic; using System.Linq; using System.Text; using System.Threading.Tasks; //using System.Windows.Forms; using System.Data; using Nuxsql; public class UXDB
{
    DataSet DS;     
    bool ECode;     
    string ErrString;
    NuxsqlConnection Conn = new NuxsqlConnection();
    public UXDB(string ServerName, string ServerPort, string DBName, string UserName, string
 Pwd)
    {
        ECode = false;
        Conn.ConnectionString = "Server=" + ServerName + ";Port=" + ServerPort + ";User Id=" +  UserName + ";Password=" + Pwd + ";Database=" + DBName;         
        try
        {
            Conn.Open();
        }
        catch (Exception e)
        {
            ECode = true;
            ErrString = e.Message;
        }
    }
    
    public DataSet GetRecordSet(string sql)
    {
        NuxsqlCommand sqlCmd = new NuxsqlCommand();
        sqlCmd.Connection = Conn;        
        sqlCmd.CommandText = sql;
        try
        {
            NuxsqlDataAdapter adp = new NuxsqlDataAdapter(sqlCmd);             
            DS = new DataSet();             
            adp.Fill(DS);
        }
        catch (Exception e)
        {
            ErrString = e.Message;            
            ECode = true;             
            return null;
        }         
        return DS;
    }
    
    public int ExecuteSQLScalar(string Sqls)
    {
        string s;
        NuxsqlCommand sqlCmd = new NuxsqlCommand();
        sqlCmd.Connection = Conn;         
        sqlCmd.CommandText = Sqls;         
        sqlCmd.CommandType = CommandType.Text;         
        try
        {
            s = sqlCmd.ExecuteScalar().ToString();
        }
        catch (Exception e)
        {
            ErrString = e.Message;             
            ECode = true;             
            return -1;
        }
        return (int.Parse(s));
    }
    
    public string ExecuteSQLScalarTOstring(string Sqls)
    {         string s;
        NuxsqlCommand sqlCmd = new NuxsqlCommand();
        sqlCmd.Connection = Conn;         
        sqlCmd.CommandText = Sqls;         
        sqlCmd.CommandType = CommandType.Text;
        try
        {
            s = sqlCmd.ExecuteScalar().ToString();
        }
        catch (Exception e)
        {
            ErrString = e.Message;             
            ECode = true;             
            return "-1";         
        }         
     return s;
    }
    
    public string ExecuteSQLWithTrans(string Sqls)
    {   string s;
        NuxsqlTransaction myTrans;         
     	myTrans = Conn.BeginTransaction();         
     	NuxsqlCommand sqlCmd = new NuxsqlCommand();
        sqlCmd.Connection = Conn;         
     	sqlCmd.CommandText = Sqls;         
     	sqlCmd.CommandType = CommandType.Text;         
     	sqlCmd.Transaction = myTrans;         
     	sqlCmd.ExecuteNonQuery();         
        //Sqls="SELECT @@IDENTITY AS ID";          sqlCmd.CommandText = Sqls;
        try
        {
            s = sqlCmd.ExecuteScalar().ToString();
        }
        catch (Exception e)
        {
            ErrString = e.Message;             
            ECode = true;             
            myTrans.Commit();             
            return "";
        }
        myTrans.Commit();
        return (s);
    }
    
    public void ExecuteSQL(string Sqls)
    {
        NuxsqlCommand sqlCmd = new NuxsqlCommand();
        sqlCmd.Connection = Conn;         
        sqlCmd.CommandText = Sqls;         
        sqlCmd.CommandType = CommandType.Text;
        try
        {
            sqlCmd.ExecuteNonQuery();
        }
        catch (Exception e)
        {
            ErrString = e.Message;
            ECode = true;
        }
    }
    
    public NuxsqlDataReader DBDataReader(string Sqls)
    {
        NuxsqlCommand sqlCmd = new NuxsqlCommand();
        sqlCmd.Connection = Conn;         
        sqlCmd.CommandText = Sqls;         
        sqlCmd.CommandType = CommandType.Text;         
        try
        {
            return sqlCmd.ExecuteReader(CommandBehavior.CloseConnection);
        }
        catch (Exception e)
        {
            ErrString = e.Message;             
            ECode = true;             
            return null;
        }
    }
    
    public void DBClose()
    {         
        try
        {
            Conn.Close();
        }
        	catch (Exception e)
        {
            ErrString = e.Message;
            ECode = true;
        }
    }
    public bool ErrorCode()
    {
        return ECode;
    }
    public string ErrMessage()
    {
        return ErrString;
    }
    ~UXDB()
    {
    }
}
```

### 创建测试窗体

右键项目名称，添加新项-->Visual C#->Web窗体，点击添加。

### 创建测试窗体

右键项目名称，添加新项-->Visual C#->Web窗体，点击添加。

Default.aspx.cs代码如下：

```csharp
using System;
using System.Collections.Generic; using System.Linq; using System.Web; using System.Web.UI; using System.Web.UI.WebControls;
using Nuxsql;
public partial class _Default : System.Web.UI.Page
{
    protected void Page_Load(object sender, EventArgs e)
    {
        //NuxDB myDb = new NuxDB("localhost", "5432", "uxdb", "uxdb", "123456");         UXDB myDb = new UXDB("192.168.138.131", "5432", "test", "uxdb", "123456");         string retStr = "";
        string testSql = "select * from student";
        NuxsqlDataReader reader = myDb.DBDataReader(testSql);
      	  // 判断数据是否读到尾.
        while (reader.Read())
        {
            //控制台输入
            string temp = String.Format("{0},{1},{2},{3}", reader[0], reader[1], reader[2], reader[3]); 
	        }	 
        System.Diagnostics.Debug.WriteLine("student\n" + retStr);
        Console.WriteLine(retStr);         // 关闭 reader 对象.
        reader.Close();
    }
}
```

### 运行结果

•	运行结果

右键项目名称-->生成网站，启动调试（F5），查看即时窗口输出。
运行结果显示时asp.net连接使用uxdb成功，显示student表的数据与uxdb数据库中查看结果一致。