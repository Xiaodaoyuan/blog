---
title: springboot使用mybatis加载typealias的package问题
date: 2018-09-30 18:02:32
categories: web
tags: springboot
---
 **springboot打成jar包后运行会造成mybatis设置的别名包找不到实体类的问题，这是由于mybatis默认使用DefaultVFS扫描包，在springboot中可以改成SpringBootVFS进行包扫描。具体见下面代码。**
 
 ---
<!--more-->
```
@Bean
    public SqlSessionFactory db1SqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSource);
        //mybatis使用默认DefaultVFS扫描包会有问题，使用SpringBootVFS
        sqlSessionFactoryBean.setVfs(SpringBootVFS.class); 
        PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        sqlSessionFactoryBean.setMapperLocations(resolver.getResources("classpath*:/mybatis/mapper/**.xml"));
        sqlSessionFactoryBean.setConfigLocation(new ClassPathResource("mybatis/mybatis-config.xml"));
        return sqlSessionFactoryBean.getObject();
    }
```

---

如果使用 mybatis autoconfigure生成的 SessionFactory可能就没有这个问题了。
