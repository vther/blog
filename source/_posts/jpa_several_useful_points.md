JPA 不区分大小写查询

    // 不区分大小，so，查询的时候，都转成大写查~   不建议这样写   
   predicates.add(builder.or(
                    builder.equal(builder.upper(auditRoot.get("auditor").as(String.class)), operator.toUpperCase()),
                    builder.isNull(auditRoot.get("auditor").as(String.class))));
                    
*_bin: 表示的是binary case sensitive collation，也就是说是区分大小写的 
*_cs: case sensitive collation，区分大小写 
*_ci: case insensitive collation，不区分大小写 
ALTER TABLE t_mkt_auditlog CHANGE AUDITOR AUDITOR VARCHAR(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL

```java
  @Transactional
  public void save(){

      //New 状态
      Task t = new Task();
      t.setTaskName("task" + new Date().getTime());
      t.setCreateTime(new Date());

      //Managed状态
      em.persist(t); //实体类t已经有id t.getId();
      t.setTaskName("kkk");  //更新任务名称，这时，如果提交事务，则直接将kkk更新到数据库

      //Detached状态 事务提交或者调用em.clear都直接将实体任务状态变为Detached
      em.clear();
      t.setTaskName("kkk"); //更新数据不会更新到数据库

      //Removed状态
      em.remove(t);
  }
```

实际上，实体共有4种状态。
    new--新实体：实体由应用产生，和实体管理器没有任何联系，也没有唯一的标识符。
    managed--持久化实体：新实体和实体管理器产生关联（通过persist()、merge()等方法），在实体管理器中存在和被管理，标志是在实体管理器中有一个唯一的标识符。
    detached--分离的实体：实体有唯一的标识符，但它的标识符不被实体管理器管理。
    removed--删除的实体：实体被remove()方法删除，对应的记录将会在当前事务提交的时候从数据库中删除。
 
entityManager.detach(object)应该是Jpa2.0后出的方法，其义在于：将object设置为游离状态，不受持久化上下文的管理，这样当find出来的实体直接进行属性set方法赋新值，不会直接自动保存到数据库，且此方法后的游离状态的实体支持懒加载。


各种Listener：http://www.cnblogs.com/thlzhf/p/4249816.html






EntityManager的find()与getReference()的区别

http://blog.csdn.net/superdog007/article/details/38852369
相同点
  1）这两个方法都接受实体的class和代表实体主键的对象作为参数。由于它们使用了Java泛型方法，无需任何显示的类型转换即可获得特定类型的实体对象。其中，在primaryKey上面普遍使用了java5的autoboxing（自动装箱）的特性。
   2） 再者，就是两者都会在EntityManager关闭的情况下抛出IllegalStateException -if this EntityManager has been closed.在传入的第一个参数不是实体或者第二个参数不是一个有效的主键的情况下抛出IlegalArgumentException -if the first argument does not denote an entity type or the secondargument is not a valid type for that entity's primary key
  
不同点：
    find()返回指定OID的实体，如果这个实体存在于当前的persistencecontext中，那么返回值是被缓存的对象；否则会创建一个新的实体，并从数据库中加载相关的持久状态。如果数据库不存在指定的OID的记录，那么find()方法返回null。
    getReference()方法和find()相似。不同的是：如果缓存中没有指定的实体，EntityManager会创建一个新的实体，但是不会立即访问数据库来加载持久状态，而是在第一次访问某个属性的时候才加载。此外，getReference()方法不返回null，如果数据库找不到相应的实体，这个方法会抛出javax.persistence.EntityNotFoundException。
EntityNotFoundException -if the entity state cannot be accessed
某些场合下使用getReference()方法可以避免从数据库加载持久状态的性能开销。
 
   这里要着重提出的是两句话：
   如果缓存中没有指定的实体，EntityManager会创建一个新的实体，但是不会立即访问数据库来加载持久状态，而是在第一次访问某个属性的时候才加载。
   比如，em.find()返回的实体，我们就可以对它进行各种操作，而若对em.getReference()返回的实体，由于不会立即访问数据库来加载持久状态，对它进行的操作很可能就会出现Exception，比如在对它返回的实体做getter操作时，由于EntityManager对此采用延时加载，就会抛出org.hibernate.lazyinitializationexceptioncould not initialize proxy no session
   因此将一个新的实体传递给事务的时候通常使用find()方法，而当不连接数据库，不使用getter方法，即使用setter方法改变状态时才使用getReference（）方法。（这是由于getReference返回是一个Proxy实体，即没有加载持久状态）
 
   某些场合下使用getReference()方法可以避免从数据库加载持久状态的性能开销。
    这也完全是由于getReference返回是一个Proxy实体.
    比如一个简单的update操作，先使用find()获取实体，而后使用实体的setter方法；或者是getReference()方法，而后使用实体的setter方法。
    对于前者JPA调用的SQL：select****，而后才是update ****
    对于后者：仅为update*****
 
又如：
操作                                                                            执行的SQL
em.remove(em.getReference(Person.class,1))         deletefrom Person where personid = 1
em.remove(em.find(Person.class,1))                 select* from Person where personid =1
                                                   deletefrom Person where personid =1
 
   由此可以看出，find()做了一次select的操作，而getReference并没有做有关数据库的操作，而是返回一个代理，这样它就减少了连接数据库和从数据库加载持久状态的开销。




设置Flush刷新模式setFlushMode()

http://blog.csdn.net/superdog007/article/details/38852399
是否flush	find	getReference	其他查询	事务提交
AUTO	×	×	√	√
COMMIT 	×	×	×	√
 设置Flush刷新模式setFlushMode()
上面的flush()函数是手动调用的，如果不手动调用，则只能依赖于容器的自动刷新。在默认情况下容器是自动刷新的，这是因为它对应了刷新了的AUTO值：
public enum FlushModeType {  
    AUTO,  
    COMMIT  
} 
我们可以调用下面的方法改变刷新模式：
em.setFlushMode(FlushModeType.COMMIT); 
这两种模式的区别如下。
AUTO：刷新在查询语句执行前（除了find()和getreference()查询）或事务提交时才发生，适用于在大量更新数据的过程中没有任何查询语句（除了find()和getreference()查询）时执行。
COMMIT：刷新只有在事务提交时才发生，适用于在大量更新数据的过程中存在查询语句（除了find()和getreference()查询）时执行。
这两种模式的区别体现在数据库底层SQL的执行上，即JDBC驱动跟数据库交互的次数。COMMIT模式使更新只在一次网络交互中完成，而AUTO模式可能需要多次交互，它触发了多少次Flush就产生了多少次网络交互。



CriteriaBuilderbuilder=entityManager.getCriteriaBuilder();
CriteriaQuery<Employee>query=builder.createQuery(Employee.class);
Root<Employee>employee=query.from(Employee.class);
Joinaddress=employee.join("projects",JoinType.LEFT);

@Cache(expiry=0,alwaysRefresh=true,isolation=CacheIsolationType.ISOLATED)
@Table(name="t_employee")
@Entity
publicclassEmployee{
	@Id
	privateStringid;
	@OneToMany(targetEntity=Project.class,cascade=CascadeType.ALL,fetch=FetchType.EAGER)
	@JoinColumn(name="employeeId")
	privateList<Project>projects;
}
SELECT t1.ID FROM t_employee t1 LEFT OUTER JOIN t_project t0 ON (t0.EMPLOYEEID = t1.ID)
JPA 默认内连接，即JoinType为Inner对应SQL为SELECT t1.ID FROM t_project t0, t_employee t1 WHERE (t0.EMPLOYEEID = t1.ID)


@Cache(expiry = 0, alwaysRefresh = true, isolation = CacheIsolationType.ISOLATED)
@Table(name = "t")
@Entity

