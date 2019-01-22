# Spark SQL基础

[[北风网 Spark 2.0从入门到精通] (278讲)](https://www.bilibili.com/video/av19995678/?p=101&t=612)

* Spark SQL特点

```markdown
1. 支持多种数据源: Hive,RDD,Parquet,JSON,JDBC
2. 多种性能优化技术: in-memory columnar storage, byte-code generation, cost model动态评估等
3. 组件扩展性: 对于SQL的讲法解析器,分析器以及优化器,用户都可以自己开发,并且动态扩展

```

## Spark SQL 开发步骤 

* 创建SQLContext /HiveContext(官方推荐)对象

```java
 SparkConf sparkConf = new SparkConf()
                .setAppName("...")
                .setMaster("local");

        JavaSparkContext jsc = new JavaSparkContext(sparkConf);

        SQLContext sqlContext = new SQLContext(jsc);
```

## 使用Json文件创建DataFrame DataFrameCreate.java

```java
SparkConf sparkConf = new SparkConf()
                .setAppName("DataFrameCreate")
                .setMaster("local");

        JavaSparkContext jsc = new JavaSparkContext(sparkConf);

        SQLContext sqlContext = new SQLContext(jsc);


        DataFrame df =  sqlContext.read().json("/home/sotowang/user/aur/ide/idea/idea-IU-182.3684.101/workspace/SparkSQLProject/src/resources/students.json");

        df.show();
```

### DataFrame 常用操作

打印DataFrame中所有的数据

```java
df.show();
```

打印DataFrame的元数据 (Schema)

```java
df.printSchema();

```

查询某列所有数据

```java
df.select("name").show();

```

查询某几列所有数据,并对列进行计算

```java
df.select(df.col("name"),df.col("age").plus(1)).show();

```

根据某一列的值进行过滤

```java
df.filter(df.col("age").gt(18)).show();

```

根据某一列进行分组,然后进行聚合

```java
df.groupBy(df.col("age")).count().show();
```

## RDD转DataFrame

### 为什么要将RDD转DataFrame?

> 因为这样的话,我们可以直接针对HDFS等任何可以构建为RDD的数据,使用Spark SQL进行SQL查询,这个功能很强大

### Spark SQL 支持2种方式来将RDD转为DataFrame

```markdown
1. 使用反射来推断包含了特定数据类型的RDD元数据.这种基于反射的方式,代码比较简洁,当你已经知道你的RDD的元素时,是一种不错的方式

2. 通过编程接口来创建DataFrame,你可以在程序运行时动态构建一份元数据,然后将其应用到已经存在的RDD上,代码比较冗长,
    但如果在编写程序时,还不知道RDD的元数据,只有在程序运行时,才能动态得知其元数据,只能通过动态构建元数据的方式
```

#### 使用反射的方式将RDD转换为Dataframe RDD2DataFrameReflection.java

```markdown
JavaRDD<String> lines = jsc.textFile("/home/sotowang/user/aur/ide/idea/idea-IU-182.3684.101/workspace/SparkSQLProject/src/resources/students.json");

        JavaRDD<Student> students = lines.map(new Function<String, Student>() {
            public Student call(String line) throws Exception {
                String[] lineSplited = line.split(",");
                Student stu = new Student();
                stu.setId(Integer.valueOf(lineSplited[0]));
                stu.setName(lineSplited[1]);
                stu.setAge(Integer.valueOf(lineSplited[2]));
                return stu;
            }
        });

        //使用反射方式将RDD转换为DataFrame
        //将Student.Class传入进去,其实就是用反射的方式来创建DataFrame
        //因为Student.class本身就是反射的一个应用
        //然后底层还得通过对Student Class进行反射,来获取其中的field
        DataFrame studentDF = sqlContext.createDataFrame(students, Student.class);

        //拿到DataFrame后,将其注册为一个临时表,然后针对其中的数据进行SQL语句
        studentDF.registerTempTable("students");


        //针对students临时表执行语句,查询年龄小于等于18岁的学生,就是teenager

        DataFrame teenagerDF = sqlContext.sql("select * from students where age <= 18");

        //将查询出的DataFrame,再次转换为RDD
        JavaRDD<Row> teenagerRDD = teenagerDF.javaRDD();

        //将RDD中的数据,进行映射,映射为student
        JavaRDD<Student> teenagerStudentRDD = teenagerRDD.map(new Function<Row, Student>() {
            public Student call(Row row) throws Exception {
                Student student = new Student();
                student.setAge(row.getInt(0));
                student.setName(row.getString(2));
                student.setId(row.getInt(1));
                return student;
            }
        });


        //将数据collect,打印出来
        List<Student> studentList = teenagerStudentRDD.collect();

        for (Student student : studentList) {
            System.out.println(student.toString());
        }
```

出现的问题:

> 1.Java Bean: Student.java 的序列化问题(不序列化会报错),public class Student implements Serializable

> 2.将RDD中的数据,进行映射,映射为student时,其RDD中student的属性顺序会乱(与文件中顺序不一致)






































































































































