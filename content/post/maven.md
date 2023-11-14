---
title: "Maven"
date: 2023-11-14T11:28:41+08:00
tags: ["基础知识"]
---

# Maven

Java打包工具

# Setting文件

参考https://juejin.cn/post/7035620309915058207

- LocalRepository

  本地仓库地址

- servers

  ```xml
  <servers>
      <server>
        <!-- 服务的唯一定义符, 用来区分不同的代理元素 -->
        <id>deploymentRepo</id>
  	  <!-- 鉴权用户名 -->        
        <username>repouser</username>
        <!-- 鉴权密码 -->
        <password>repopwd</password>
      </server>
      
      <server>
        <id>siteServer</id>
        <!-- 鉴权时的私钥位置 -->
        <privateKey>/path/to/private/key</privateKey>
        <!-- 鉴权时的私钥密码 -->
        <passphrase>optional; leave empty if not used.</passphrase>
      </server>
      
  </servers>
  
  ```

  
