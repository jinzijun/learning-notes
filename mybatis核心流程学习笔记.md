[原视频链接](https://www.bilibili.com/video/av36939017)
# mybatis核心流程三大阶段
## 初始化阶段
读取XML配置文件和注解中的配置信息，创建配置对象，并完成各个模块的初始化的工作。
## 代理阶段
1. 创建sqlsession响应请求
2. 动态代理封装iBatis的编程模型
3. 使用mapper接口访问数据库

## 数据读写阶段
遵循JDBC规范，通过sqlsession完成SQL的解析，参数的映射、SQL的执行、结果的解析过程。

# mybatis的初始化
XMLConfigBuilder用于加载mybatis-config.xml的配置信息。
XMLMapperBuilder用于加载mapper.java和mapper.xml。
XMLStatementBuilder用于加载sql语句。
最终所有的配置信息会保存在Configuration对象中。

# Configuration类
| 变量 | 存储内容 |
| ------ | ------ |
| mapperRegistry | 注册接口的动态代理对象 |
| loadedResources | 填充xml文件资源 |
| resultMaps | 填充resultMap |
| sqlFragments | 填充sql元素 |
| mappedStatement | 填充MappedStatement |
| keyGenerators | 填充keyGenerator |

## 编程模型
jdbc
```
//注册JDBC驱动程序
Class.forName("com.mysql.jdbc.Driver");
final String USER = "root";
final String PASS = "jinzijun";
final String DB_URL = "jdbc:mysql://localhost:3306/test";
Connection conn = DriverManager.getConnection(DB_URL, USER, PASS);
Statement statement = conn.createStatement();
String sql = "select id,name,age from person";
ResultSet resultSet = statement.executeQuery(sql);
while (resultSet.next()){
    int id = resultSet.getInt("id");
    String name = resultSet.getString("name");
    int age = resultSet.getInt("age");
    System.out.println(String.format("%d,%s,%d",id,name,age));
}
```
iBatis
```
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
SqlSession sqlSession = sqlSessionFactory.openSession();
Person person = (Person) sqlSession.selectOne("com.practice.jinzijun.db.mybatis.PersonMapper.getById", 1);
```
MyBatis
```
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
SqlSession sqlSession = sqlSessionFactory.openSession();
// 生成动态代理对象
PersonMapper mapper = sqlSession.getMapper(PersonMapper.class);
Person person = mapper.getById(1);
```
Mybatis通过配置文件解读和动态代理，封装iBatis的编程模型，提供了面向接口编程的方式。
1. 找到sqlSession中对应的方法执行。
2. 找到命名空间和方法名
3. 传递参数
翻译过程
1. MapperMethod.SqlCommand.type和MapperMethod.MethodSignature.returnType -》找到sqlSession中对应的方法执行。
2. MapperMethod.SqlCommand.name -> 找到命名空间和方法名
3. MapperMethod.MethodSignature.paramNameResolver -> 传递参数
翻译过程完成后，就会通过iBatis的方式进行查询。

# SqlSession查询接口的嵌套关系
所有查询接口最终都会通过Executor.query(MappedStatement, Object parameter, RowBounds, ResultHandler)来执行。Executor查询的过程，最终会转化为通过jdbc去执行。

# Executor的三个重要组件
1. StatementHandler：它的作用是使用数据库的Statement或PrepareStatement执行操作，启承上启下作用；
2. ParameterHandler：对预编译的SQL语句进行参数设置
3. ResultSetHandler：对数据库返回的结果集进行封装，返回用户指定的实体类型。
