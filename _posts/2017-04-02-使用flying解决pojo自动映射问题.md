---
layout: post
title: 使用flying解决pojo自动映射问题
description: 本节内容向您讲解如何使用AutoMapperInterceptor拦截器来实现pojo的自动映射。
category: blog
---

## Hello World

上一篇文章中我们介绍了flying的基本情况，在展示第一个demo之前还需要做一些额外的工作，即描述你想让mybatis托管的数据的表结构。

无论是否使用flying插件，对于每一个由mybatis托管的表，都要有一个<i>pojo_mapper.xml</i>来告诉mybatis这个表的基本信息。在以往这个配置文件可能会因为sql片段而变得非常复杂，但加入flying插件后，这个配置文件中将不需要sql片段，变得精简而统一。下面是一个有代表性的配置文件account.xml：
``` 
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="myPackage.AccountMapper">
    <cache />
    <select id="select" resultMap="result">#{id}</select>
    <select id="selectOne" resultMap="result">#{cacheKey}</select>
    <resultMap id="result" type="Account" autoMapping="true">
        <id property="id" column="account_id" />
    </resultMap>
</mapper>
``` 
在以上配置文件中，我们描述了一个接口<i>myPackage.AccountMapper</i>，一个方法<i>select</i>，一个方法<i>selectOne</i>，一个对象实体<i>Account</i>，以及表结构<i>resultMap</i>。在<i>resultMap</i>中由于设置了`autoMapping="true"`，我们只需要写出主键（以及外键，在稍后的章节会讲到），其余字段mybatis会自动感知。

<i>myPackage.AccountMapper</i>接口是mybatis本身需要的，里面的内容和此配置文件中定义的方法相对应。如果你有使用mybatis的经验你就能猜到，<i>AccountMapper.java</i>中的内容是：
```
package myPackage;
public interface AccountMapper {
    public Account select(Object id);
    public Account selectOne(Account t);
}
```
到目前为止一切都和不使用flying时一模一样，你可能唯一奇怪的一点就是account.xml中的select方法描述中的<i>#{id}</i>，selectOne方法描述中的<i>#{cacheKey}</i>，以及具体的sql在哪里。不要急，马上在对象实体<i>Account</i>中我们就会认识到flying的存在。<i>Account.java</i>的代码如下：
```
package myPackage;
import org.apache.ibatis.type.JdbcType;
import indi.mybatis.flying.annotations.FieldMapperAnnotation;
import indi.mybatis.flying.annotations.TableMapperAnnotation;
    
@TableMapperAnnotation(tableName = "account")
public class Account {
    @FieldMapperAnnotation(dbFieldName = "account_id", jdbcType = JdbcType.INTEGER, isUniqueKey = true)
    private Integer id;
	    
    @FieldMapperAnnotation(dbFieldName = "name", jdbcType = JdbcType.VARCHAR)
    private java.lang.String name;
	    
    public Integer getId() {
	    return id;
    }
    public void setId(Integer id) {
		this.id = id;
    }
    public String getName() {
		return name;
    }
    public void setName(String name) {
		this.name = name;
    }
}
```    
可见，和普通的pojo相比，<i>Account.java</i>只是多了以下3行注解而已：
```
    @TableMapperAnnotation(tableName = "account")
    @FieldMapperAnnotation(dbFieldName = "id", jdbcType = JdbcType.INTEGER, isUniqueKey = true)
    @FieldMapperAnnotation(dbFieldName = "name", jdbcType = JdbcType.VARCHAR) 
```
下面我们分别来解释它们的含义。

第1行@TableMapperAnnotation只能放在类定义之上，它声明这个类是一个表，它的属性tableName描述了这个表在数据库中的名字。

第2行@FieldMapperAnnotation只能放在变量定义之上，它声明这个变量是一个字段，它的属性dbFieldName描述了在数据库中这个字段的名称，属性jdbcType描述了在数据库中这个字段的类型，属性isUniqueKey为true描述了这个字段是主键。

第3行@FieldMapperAnnotation与第二行相同，它描述了另一个字段name，值得注意的是这个字段的类型是VARCHAR并且不是主键。

以上3个注解直白的描述了表account的数据结构，然后我们就可以使用<i>AccountService</i>非常方便的操纵数据库的读取了。（<i>AccountService</i>是<i>AccountMapper</i>的实现类，单独使用或在spring中使用都有多种方法进行配置，本文档不对此进行赘述）

使用以下代码，可以查询id为1的账户：
```
Account account = accountService.select(1);
```     
使用以下代码，可以查询name为andy的<b>1</b>条账户数据：
```
Account accountCondition = new Account();
accountCondition.setName("andy");
Account account = accountService.selectOne(accountCondition);
```
与以往的方式相比，这种方式是不是变得优雅了很多？关于select和selectOne之间的区别，我们在后面的章节会讲到。

## insert & delete

在最基本的select之后，我们再看新增功能。但在此之前，需要先在<i>account.xml</i>中增加以下内容：
```
<insert id="insert" useGeneratedKeys="true" keyProperty="id" />
```
上面的`useGeneratedKeys="true"`表示主键自增，如果你不使用主键自增策略此处可以省略，上面的语句和一般mybatis映射文件的区别在于没有具体sql语句。

同样在<i>AccountMapper.java</i>中我们需要加入
```
public void insert(Account t);
```
就可以了。例如使用以下代码，可以增加1条name为<i>bob</i>的账户数据（由于我们配置了主键自增，新增数据时不需要指定主键）：
```
    Account newAccount = new Account();
    newAccount.setName("bob");
    accountService.insert(newAccount);
```
然后我们再看删除功能。先在<i>account.xml</i>中增加以下内容：
```
<delete id="delete" />
```
然后在<i>AccountMapper.java</i>中加入
```
public int delete(Account t);
```
就可以了。例如使用以下代码，可以删掉id与<i>accountToDelete</i>的id一致的数据。
```
accountService.delete(accountToDelete);
```
delete方法的返回值代表执行sql后产生影响的条数，一般来说，返回值为0表示sql执行后没有效果，返回值为1表示sql执行成功，在代码中可以通过判断delete方法的返回值来实现更复杂的事务逻辑。

## update & updatePersistent

接下来我们看看更新功能，这里我们要介绍两个方法：update（更新）和updatePersistent（完全更新）。首先，在<i>account.xml</i>中增加以下内容：
```
<update id="update" />
<update id="updatePersistent" />
```
上面的语句和一般mybatis映射文件的区别在于没有具体sql语句。

然后在<i>AccountMapper.java</i>中加入
```
public int update(Account t);
public int updatePersistent(Account t);
```
就可以了。例如使用以下代码，可以将<i>accountToUpdate</i>的name更新为duke。
```
accountToUpdate.setName("duke");
accountService.update(accountToUpdate);
```
update和updatePersistent方法的返回值代表执行sql后产生影响的条数，一般来说，返回值为0表示sql执行后没有效果，返回值为1表示sql执行成功，在代码中可以通过判断update和updatePersistent方法的返回值来实现更复杂的事务逻辑。

下面我们来说明update和updatePersistent和关系。如果我们执行
```
accountToUpdate.setName(null);
accountService.update(accountToUpdate);
```
实际上数据库中这条数据的name字段不会改变，因为flying对为null的属性有保护措施。这在大多数情况下都是方便的，但如果我们真的需要在数据库中将这条数据的name字段设为null，updatePersistent就派上了用场。我们可以执行
```
accountToUpdate.setName(null);
accountService.updatePersistent(accountToUpdate);
```
这样数据库中这条数据的name字段就会变为null。可见`updatePersistent`会把pojo中所有的属性都更新到数据库中，而`update`只更新不为null的属性。在实际使用`updatePersistent`时，需要特别小心慎重，因为你的pojo中当时为null的属性有可能比你想象的多。

## selectAll & count

在之前学习select和selectOne时，细心的你可能已经发现，这两个方法要完成的工作似乎是相同的。的确select和selectOne都返回<b>1</b>个绑定了数据的pojo，但它们接受的参数不同：select接受主键参数；selectOne接受pojo参数，这个pojo中的所有被`@FieldMapperAnnotation`标记过的属性都会作为“相等”条件传递到sql语句中。之所以要这么设计，是因为我们有时会需要按照一组条件返回多条数据或者数量，即selectAll方法与count方法，这个时候以pojo作为入参最为合适。为了更清晰的讲述，我们先给给<i>Account.java</i>再增加一个属性address：
```
@FieldMapperAnnotation(dbFieldName = "address", jdbcType = JdbcType.VARCHAR)
private java.lang.String address;
//相关的getter和setter方法请自行补充
```
然后我们在<i>account.xml</i>中增加以下内容：
```
<select id="selectAll" resultMap="result">#{cacheKey}</select>
<select id="count" resultType="int">#{cacheKey}</select>
```
再在<i>AccountMapper.java</i>中加入
```
public Collection<Account> selectAll(Account t);
public int count(Account t);
```
就可以了。例如使用以下代码，可以查询所有address为“beijing”的数据和数量：
```
Account condition = new Account();
condition.setAddress("beijing");
Collection<Account> accountCollection = accountService.selectAll(condition);
int accountNumber = accountService.count(condition);
```
（当然一般来说执行selectAll后就不需要执行count了，我们取结果集的size即可，但如果我们只关心数量不关心具体数据集时，执行count比执行selectAll更节省时间）

如果我们想查询所有address为“shanghai”<b>同时</b>name为“ella”的账户，则执行以下代码：
```
Account condition = new Account();
condition.setAddress("shanghai");
condition.setName("ella");
Collection<Account> accountCollection = accountService.selectAll(condition);
```
如果我们知道address为“shanghai”<b>同时</b>name为“ella”的账户只有一个，并想直接返回这个数据绑定的pojo，可以执行： 
```
Account account = accountService.selectOne(condition);
```
由此可见selectOne可以称作是selectAll的特殊形式，它只会返回一个pojo而不是pojo的集合。如果确实有多条数据符合给定的codition，也只会返回查询结果中排在最前面的数据，这一点用户在使用selectOne时需要了解。无论如何，在合适的地方使用selectOne代替selectAll，会让你的程序获得极大便利。

## foreign key

一般来说我们的pojo都是业务相关的，而这些相关性归纳起来无外乎一对一、一对多和多对多。其中一对一是一对多的特殊形式，多对多本质上是由两个一对多组成，所以我们只需要着重解决一对多关系，而flying完全就是为此而生。

首先我们定义一个新的pojo：角色（role）。角色和账户是一对多关系，即一个账户只能拥有一个角色，一个角色可以被多个账户拥有。为此我们要新建<i>role.xml</i>、<i>RoleMapper.java</i>以及<i>Role.java</i>。<i>role.xml</i>如下：
``` 
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="myPackage.RoleMapper">
    <cache />
	<select id="select" resultMap="result">#{id}</select>
	<select id="selectOne" resultMap="result">#{cacheKey}</select>
	<select id="selectAll" resultMap="result">#{cacheKey}</select>
	<select id="count" resultType="int">#{cacheKey}</select>
	<insert id="insert" useGeneratedKeys="true" keyProperty="id" />
	<update id="update" />
	<update id="updatePersistent" />
	<resultMap id="result" type="Role" autoMapping="true">
        <id property="id" column="role_id" />
    </resultMap>
</mapper>
```
 <i>RoleMapper.java</i>如下：
``` 
package myPackage;
public interface RoleMapper {
    public Role select(Object id);
	public Role selectOne(Role t);
	public Collection<Role> selectAll(Role t);
	public void insert(Role t);
	public int update(Role t);
	public int updatePersistent(Role t);
	public int delete(Role t);
	public int count(Role t);
}
```
<i>Role.java</i>如下：
```  
package myPackage;
import org.apache.ibatis.type.JdbcType;
import indi.mybatis.flying.annotations.FieldMapperAnnotation;
import indi.mybatis.flying.annotations.TableMapperAnnotation;
    
@TableMapperAnnotation(tableName = "role")
public class Role {
    @FieldMapperAnnotation(dbFieldName = "role_id", jdbcType = JdbcType.INTEGER, isUniqueKey = true)
    private Integer id;
	    
	@FieldMapperAnnotation(dbFieldName = "role_name", jdbcType = JdbcType.VARCHAR)
	private String roleName;
	//相关的getter和setter方法请自行补充
}
```
然后在<i>Account.java</i>中，加入以下内容：
```   
@FieldMapperAnnotation(dbFieldName = "fk_role_id", jdbcType = JdbcType.INTEGER, dbAssociationUniqueKey = "role_id")
private Role role;   
//相关的getter和setter方法请自行补充
```
以上代码中，<b>dbFieldName</b>的值为数据库表account中指向表role的外键名，<b>jdbcType</b>的值为这个外键的类型，<b>dbAssociationUniqueKey</b>的值为此外键对应的另一表的主键的名称，写出以上信息后，flying在代码层面已经完全理解了数据结构。

最后在<i>account.xml</i>的resultMap元素中，加入以下内容
```   
<association property="role" javaType="Role" select="myPackage.RoleMapper.select" column="fk_role_id" /> 
```
写出以上信息后，flying在配置文件层面已经完全理解了数据结构。

最后完整版的<i>account.xml</i>如下：
``` 
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="myPackage.AccountMapper">
    <cache />
	<select id="select" resultMap="result">#{id}</select>
	<select id="selectOne" resultMap="result">#{cacheKey}</select>
	<select id="selectAll" resultMap="result">#{cacheKey}</select>
	<select id="count" resultType="int">#{cacheKey}</select>
	<insert id="insert" useGeneratedKeys="true" keyProperty="id" />
	<update id="update" />
	<update id="updatePersistent" />
	<resultMap id="result" type="Account" autoMapping="true">
        <id property="id" column="account_id" />
	    <association property="role" javaType="Role" select="myPackage.RoleMapper.select" column="fk_role_id" />
    </resultMap>
</mapper>
```
在写完以上代码后，我们看看flying能做到什么。首先多对一关系中的<b>一</b>（也即父对象），是可以在多对一关系中的<b>多</b>（也即子对象）查询时自动查询的，为了说明接下来的例子，我们先以dataset的方式定义一个数据集
```
<dataset>
        <account account_id="1" fk_role_id="10" address="beijing" name="frank" />
        <account account_id="2" fk_role_id="11" address="tianjin" name="gale" />
        <account account_id="3" fk_role_id="11" address="beijing" name="hank" />
	<role role_id="10" role_name="user" />
	<role role_id="11" role_name="super_user" />
</dataset>
``` 
我们使用这个数据集进行测试，当我们输入以下代码时：
```
Account account1 = accountService.select(1);
//此时account1的role属性也已经加载了真实数据
Role role1 = account1.getRole();
//role1.getId()为10，role1.getRoleName()为"user"
```
这种传递是可以迭代的，即如果Role自己也有父对象，则Role的父对象也会一并加载，只要它的代码和配置正确。

不仅如此，我们可以在入参pojo中加入父对象，比如下面的代码查询的是角色名为"super_user"的所有帐户：
```
Role roleCondition = new Role();
roleCondition.setRoleName("super_user");
Account accountCondition = new Account();
accountCondition.setRole(roleCondition);
Collection<Account> accounts = accountService.selectAll(accountCondition);
//accounts.seiz()为2，里面包含的对象的account_id是2和3

//我们再给入参pojo加一个address限制
accountCondition.setAddress("beijing");
Collection<Account> accounts2 = accountService.selectAll(accountCondition);
//accounts.size()为1，里面包含的对象的account_id是3，这说明account的条件和父对象role的条件同时都生效了
```
这个特性在`selectOne`、`count`中同样存在

最后，父对象同样可以参与子对象的`insert`、`update`、`updatePersistent`，代码如下：
```
Account newAccount = new Account();
//我们新建一个姓名为iris，角色名称为"user"的账号
newAccount.setName("iris");
Role role1 = roleService.select(10);
//角色名称为"user"的数据的role_id是10，现在role1已经加载了它
newAccount.setRole(role1);
accountService.insert(newAccount);
//一个姓名为iris，角色名称为"user"的账号建立完成

//我们用update方法将iris的角色变为"super_user"
Role role2 = roleService.select(11);
//角色名称为"super_user"的数据的role_id是11，现在role1已经加载了它
newAccount.setRole(role2);
accountService.update(newAccount);
//现在newAccount.getRole().getId()为11，newAccount.getRole().getRoleName为"super_user"

//我们用updatePersistent方法将iris的角色变为null，即不再关联
newAccount.setRole(null);
accountService.updatePersistent(newAccount);
//现在newAccount.getRole()为null，在数据库中也不再有关联，注意在这里update方法起不到这种效果
```

## complex condition

之前我们展示的例子中，条件只有“相等”一种，但在实际情况中我们会遇到各种各样的条件：大于、不等于、like、in、is not null 等等。这些情况 flying 也是能够处理的，但首先我们要引入一个“条件对象”的概念。条件对象是实体对象的子类，但它只为查询而存在，它拥有实体对象的全部属性，同时它还有一些专为查询服务的属性。例如下面是 Account 对象的条件对象 AccountCondition 的代码：
```
package myPackage;
import indi.mybatis.flying.annotations.ConditionMapperAnnotation;
import indi.mybatis.flying.annotations.QueryMapperAnnotation;
import indi.mybatis.flying.models.Conditionable;
import indi.mybatis.flying.statics.ConditionType;
@QueryMapperAnnotation(tableName = "account")
public class AccountCondition extends Account implements Conditionable {
        @ConditionMapperAnnotation(dbFieldName = "name", conditionType = ConditionType.Like)
        //用作 name 全匹配的值
	private String nameLike;
	
	@ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.HeadLike)
	//用作 address 开头匹配的值
	private String addressHeadLike;
	
	@ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.TailLike)
	//用作 address 结尾匹配的值
	private String addressTailLike;
	
	@ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.MultiLikeAND)
	//用作 address 需要同时匹配的若干个值的集合（类型只能为List）
	private List<String> addressMultiLikeAND;
	
	@ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.MultiLikeOR)
	//用作 address 需要至少匹配之一的若干个值的集合（类型只能为List）
	private List<String> addressMultiLikeOR;
	
	@ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.In)
	//用作 address 可能等于的若干个值的集合（类型可为任意Collection）
	private Collection<String> addressIn;
	
	@ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.NotIn)
	//用作 address 不可能等于的若干个值的集合（类型可为任意Collection）
	private Collection<String> addressNotIn;
	
	@ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.NullOrNot)
	//用作 address 是否为 null 的判断（类型只能为Boolean）
	private Boolean addressIsNull;
	
	//相关的getter和setter方法请自行补充
}

```
