# 在已有业务上添加新功能

## 概述
后端添加新功能遵循**数据库优先、自底向上**的原则：Database → Domain → Mapper → Service → Controller。

---

## 标准流程

### Step 1: 数据库设计

#### 评估影响
- 是否需要新增表？
- 是否需要修改现有表结构？
- 是否需要添加索引？
- 是否影响现有数据？

#### 编写 DDL 脚本
```sql
-- 新增字段示例
ALTER TABLE sys_user
ADD COLUMN last_login_time datetime COMMENT '最后登录时间' AFTER login_date;

-- 添加索引
ALTER TABLE sys_user
ADD INDEX idx_last_login_time (last_login_time);

-- 新增表示例
CREATE TABLE sys_user_ext (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键',
    user_id BIGINT NOT NULL COMMENT '用户ID',
    score INT DEFAULT 0 COMMENT '积分',
    level VARCHAR(20) COMMENT '等级',
    create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    update_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    UNIQUE KEY uk_user_id (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户扩展表';
```

#### ⚠️ 数据迁移
如果涉及现有数据，必须编写数据迁移脚本：
```sql
-- 为现有用户初始化扩展数据
INSERT INTO sys_user_ext (user_id, score, level)
SELECT user_id, 0, 'NORMAL'
FROM sys_user
WHERE NOT EXISTS (
    SELECT 1 FROM sys_user_ext WHERE sys_user_ext.user_id = sys_user.user_id
);
```

---

### Step 2: Domain 层

#### 修改/新增实体类
```java
// 修改现有实体
@TableName("sys_user")
public class SysUser extends BaseEntity {
    // 新增字段
    private LocalDateTime lastLoginTime;
    // getter/setter
}

// 新增实体类
@TableName("sys_user_ext")
public class SysUserExt extends BaseEntity {
    private Long userId;
    private Integer score;
    private String level;
}
```

#### 修改/新增 BO
```java
// SysUserBo.java - 用于接收前端参数
public class SysUserBo {
    private Long userId;
    private String userName;
    private LocalDateTime lastLoginTime;  // 新增
    // 验证注解
}
```

#### 修改/新增 VO
```java
// SysUserVo.java - 用于返回前端
public class SysUserVo implements Serializable {
    private Long userId;
    private String userName;
    private LocalDateTime lastLoginTime;  // 新增
    private Integer score;  // 关联扩展表
}
```

---

### Step 3: Mapper 层

#### 检查现有 Mapper
```
是否会影响现有 SQL 查询？
    ├── 会 → 评估影响范围，考虑新增方法
    └── 否 → 直接修改/新增
```

#### 新增 Mapper 方法
```java
// ISysUserMapper.java
public interface ISysUserMapper extends BaseMapperPlus<SysUser, SysUserVo> {
    /**
     * 更新最后登录时间
     */
    int updateLastLoginTime(@Param("userId") Long userId,
                            @Param("time") LocalDateTime time);
}
```

#### 编写 XML SQL
```xml
<!-- SysUserMapper.xml -->
<update id="updateLastLoginTime">
    UPDATE sys_user
    SET last_login_time = #{time}
    WHERE user_id = #{userId}
</update>
```

---

### Step 4: Service 层

#### 修改接口
```java
// ISysUserService.java
public interface ISysUserService extends IServicePlus<SysUser, SysUserVo> {
    /**
     * 记录登录时间
     */
    void recordLoginTime(Long userId);
}
```

#### 实现业务逻辑
```java
// SysUserServiceImpl.java
@Service
@RequiredArgsConstructor
public class SysUserServiceImpl extends ServicePlusImpl<ISysUserMapper, SysUser, SysUserVo>
    implements ISysUserService {

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void recordLoginTime(Long userId) {
        // 业务逻辑
        baseMapper.updateLastLoginTime(userId, LocalDateTime.now());

        // 如果涉及多表操作，需在同一事务中
        // userExtMapper.updateScore(userId, 10);
    }
}
```

---

### Step 5: Controller 层

#### 新增/修改接口
```java
@RestController
@RequestMapping("/system/user")
public class SysUserController extends BaseController {

    private final ISysUserService userService;

    /**
     * 记录登录时间
     */
    @SaCheckPermission("system:user:edit")
    @Log(title = "用户管理", businessType = BusinessType.UPDATE)
    @PostMapping("/recordLogin")
    public R<Void> recordLogin(@RequestBody Long userId) {
        userService.recordLoginTime(userId);
        return R.ok();
    }
}
```

---

### Step 6: 权限配置

#### 后端权限字符串
```java
@SaCheckPermission("system:user:recordLogin")  // 新增权限标识
```

#### 数据库菜单配置
```sql
-- 如果是新功能，需在 sys_menu 表中添加菜单/按钮权限
INSERT INTO sys_menu (menu_name, parent_id, order_num, path, component,
                      is_frame, is_cache, menu_type, visible, status,
                      perms, icon, create_by, create_time)
VALUES ('记录登录', 100, 6, '', '', 1, 0, 'F', '0', '0',
        'system:user:recordLogin', '#', 'admin', sysdate());
```

---

## Checklist

- [ ] 数据库 DDL 脚本已编写并执行
- [ ] 数据迁移脚本已执行（如需要）
- [ ] Domain 对象已更新
- [ ] Mapper 方法和 XML 已编写
- [ ] Service 业务逻辑已实现
- [ ] 事务注解已正确添加
- [ ] Controller 接口已添加
- [ ] 权限注解已配置
- [ ] 日志注解已添加
- [ ] 单元测试已编写
- [ ] 接口文档已更新
