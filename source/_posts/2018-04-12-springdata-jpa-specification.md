---
title: SpringData-JPA使用Specification实现动态查询
description: 在jpa中使用specification接口实现多条件动态查询
categories:
 - 后端
 - Spring
tags:
 - spring
 - jpa
---

最近项目技术选型db框架选择了使用JPA，刚开始时，使用jpa进行一些单表简单的查询非常轻松，大家写的不亦乐乎，后来在遇到多条件动态查询的业务场景时，发现现有的`JpaRepository`提供的方法和自己写`@Query`已经满足了不了需求，难不成要对所有的条件和字段进行判断，再写很多个dao方法？后面查到jpa提供了围绕`Specification`这个接口的一系列类，来用于实现动态查询。

首先，需要定义一个dao接口，这个接口除了继承`JpaRepository`之外，还需要继承`JpaSpecificationExecutor`。
```java
public interface FileInfoDao extends JpaRepository<FileInfo, Long>, JpaSpecificationExecutor<FileInfo>{
}
```

这个接口我用于查询文件信息，我们可以看到`JpaSpecificationExecutor`有以下这些方法:
```java
	T findOne(Specification<T> spec);
	List<T> findAll(Specification<T> spec);
	Page<T> findAll(Specification<T> spec, Pageable pageable);
	List<T> findAll(Specification<T> spec, Sort sort);
	long count(Specification<T> spec);
```

`indOne(spec)`: 根据条件查询获取一条数据;

`findAll(spec)`: 根据条件查询获取全部数据；

`findAll(specification, pageable)`: 根据条件分页查询数据，pageable设定页码、一页数据量，同时返回的是Page类对象，可以通过getContent()方法拿到List集合数据；

`findAll(specification, sort)`: 根据条件查询并返回排序后的数据；

`count(spec)`: 获取满足当前条件查询的数据总数；

现在光定义了dao也没用，人家需要你往方法里传Specification实现类的对象，Specification是一个接口：

```java
public interface Specification<T> {
	Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb);
}
```

所以我定义了一个实现类：

```java
public class SinoSpecification<T> implements Specification<T> {
	
	private TableQueryParam param;
	
	private Class<T> clazz;
	
	public SinoSpecification(TableQueryParam param, Class<T> clazz) {
		this.param = param;
		this.clazz = clazz;
	}
	
	@Override
	public Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb) {
		Field[] fields =  clazz.getDeclaredFields();
		Map<String, Object> conditions = param.getCondition();
		List<Predicate> predicates = new ArrayList<>();
		for (int i = 0; i < fields.length; i++) {
			fields[i].setAccessible(true);
			String name = fields[i].getName();
			if(conditions.containsKey(name)) {
				if(ObjectUtils.isEmpty(conditions.get(name))) {
					continue;
				}
				predicates.add(cb.like(root.get(name), "%"+conditions.get(name)+"%"));
			}
		}
		return cb.and(predicates.toArray(new Predicate[predicates.size()]));
	}
 
}
```

> 需要解释一下的是，`TableQueryParam`是我自定义的用于封装查询条件的类，有一个`condition`属性是条件Map集合。

这个类主要是通过反射获取实体类的字段属性，通过再去和`TableQueryParam`中的`conditon`里的条件进行比对，如果条件中包含，则将这个条件以及查询的值一起构造加入进`Predicate`，`cb.like(arg1, arg2)`这个方法，就代表模糊查询，类似sql中的Like，arg1参数代表要查询的字段，arg2代表查询的字段的值，转换成sql就是 `arg1 LIKE arg2`。当然，我这个自定义的`SinoSpecification`主要是用于模糊查询的，而真正Specification不仅仅支持模糊，还可以通过`CriteriaBuilder`里的方法来实现各种组合查询，例如`cb.equal()`进行等于查询，`cb.greaterThan()`、`cb.lessThan()`来实现大于小于，等等，可以说非常丰富。

定义好Specification之后，就可以调用前面说的findAll方法，将自定义的specification对象放进去即可。