# MyBatisPlus学习笔记

## 用法

### 实体类注解

- mybatis-plus为使用者封装了很多的注解，方便我们使用，我们首先看下实体类中有哪些注解。有如下的实体类：

```java
@TableName(value = "user")  
public class UserDO {  
  
    /**  
     * 主键  
     */  
    @TableId(value = "id", type = IdType.AUTO)  
    private Long id;  
  
    /**  
     * 昵称  
     */  
    @TableField("nickname")  
    private String nickname;  
  
    /**  
     * 真实姓名  
     */  
    private String realName;  
}
```

- @TableName 表名注解，用于标识实体类对应的表。
- @TableId 主键注解

其中IdType很重要：

|名称 | 描述 |
|-- | -----|
| AUTO | 数据库自增ID  |
|NONE|该类型为未设置主键类型(注解里等于跟随全局,全局里约等于 INPUT) 
|   INPUT     |  用户自己设置的ID                              |
|  ASSIGN_ID | 当用户传入为空时，自动分配类型为Number或String的主键（雪花算法）  | 
| ASSIGN_UUID | 当用户传入为空时，自动分配类型为String的主键  |


- @TableFiled 表字段标识

![[1660618795558.png]]

关于其他的属性，我不太推荐使用，用得越多，越容易蒙圈。可以通过wapper查询去设置。

### CRUD

mybatis-plus封装好了一条接口供我们直接调用。关于内部的具体方法，在使用时候自己体会吧，此处不列举了。

####  **Service层CRUD**

我们使用的时候，需要在自己定义的service接口当中继承**IService**接口, 同时要在我们的接口实现impl当中继承`ServiceImpl`，实现自己的接口：

```java
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;  
import com.wjbgn.user.entity.UserDO;  
import com.wjbgn.user.mapper.UserMapper;  
import com.wjbgn.user.service.IUserService;  
  
/**  
 * @description：用户接口实现  
 * @author：weirx  
 * @date：2022/1/17 15:03  
 * @version：3.0  
 */  
public class UserServiceImpl extends ServiceImpl<UserMapper, UserDO> implements IUserService {  
  
}
```

#### Mapper层的CRUD

mybatis-plus将常用的CRUD接口封装成了`BaseMapper`接口,我们只需要在自己的Mapper中继承它就可以了：

```java
/**  
 * @description：用户mapper  
 * @author：weirx  
 * @date：2022/1/17 14:55  
 * @version：3.0  
 */  
@Mapper  
public interface UserMapper extends BaseMapper<UserDO> {  
}
```

## 分页

使用分页话需要增加分页插件的配置：

```java
import com.baomidou.mybatisplus.annotation.DbType;  
import com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor;  
import com.baomidou.mybatisplus.extension.plugins.inner.PaginationInnerInterceptor;  
import org.mybatis.spring.annotation.MapperScan;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
  
@Configuration  
@MapperScan("com.wjbgn.*.mapper*")  
public class MybatisPlusConfig {  
  
    @Bean  
    public MybatisPlusInterceptor mybatisPlusInterceptor() {  
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();  
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));  
        return interceptor;  
    }  
  
}
```


如上配置后，我们直接使用分页方法就行。

## 逻辑删除配置

很多情况下我们的系统都需要逻辑删除，方便恢复查找误删除的数据。  

通过mybatis-plus可以通过全局配置的方式，而不需要再去手动处理。针对更新和查询操作有效，新增不做限制。

通常以我的习惯逻辑删除字段通常定义为**is_delete**,在实体类当中就是**isDelete**。那么在配置文件中就可以有如下的配置：

```yaml
mybatis-plus:  
  global-config:  
    db-config:  
      logic-delete-field: isDelete # 全局逻辑删除的实体字段名(since 3.3.0,配置后可以忽略不配置步骤2)  
      logic-delete-value: 1 # 逻辑已删除值(默认为 1)  
      logic-not-delete-value: 0 # 逻辑未删除值(默认为 0)
```

或者通过注解 **@TableLogic**

```java
@TableLogic  
private Integer isDelete;
```

## 通用枚举配置

相信后端的同学都经历过一个情况，比如`性别`这个字段，分别值和名称对应**1男、2女**,这个字段在数据库时是数值类型，而前端展示则是展示字符串的名称。有几种常见实现方案呢？

-   数据库查询sql通过case判断，返回名称，以前oracle经常这么做  
-   数据库返回的值，重新遍历赋值进去，这时候还需要判断这个值到底是男是女。
-   前端写死，返回1就是男，返回2就是女。

相信无论哪种方法都有其缺点，所以我们可以使用mybatis-plus提供的方式。我们在返回给前端时：

-   只需要在遍历时get这个枚举，直接赋值其名称，不需要再次判断。
-   直接返回给前端，让前端去取枚举的name

- 定义IEnum枚举接口:

```java
import com.baomidou.mybatisplus.annotation.IEnum;  
import com.fasterxml.jackson.annotation.JsonFormat;  
  
/**  
 * @description：性别枚举  
 * @author：weirx  
 * @date：2022/1/17 16:26  
 * @version：3.0  
 */  
@JsonFormat(shape = JsonFormat.Shape.OBJECT)  
public enum SexEnum implements IEnum<Integer> {  
    MAN(1, "男"),  
    WOMAN(2, "女");  
    private Integer code;  
    private String name;  
  
    SexEnum(Integer code, String name) {  
        this.code = code;  
        this.name = name;  
    }  
  
    @Override  
    public Integer getValue() {  
        return code;  
    }  
  
    public String getName() {  
        return name;  
    }  
  
}
```

@JsonFormat注解为了解决枚举类返回前端只展示构造器名称的问题。

- 实体类性别字段

```java
@TableName(value = "user")  
public class UserDO {  
  
    /**  
     * 主键  
     */  
    @TableId(value = "id", type = IdType.AUTO)  
    private Long id;  
  
    /**  
     * 昵称  
     */  
    @TableField(value = "nickname",condition = SqlCondition.EQUAL)  
    private String nickname;  
  
    /**  
     * 性别  
     */  
    @TableField(value = "sex")  
    private SexEnum sex;  
  
    /**  
     * 版本  
     */  
    @TableField(value = "version",update = "%s+1")  
    private Integer version;  
  
    /**  
     * 时间字段，自动添加  
     */  
    @TableField(value = "create_time",fill = FieldFill.INSERT)  
    private LocalDateTime createTime;  
}
```

- 配置文件扫描枚举

```yaml
mybatis-plus:  
  # 支持统配符 * 或者 ; 分割  
  typeEnumsPackage: com.wjbgn.*.enums
```

- 定义配置文件

```java
@Bean  
public MybatisPlusPropertiesCustomizer mybatisPlusPropertiesCustomizer() {  
    return properties -> {  
        GlobalConfig globalConfig = properties.getGlobalConfig();  
        globalConfig.setBanner(false);  
        MybatisConfiguration configuration = new MybatisConfiguration();  
        configuration.setDefaultEnumTypeHandler(MybatisEnumTypeHandler.class);  
        properties.setConfiguration(configuration);  
    };  
}
```

- 序列化枚举值为数据库值

例子是FastJson

  - 全局

```java
@Bean  
 public MybatisPlusPropertiesCustomizer mybatisPlusPropertiesCustomizer() {  
     // 序列化枚举值为数据库存储值  
     FastJsonConfig config = new FastJsonConfig();  
     config.setSerializerFeatures(SerializerFeature.WriteEnumUsingToString);  
  
     return properties -> {  
         GlobalConfig globalConfig = properties.getGlobalConfig();  
         globalConfig.setBanner(false);  
         MybatisConfiguration configuration = new MybatisConfiguration();  
         configuration.setDefaultEnumTypeHandler(MybatisEnumTypeHandler.class);  
         properties.setConfiguration(configuration);  
     };  
 }
```

   -  局部

```java
@JSONField(serialzeFeatures= SerializerFeature.WriteEnumUsingToString)  
 private SexEnum sex;
```


## 自动填充

还记得前面提到的实体类当中的注解**@TableFeild**吗？当中有个属性叫做fill，通过`FieldFill`设置属性，这个就是做自动填充用的。

```java
public enum FieldFill {  
    /**  
     * 默认不处理  
     */  
    DEFAULT,  
    /**  
     * 插入填充字段  
     */  
    INSERT,  
    /**  
     * 更新填充字段  
     */  
    UPDATE,  
    /**  
     * 插入和更新填充字段  
     */  
    INSERT_UPDATE  
}
```

但是这个直接是不能使用的，需要通过实现mybatis-plus提供的接口，增加如下配置：

```java
import com.baomidou.mybatisplus.core.handlers.MetaObjectHandler;  
import org.apache.ibatis.reflection.MetaObject;  
import org.springframework.stereotype.Component;  
  
import java.time.LocalDateTime;  
  
/**  
 * description: 启动自动填充功能  
  
 * @return:  
 * @author: weirx  
 * @time: 2022/1/17 17:00  
 */  
@Component  
public class MyMetaObjectHandler implements MetaObjectHandler {  
  
    @Override  
    public void insertFill(MetaObject metaObject) {  
        // 起始版本 3.3.0(推荐使用)  
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());  
    }  
  
    @Override  
    public void updateFill(MetaObject metaObject) {  
        // 起始版本 3.3.0(推荐)  
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());  
    }  
}
```

```java
/**  
 * 时间字段，自动添加  
 */  
@TableField(value = "create_time",fill = FieldFill.INSERT)  
private LocalDateTime createTime;
```

## 多数据源

前面提到过，配置文件当中配置了主从的方式，其实mybatis-plus还支持更多的方式：

- 多主多从

```yaml
spring:  
  datasource:  
    dynamic:  
      primary: master #设置默认的数据源或者数据源组,默认值即为master  
      strict: false #严格匹配数据源,默认false. true未匹配到指定数据源时抛异常,false使用默认数据源  
      datasource:  
        master_1:  
        master_2:  
        slave_1:  
        slave_2:  
        slave_3:
```

- 多种数据库

```yaml
spring:  
  datasource:  
    dynamic:  
      primary: mysql #设置默认的数据源或者数据源组,默认值即为master  
      strict: false #严格匹配数据源,默认false. true未匹配到指定数据源时抛异常,false使用默认数据源  
      datasource:  
        mysql:  
        oracle:  
        postgresql:  
        h2:  
        sqlserver:
```

- 混合配置

```yaml
spring:  
  datasource:  
    dynamic:  
      primary: master #设置默认的数据源或者数据源组,默认值即为master  
      strict: false #严格匹配数据源,默认false. true未匹配到指定数据源时抛异常,false使用默认数据源  
      datasource:  
        master_1:  
        slave_1:  
        slave_2:  
        oracle_1:  
        oracle_2:
```

- DS注解

可以注解在方法上或类上，**同时存在就近原则 【方法上注解】 优先于 【类上注解】**：

```java
@DS("slave_1")  
public class UserServiceImpl extends ServiceImpl<UserMapper, UserDO> implements IUserService {  
  
  
    @DS("salve_1")  
    @Override  
    public List<UserDO> getList() {  
        return this.getList();  
    }  
  
    @DS("master")  
    @Override  
    public int saveUser(UserDO userDO) {  
        boolean save = this.save(userDO);  
        if (save){  
            return 1;  
        }else{  
            return 0;  
        }  
    }  
}
```

## 测试以上功能

- 新建User表

```sql
CREATE TABLE `user` (  
  `id` int(11) NOT NULL AUTO_INCREMENT,  
  `nickname` varchar(255) NOT NULL COMMENT '昵称',  
  `sex` tinyint(1) NOT NULL COMMENT '性别，1男2女',  
  `create_time` datetime NOT NULL COMMENT '创建时间',  
  `is_delete` tinyint(1) NOT NULL DEFAULT '0' COMMENT '是否删除 1是，0否',  
  PRIMARY KEY (`id`)  
) ENGINE=InnoDB AUTO_INCREMENT=50 DEFAULT CHARSET=utf8mb4;
```

- 加入Controller

```java
/**  
 * @description：用户controller  
 * @author：weirx  
 * @date：2022/1/17 17:39  
 * @version：3.0  
 */  
@RestController  
@RequestMapping("/user")  
public class UserController {  
  
    @Autowired  
    private IUserService userService;  
  
    /**  
     * description: 新增  
  
     * @return: boolean  
     * @author: weirx  
     * @time: 2022/1/17 19:11  
     */  
    @RequestMapping("/save")  
    public boolean save() {  
        UserDO userDO = new UserDO();  
        userDO.setNickname("大漂亮");  
        userDO.setSex(SexEnum.MAN);  
  
        return userService.save(userDO);  
    }  
  
    /**  
     * description: 修改  
     * @param nickname  
     * @param id  
     * @return: boolean  
     * @author: weirx  
     * @time: 2022/1/17 19:11  
     */  
    @RequestMapping("/update")  
    public boolean update(@RequestParam String nickname,@RequestParam Long id) {  
        UserDO userDO = new UserDO();  
        userDO.setNickname(nickname);  
        userDO.setId(id);  
        return userService.updateById(userDO);  
    }  
  
    /**  
     * description: 删除  
     * @param id  
     * @return: boolean  
     * @author: weirx  
     * @time: 2022/1/17 19:11  
     */  
    @RequestMapping("/delete")  
    public boolean delete(@RequestParam Long id) {  
        UserDO userDO = new UserDO();  
        userDO.setId(id);  
        return userService.removeById(userDO);  
    }  
  
    /**  
     * description: 列表  
     * @return: java.util.List<com.wjbgn.user.entity.UserDO>  
     * @author: weirx  
     * @time: 2022/1/17 19:11  
     */  
    @RequestMapping("/list")  
    public List<UserDO> list() {  
        return userService.list();  
    }  
  
    /**  
     * description: 分页列表  
     * @param current  
     * @param size  
     * @return: com.baomidou.mybatisplus.extension.plugins.pagination.Page  
     * @author: weirx  
     * @time: 2022/1/17 19:11  
     */  
    @RequestMapping("/page")  
    public Page page(@RequestParam int current,@RequestParam int size) {  
        return userService.page(new Page<>(current,size), new QueryWrapper(new UserDO()));  
    }  
  
}
```

