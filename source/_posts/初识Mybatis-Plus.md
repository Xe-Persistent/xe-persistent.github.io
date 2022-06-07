---
title: 初识Mybatis-Plus
date: 2022-05-30 14:12:50
tags: 
    - 后端
    - Java
    - Mybatis-Plus
categories: Java
keywords: "Java, Mybatis-Plus"
---
# 前言
在前面的Java学习笔记中，我提到过两次Mybatis-Plus，可能大家已经对这个名词有些耳熟。我对它的定义是：一个方便的操作数据库功能的工具包。

在它的官方文档上写道：MyBatis-Plus是一个MyBatis的增强工具，在MyBatis的基础上只做增强不做改变，为简化开发、提高效率而生。那么，MyBatis又是什么呢？这里我不做更多描述。在使用MyBatis-Plus的时候，我总是为之惊叹：原来数据库操作能这么简单！于是写下这篇文章记录它的一些使用方法。

# 应用
接下来，我将介绍如何使用Mybatis-Plus的条件构造器编写查询逻辑。大体上，Mybatis-Plus从数据库表到前端接口的业务流程是Table->DTO->Mapper->Service->Controller，因此，我将顺着这条流程依次讲解每个环节要使用Mybatis-Plus编写哪些类。

## DTO
首先是DTO，DTO即数据传输对象，是数据从后端传输到前端的载体。大体上，一个DTO要包含一个数据库表的部分或所有字段信息。为了减少重复代码，我将这些表的重复字段单列出一个ModelDTO，让其他的DTO继承这个ModelDTO。建议使用Lombok插件提供的@Data注解，可以为DTO的私有成员自动生成getter和setter方法。

> ModelDTO.java
```java
@Data
public class ModelDTO {

    private String name;

    private Long id;
}
```

> MonitoringDeviceDTO.java
```java
@EqualsAndHashCode(callSuper = true)
@TableName("monitoringdevice")
@Data
public class MonitoringDeviceDTO extends ModelDTO {
    private String model;

    private String type;

    private String illustrate;

    private String location;

    private String accuracy;

    private String frequency;
}
```

## Mapper
Mapper的编写非常简单，只需继承BaseMapper即可，以MonitoringDeviceMapper为例：
> MonitoringDeviceMapper.java
```java
@Mapper
public class MonitoringDeviceMapper extends BaseMapper<MonitoringDeviceDTO> {

}
```

## Service
事实上，这些Mapper继承BaseMapper后已经为我们提供了默认的CRUD接口。当我们需要在这些接口的基础之上扩展功能时，就需要自行编写代码啦。
> CRUD是指做计算处理时的增加(Create)、读取查询(Retrieve)、更新(Update)和删除(Delete)几个单词的首字母简写。代表了数据库或者持久层的基本操作功能。

首先是增加和编辑数据的逻辑。这里为了便于维护代码，我定义了RequestBody作为两个方法的入参。此外添加了判断字段是否合法的逻辑。

由于DTO与RequestBody都是JavaBean，可以使用BeanUtils的copyProperties()方法将RequestBody的参数复制给DTO。

在编辑方法中，使用链式条件构造器`LambdaQueryChainWrapper`根据传入的实体id查询数据库中是否有对应的数据，再使用`exists()`方法返回布尔值，达到判断编辑的数据是否存在的目的。
> MonitoringDeviceService.java
```java
public Boolean addDevice(AddMonitoringDeviceReq device) {
    if (!DetermineEnum.isInclude(device.getType())) {
        throw new ParamException("参数不支持");
    }
    MonitoringDeviceDTO instance = new MonitoringDeviceDTO();
    BeanUtils.copyProperties(device, instance);
    monitoringDeviceMapper.insert(instance);
    return true;
}

public MonitoringDeviceDTO editDevice(EditMonitoringDeviceReq device) {
    boolean exists = new LambdaQueryChainWrapper<>(monitoringDeviceMapper)
            .eq(MonitoringDeviceDTO::getId, device.getId()).exists();
    if (!exists) {
        throw new NotFoundException("设备不存在");
    }
    MonitoringDeviceDTO instance = new MonitoringDeviceDTO();
    BeanUtils.copyProperties(device, instance);
    return monitoringDeviceMapper.insert(instance);
}
```

然后是查询数据的逻辑，
```java
public MonitoringDeviceDTO getDeviceById(Long deviceId) {
    if (Objects.isNull(deviceId)) {
        return null;
    }
    return monitoringDeviceMapper.selectById(deviceId);
}

public MonitoringDeviceDTO getDeviceByParamId(Long paramId) {
    if (Objects.isNull(paramId)) {
        return null;
    }
    return getDeviceById(new LambdaQueryChainWrapper<>(determineMapper)
            .eq(DetermineDTO::getCalcParamId, paramId)
            .one().getDeviceId());
}

public List<MonitoringDeviceDTO> getDeviceByType(String type) {
    if (!DetermineEnum.isInclude(type)) {
        throw new ParamException("排放参数不支持");
    }
    return new LambdaQueryChainWrapper<>(monitoringDeviceMapper)
            .eq(MonitoringDeviceDTO::getType, type).list();
}

public List<MonitoringDeviceDTO> getDeviceBySource(Long sourceId) {
    boolean exists = new LambdaQueryChainWrapper<>(emissionSourceMapper)
            .eq(EmissionSourceDTO::getId, sourceId).exists();
    if (!exists) {
        throw new NotFoundException("排放源不存在");
    }
    List<EmissionCalcParamDTO> emissionCalcParams = emissionSourceCalcParamMapper
            .selectList(new QueryWrapper<>(EmissionCalcParamDTO.class)
                    .treeNode(ModelLabelConstant.EMISSION_SOURCE, sourceId));
    if (CollectionUtils.isEmpty(emissionCalcParams)) {
        throw new DefaultCarbonException("该排放源的排放参数异常");
    }
    List<Long> paramIdList = emissionCalcParams.stream()
            .map(EmissionCalcParamDTO::getId)
            .collect(Collectors.toList());
    List<DetermineDTO> determineList = new LambdaQueryChainWrapper<>(determineMapper)
            .eq(DetermineDTO::getObtainingMethod, 2)
            .in(DetermineDTO::getCalcParamId, paramIdList)
            .list();
    return determineList.stream()
            .map(DetermineDTO::getDeviceId)
            .map(this::getDeviceById)
            .collect(Collectors.toList());
}
```
最后是删除数据的逻辑
```java
public Boolean deleteDevice(Long deviceId) {
    List<MaintenanceRecordDTO> records = maintenanceRecordService.getRecordByDeviceId(deviceId);
    if (CollectionUtils.isNotEmpty(records)) {
        monitoringDeviceMapper.deleteBatchIds(records.stream()
                .map(MaintenanceRecordDTO::getId)
                .collect(Collectors.toList()));
    }
    return monitoringDeviceMapper.deleteById(deviceId) == 1;
}
```

# 小结
| 函数名         | SQL                      | 函数例子                                                           | SQL例子                                              |
|:------------|:-------------------------|:---------------------------------------------------------------|:---------------------------------------------------|
| eq          | 等于=                      | eq("name", "老王)                                                | name = '老王'                                        |
| ne          | 不等于<>                    | ne("name", "老王)                                                | name <> '老王'                                       |
| gt          | 大于>                      | gt("age", 18)                                                  | age > 18                                           |
| e           | 大于等于>=                   | ge("age", 18)                                                  | age >= 18                                          |
| t           | 小于<                      | It("age", 18)                                                  | age < 18                                           |
| e           | 小于<=                     | le("age", 18)                                                  | age <= 18                                          |
| between     | BETWEEN 值1 AND 值2        | between("age", 18, 30)                                         | age between 18 and 30                              |
| notBetween  | NOT BETWEEN 值1 AND 值2    | notBetween("age", 18, 30)                                      | age not between 18 and 30                          |
| like        | LIKE '%值%'               | like("name", "王")                                              | name like '%王%'                                    |
| notLike     | NOT LIKE '%值%'           | notLike("name", "王")                                           | name not like '%王%'                                |
| likeLeft    | LIKE '%值'                | likeLeft("name", "王")                                          | name like '%王'                                     |
| likeRight   | LIKE '值%'                | likeRight("name", "王")                                         | name like '王%'                                     |
| isNull      | 字段 IS NULL               | isNull("name")                                                 | name is null                                       |
| isNotNull   | 字段 IS NOT NULL           | isNotNull("name")                                              | name is not null                                   |
| in          | 字段 IN (v0, v1, ...)      | in("age", {1, 2, 3})                                           | age in (1, 2, 3)                                   |
| notIn       | 字段 NOT IN (v0, v1, ...)  | notIn("age", {1, 2, 3})                                        | age not in (1, 2, 3)                               |
| inSql       | 字段 IN(sql语句)             | inSql("id", "select id from table where id < 3")               | id in (select id from table where id < 3)          |
| notInSql    | 字段 NOT IN (sql语句)        | notInSql("id", "select id from table where id < 3")            | age not in (select id from table where id < 3)     |
| groupBy     | 分组 GROUP BY 字段, ...      | groupBy("id", "name")                                          | group by id, name                                  |
| orderByAsc  | 排序 ORDER BY 字段, ... ASC  | orderByAsc("id", "name")                                       | order by id ASC, name ASC                          |
| orderByDesc | 排序 ORDER BY 字段, ... DESC | orderByDesc("id", "name")                                      | order by id DESC, name DESC                        |
| orderBy     | 排序 ORDER BY 字段, ...      | orderBy(true, true, "id", "name")                              | order by id ASC, name ASC                          |
| having      | HAVING (sql语句)           | having("sum(age) > {0}", 11)                                   | having sum(age) > 11                               |
| or          | 拼接 OR                    | eq("id", 1).or().eq("name", "老王")                              | id = 1 or name = '老王                               |
| and         | AND 嵌套                   | and(i -> i.eq("name", "李白").ne("status", "活着"))                | and (name = '李白' and status <> '活着')               |
| apply       | 拼接sql                    | apply("date_format(dateColumn, '%Y-%m-%d')={0}", "2008-08-08") | date_format(dateColumn,'%Y-%m-%d') = '2008-08-08') |
| last        | 无视优化规则直接拼接到sql的最后        | last("limit 1")                                                |                                                    |
| exists      | 拼接 EXISTS (sql语句)        | exists("select id from table where age = 1")                   | exists (select id from table where age = 1)        |
| notExists   | 拼接 NOT EXISTS (sql语句)    | notExists("select id from table where age = 1")                | not exists (select id from table where age = 1)    |
| nested      | 正常嵌套不带AND或者0R            | nested(i -> i.eq("name", "李白"). ne("status", "活着"))            | (name = '李白' and status <> '活着')                   |


---
**非常感谢你的阅读，辛苦了！**

---
参考文章： (感谢以下资料提供的帮助)
- [MyBatis-Plus LambdaQueryWrapper使用说明](https://blog.csdn.net/qlzw1990/article/details/116996422)
- [Mybatis plus 之 QueryWrapper、LambdaQueryWrapper、LambdaQueryChainWrapper](https://adong.blog.csdn.net/article/details/122154137)
