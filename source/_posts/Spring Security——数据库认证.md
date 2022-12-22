---
title: Spring Security——数据库认证
tags: ["Spring", "Spring Security", "认证授权"]
categories: ["Spring Security"]
---
在上一篇中我们介绍了多种`Spring Security`配置用户名密码的方式，但那些都是基于内存的，实际生产中用户信息往往存储在数据库中，因此本篇我们来学习如何基于数据库实现认证。
# RBAC相关模型设计
RBAC——Role Based Access Control，基于角色的访问控制。通常设计5张表：用户表、角色表、权限表、用户角色关系表以及角色权限关系表，具体表结构如下：
<!-- more -->
```sql
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for permissions
-- ----------------------------
DROP TABLE IF EXISTS `permissions`;
CREATE TABLE `permissions`  (
  `id` bigint(0) NOT NULL AUTO_INCREMENT,
  `parent_id` bigint(0) DEFAULT NULL,
  `name` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
  `en_name` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
  `url` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci DEFAULT NULL,
  `description` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 7 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci COMMENT = '权限' ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of permissions
-- ----------------------------
INSERT INTO `permissions` VALUES (1, NULL, '系统管理', 'SYSTEM_MANAGE', '/', NULL);
INSERT INTO `permissions` VALUES (2, 1, '用户管理', 'USER_MANAGE', '/user', NULL);
INSERT INTO `permissions` VALUES (3, 2, '查看用户', 'USER_VIEW', NULL, NULL);
INSERT INTO `permissions` VALUES (4, 2, '新增用户', 'USER_ADD', NULL, NULL);
INSERT INTO `permissions` VALUES (5, 2, '修改用户', 'USER_MODIFY', NULL, NULL);
INSERT INTO `permissions` VALUES (6, 2, '删除用户', 'USER_REMOVE', NULL, NULL);

-- ----------------------------
-- Table structure for role
-- ----------------------------
DROP TABLE IF EXISTS `role`;
CREATE TABLE `role`  (
  `id` bigint(0) NOT NULL AUTO_INCREMENT,
  `parent_id` bigint(0) DEFAULT NULL,
  `name` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
  `en_name` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
  `description` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 2 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci COMMENT = '角色' ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of role
-- ----------------------------
INSERT INTO `role` VALUES (1, NULL, '超级管理员', 'SUPER_ADMIN', NULL);

-- ----------------------------
-- Table structure for role_permissions
-- ----------------------------
DROP TABLE IF EXISTS `role_permissions`;
CREATE TABLE `role_permissions`  (
  `id` bigint(0) NOT NULL AUTO_INCREMENT,
  `role_id` bigint(0) NOT NULL,
  `permissions_id` bigint(0) NOT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 7 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of role_permissions
-- ----------------------------
INSERT INTO `role_permissions` VALUES (1, 1, 1);
INSERT INTO `role_permissions` VALUES (2, 1, 2);
INSERT INTO `role_permissions` VALUES (3, 1, 3);
INSERT INTO `role_permissions` VALUES (4, 1, 4);
INSERT INTO `role_permissions` VALUES (5, 1, 5);
INSERT INTO `role_permissions` VALUES (6, 1, 6);

-- ----------------------------
-- Table structure for user
-- ----------------------------
DROP TABLE IF EXISTS `user`;
CREATE TABLE `user`  (
  `id` bigint(0) NOT NULL AUTO_INCREMENT,
  `username` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
  `password` varchar(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
  `phone` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci DEFAULT NULL,
  `email` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE INDEX `uniq_username`(`username`) USING BTREE,
  UNIQUE INDEX `uniq_phone`(`phone`) USING BTREE,
  UNIQUE INDEX `uniq_email`(`email`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 29 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci COMMENT = '用户' ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of user
-- ----------------------------
INSERT INTO `user` VALUES (1, 'liteng', '$2a$10$dRQvD0SVK2KcA4eXQYPR.ODP.RlmvddP5HEcjdoQ7y.JF.rTHph3u', NULL, NULL);

-- ----------------------------
-- Table structure for user_role
-- ----------------------------
DROP TABLE IF EXISTS `user_role`;
CREATE TABLE `user_role`  (
  `id` bigint(0) NOT NULL AUTO_INCREMENT,
  `user_id` bigint(0) NOT NULL,
  `role_id` bigint(0) NOT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 2 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of user_role
-- ----------------------------
INSERT INTO `user_role` VALUES (1, 1, 1);

SET FOREIGN_KEY_CHECKS = 1;
```
# 引入持久层依赖——MyBatis
```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.2.2</version>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.30</version>
</dependency>

<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.62</version>
</dependency>

<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.10</version>
</dependency>
```
# 配置数据源
```yml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://192.168.175.175:3306/zookeeper?useUnicode=true&characterEncoding=utf-8
    username: liteng
    password: 123456
```
# 创建实体类
```java
@Data
public class User {
    private Long id;
    private String username;
    private String password;
    private String phone;
    private String email;
}
```
```java
@Data
public class Role {
    private Long id;
    private Long parentId;
    private String name;
    private String enName;
    private String description;
}
```
```java
@Data
public class Permissions {
    private Long id;
    private Long parentId;
    private String name;
    private String enName;
    private String url;
    private String description;
}
```
# 创建Mapper
```java
public interface UserMapper {
    @Select("select * from user where username = #{username}")
    User selectByUsername(String username);
}
```
```java
public interface PermissionsMapper {
    @Select("SELECT\n" +
            "\tid, \n" +
            "\tparent_id as parentId, \n" +
            "\tname, \n" +
            "\ten_name as enName, \n" +
            "\turl, \n" +
            "\tdescription \n" +
            "FROM\n" +
            "\tpermissions \n" +
            "WHERE\n" +
            "\tid IN ( SELECT permissions_id FROM role_permissions WHERE role_id IN ( SELECT role_id FROM user_role WHERE user_id = #{id} ) )")
    List<Permissions> selectByUserId(Long id);
}
```
# 指定Mapper扫描路径
```java
@SpringBootApplication
@MapperScan(basePackages = "com.stockeeper.spring.security.study.mapper")
public class SpringSecurityStudyApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringSecurityStudyApplication.class, args);
    }

}
```
# 自定义数据库认证逻辑
```java
@Component
public class MyUserDetailsService implements UserDetailsService {

    @Autowired
    private UserMapper userMapper;

    @Autowired
    private PermissionsMapper permissionsMapper;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

        User user = userMapper.selectByUsername(username);
        if (user == null) {
            throw new UsernameNotFoundException("用户名不存在");
        }

        List<GrantedAuthority> authorities = new ArrayList<>();

        List<Permissions> permissionsList = permissionsMapper.selectByUserId(user.getId());
        permissionsList.forEach(permissions -> authorities.add(new SimpleGrantedAuthority(permissions.getEnName())));

        return org.springframework.security.core.userdetails.User.withUsername(username).password(user.getPassword()).authorities(authorities).build();

    }

}
```
# WebSecurityConfigurerAdapter配置
```java
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private MyUserDetailsService myUserDetailsService;

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(); // bcrypt加密
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(myUserDetailsService);
    }

}
```