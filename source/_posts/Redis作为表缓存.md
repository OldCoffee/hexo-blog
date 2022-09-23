title: Redis+反射实现表缓存
author: HYF
tags: []
categories: []
date: 2022-09-23 15:03:00
---
## 一、根据主键查询缓存
--- 
### 1. 实现逻辑
 1. 根据传参DO 通过反射获取主键注解
 2. 根据主键Field 创建 QueryWrapper 语句，创建redis key 的StringBuilder
 3. 根据Redis 查询是否已存在缓存，若存在则返回缓存，否则查询数据库并缓存

### 2. 示例代码
```
    /**
     * 根据枚举获取Field
     *
     * @param obj
     * @param annotationClz
     * @return
     */
    public static List<Field> getFieldsByAnnotation(Object obj, Class annotationClz) {
        Class clz = obj.getClass();
        //注解判断-key处理
        Field[] fields = clz.getDeclaredFields();
        List<Field> fieldList = new ArrayList<>();
        for (Field field : fields) {
            if (field.isAnnotationPresent(annotationClz)) {
                fieldList.add(field);
            }
        }
        return fieldList;
    }
    
        default QueryWrapper<T> getQueryWrapper(T t, List<Field> fieldList, StringBuilder stringBuilder) {
        QueryWrapper<T> queryWrapper = new QueryWrapper<T>();
        for (Field field : fieldList) {
            field.setAccessible(true);
            Object value = null;
            try {
                value = field.get(t);
            } catch (IllegalAccessException e) {
                throw new RuntimeException(e);
            }
            queryWrapper.eq(field.getName(), value);
            if (stringBuilder.length() > 0) {
                stringBuilder.append(",");
            }
            stringBuilder.append(value);
        }
        return queryWrapper;
    }

	/**
    * 根据主键查询缓存
    */
    default T selectCacheByPrimaryKey(T t) {
        List<Field> fieldList = ClzUtils.getFieldsByAnnotation(t, Key.class);
        if (fieldList.size() == 0) {
            throw new ServerException(GlobalErrorCodeConstants.DB_KEY_IS_NOT_EXISTS);
        }
        StringBuilder stringBuilder = new StringBuilder();
        QueryWrapper<T> queryWrapper = getQueryWrapper(t, fieldList, stringBuilder);
        Object redisObj = redisUtil.get(RedisConstant.DATA_CACHE_ENTITY, t.getClass().getSimpleName(), stringBuilder);
        if (redisObj != null) {
            return (T) redisObj;
        }
        //查询及缓存处理
        T result = selectOne(queryWrapper);
        redisUtil.set(String.format(RedisConstant.DATA_CACHE_ENTITY, t.getClass().getSimpleName(), stringBuilder), result);
        return result;
    }
```

## 二、实现分组缓存
--- 
### 1. 实现逻辑
 1. 根据传参DO 通过反射获取分组注解
 2. 根据分组Field 创建 QueryWrapper 语句，创建redis key 的Map<String,StringBuilder>
 3. 根据Redis 查询是否已存在缓存，若存在则根据缓存结果查询主键缓存，组装并返回数，否则查询数据库并通过循环查出主键缓存，讲主键缓存的Key存入
 
### 2. 示例代码
``` 
/**
     * 根据枚举获取Field
     *
     * @param obj
     * @param annotationClz
     * @return
     */
    public static List<Field> getFieldsByAnnotation(Object obj, Class annotationClz) {
        Class clz = obj.getClass();
        //注解判断-key处理
        Field[] fields = clz.getDeclaredFields();
        List<Field> fieldList = new ArrayList<>();
        for (Field field : fields) {
            if (field.isAnnotationPresent(annotationClz)) {
                fieldList.add(field);
            }
        }
        return fieldList;
    }

    @NotNull
    default QueryWrapper<T> getQueryWrapper(T t, List<Field> fieldList, Map<String, StringBuilder> groupNameMap) {
        QueryWrapper<T> queryWrapper = new QueryWrapper<T>();
        for (Field field : fieldList) {
            field.setAccessible(true);
            Object value = null;
            try {
                value = field.get(t);
            } catch (IllegalAccessException e) {
                throw new RuntimeException(e);
            }
            queryWrapper.eq(field.getName(), value);
            groupBuild(groupNameMap, field, value);

        }
        return queryWrapper;
    }

    /**
     * 创建分组数据
     *
     * @param groupNameMap
     * @param field
     * @param value
     */
    default void groupBuild(Map<String, StringBuilder> groupNameMap, Field field, Object value) {
        if (field.isAnnotationPresent(Group.class)) {
            Group annotation = field.getAnnotation(Group.class);
            String name = annotation.name();
            StringBuilder builder = groupNameMap.get(name);
            if (builder == null || builder.length() == 0) {
                builder = new StringBuilder();
            } else {
                builder.append(",");
            }
            builder.append(value);
            groupNameMap.put(name, builder);
        }
    }

    /**
     * 根据分组获取
     *
     * @param t
     * @return
     */
    default List<T> selectListCacheByGroup(T t) {

        Class annotationClz = Group.class;
        List<Field> fieldList = ClzUtils.getFieldsByAnnotation(t, annotationClz);
        if (fieldList.size() == 0) {
            throw new ServerException(GlobalErrorCodeConstants.DB_GROUP_IS_NOT_EXISTS);
        }
        Map<String, StringBuilder> groupNameMap = new HashMap<>();
        QueryWrapper<T> queryWrapper = getQueryWrapper(t, fieldList, groupNameMap);
        String group = null;
        StringBuilder stringBuilder = new StringBuilder();
        Set<Map.Entry<String, StringBuilder>> entrySet = groupNameMap.entrySet();
        if (entrySet.size() > 1) {
            throw new ServerException(GlobalErrorCodeConstants.DB_GROUP_NAME_DIFFERENT);
        }
        for (Map.Entry<String, StringBuilder> stringStringBuilderEntry : entrySet) {
            stringBuilder = stringStringBuilderEntry.getValue();
            group = stringStringBuilderEntry.getKey();
            if (stringBuilder == null || stringBuilder.length() == 0 || group == null) {
                throw new ServerException(GlobalErrorCodeConstants.DB_GROUP_VALUE_IS_EMPTY);
            }
        }
        //若缓存中存在，则取出所有缓存值，并通过该值取出对应实体 并组装成List 返回
        String key = String.format(RedisConstant.DATA_CACHE_GROUP, t.getClass().getSimpleName(), stringBuilder, group);
        Set<T> redisSet = redisUtil.members(key);
        if (!CollectionUtil.isEmpty(redisSet)) {
            List<T> list = new ArrayList<>();
            for (T objKey : redisSet) {
                Object obj = redisUtil.get(objKey);
                if (ObjectUtil.isNotEmpty(obj)) {
                    list.add((T) obj);
                }
            }
            return list;
        }

        List<T> result = selectList(queryWrapper);
        //保存分组缓存:分组中保存实体缓存键
        if (!CollectionUtil.isEmpty(result)) {
            for (T obj : result) {
                Map<String, StringBuilder> map = getKeyMap(obj,Key.class);
                //删除主键缓存
                StringBuilder keyValue = map.get(RedisConstant.RedisGroup.KEY.getValue());
                String objKey = String.format(RedisConstant.DATA_CACHE_ENTITY, t.getClass().getSimpleName(), keyValue);
                Object redisObj = redisUtil.get(objKey);
                if (ObjectUtil.isEmpty(redisObj)){
                    redisUtil.set(objKey, obj);
                }
                if (!redisUtil.isMember(key, objKey)) {
                    redisUtil.add(key, objKey);
                }
            }

        }
        return result;
    }
```

## 三、删除与更新逻辑
---
### 1. 实现逻辑
 1. 根据传参DO 通过反射获取主键注解
 2. 根据主键Field 创建 QueryWrapper 语句，创建redis key 的Map<String,StringBuilder>
 3. 根据Redis 查询是否已存在缓存，若存在则删除主键缓存，并删除相关分组缓存内主键缓存Key
 
### 2.示例代码
```
/**
* 获取KeyMap
*/
    default Map<String, StringBuilder> getKeyMap(T obj,Class keyClz ){
        List<Field> keyFieldList = ClzUtils.getFieldsByAnnotation(obj, keyClz);
        if (keyFieldList.size() == 0) {
            throw new ServerException(GlobalErrorCodeConstants.DB_KEY_IS_NOT_EXISTS);
        }
        Map<String, StringBuilder> map = getStringBuilderMap(obj, keyFieldList);
        return map;
    }

/**
* 删除主键缓存及分组缓存内主键缓存Key
*/
    default void removeRedisKeyAndGroup(T t) {
        Map<String, StringBuilder> map = getKeyMap(t, Key.class);
        //删除主键缓存
        StringBuilder keyValue = map.get(RedisConstant.RedisGroup.KEY.getValue());
        if (keyValue != null && keyValue.length() > 0) {
            String objKey = String.format(RedisConstant.DATA_CACHE_ENTITY, t.getClass().getSimpleName(), keyValue);
            Object obj = redisUtil.get(objKey);
            if (ObjectUtil.isNotEmpty(obj)) {
                redisUtil.delete(RedisConstant.DATA_CACHE_ENTITY, t.getClass().getSimpleName(), keyValue);
                Map<String, StringBuilder> groupMap = getKeyMap((T) obj, Group.class);
                for (Map.Entry<String, StringBuilder> stringStringBuilderEntry : groupMap.entrySet()) {
                    String key = String.format(RedisConstant.DATA_CACHE_GROUP, t.getClass().getSimpleName(), stringStringBuilderEntry.getValue(), stringStringBuilderEntry.getKey());
                    if (redisUtil.isMember(key, objKey)) {
                        redisUtil.remove(key, objKey);
                    }
                }

            }
            map.remove(RedisConstant.RedisGroup.KEY.getValue());
        }
    }
```