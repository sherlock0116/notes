# 【SpringBoot】Q & A

最近开始啃 Spring 全家桶, 准备把后端的工作也自己做了, 之前断断续续学过 Spring, 现在各个公司都开始或已经将各个产业线微服务化, 大趋势啊!



## 一、MybatisPlus

介绍啥的见百度

1. 实体类上的注解

   ```java
   @Data
   @Builder
   @TableName("dim_user")
   public class User {
   
   	@TableId(value = "user_id", type = IdType.NONE)
   	private String id;
   	@TableField("user_name")
   	private String name;
   	@TableField("user_age")
   	private int age;
   }
   ```

   

   - @TableName("table_name")

     类名绑定表, 如果类名和表名不一致, 是该注解标明绑定到哪一张表

   - @TableId(value = "user_id", type = IdType.NONE)

     mybatis 的默认使用的主键名是id，而数据库中表的 id 是 user_id，所以查询时会报错, 需要使用该注解的 value 属性指定表主键。

     type 属性指定表主键类型, 它有如下几种类型:

     - IdType.AUTO 数据库 ID 自增
     - IdType.INPUT 用户输入 ID
     - IdType.ID_WORKER 全局唯一 ID 
     - IdType.UUID 全局唯一 ID
     - IdType.NONE 该类型为未设置主键类型
     - IdType.ID_WORKER_STR 字符串全局唯一 ID 

2. @MapperScan

   之前是，直接在 Mapper 类上面添加注解 `@Mapper`，这种方式要求每一个 mapper 类都需要添加此注解，麻烦。

   通过使用 `@MapperScan` 可以指定要扫描的 Mapper 类的包的路径,比如:

   ```java
   // java code
   @SpringBootApplication
   @MapperScan("com.a.b.mapper")
   // 同时,使用@MapperScan注解多个包
   @MapperScan({"com.a.b.mapper"})
   public class Application {
   	public static void main(String[] args) {
    		SpringApplication.run(Application.class, args);
    	}
   }
   
   // scala code
   @SpringBootApplication
   // scala 中不管是单个包还是多个包, 一定要用 Array()
   @MapperScan(Array("com.a.b.mapper"))
   class Application { }
   object Application {
   	def main(args: Array[String]): Unit {
    		SpringApplication.run(classOf[Application], args)
    	}
   }
   ```

   如果如果 mapper 类没有在 Spring Boot 主程序可以扫描的包或者子包下面，可以使用如下方式进行配置

   ```java
   // java code
   @SpringBootApplication  
   @MapperScan({"com.a.*.mapper","org.b.*.mapper"})  
   public class App {  
    	public static void main(String[] args) {  
    		SpringApplication.run(App.class, args);  
    	}  
   }
   
   // scala code
   @SpringBootApplication
   @MapperScan(Array("com.a.*.mapper","org.b.*.mapper"))
   class Application { }
   object Application {
   	def main(args: Array[String]): Unit {
    		SpringApplication.run(classOf[Application], args)
    	}
   }
   ```





## 二、SpringBootTest 

```scala
@RunWith(classOf[SpringRunner])
@SpringBootTest(classes = Array(classOf[MybatisPlusApp]), webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class MybatisPlusAppTest {
	
	@Autowired
	var userMapper: UserMapper = _
	
	@Test
	def selectList(): Unit = {
		val users: Seq[User]= userMapper.selectList(null).asScala
		println(s"=====> users size: ${users.size}")
	}
}
```



@SpringBootTest

