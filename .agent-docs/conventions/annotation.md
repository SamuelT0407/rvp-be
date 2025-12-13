# 注解使用规范

## 核心注解

### @SaCheckPermission (权限控制)
```java
// 单个权限
@SaCheckPermission("system:user:query")

// 多个权限 AND
@SaCheckPermission(value = {"system:user:query", "system:user:export"}, mode = SaMode.AND)

// 多个权限 OR
@SaCheckPermission(value = {"system:user:query", "system:dept:query"}, mode = SaMode.OR)
```

**位置**: Controller 方法上
**作用**: 接口级权限控制

---

### @Log (操作日志)
```java
@Log(title = "用户管理", businessType = BusinessType.INSERT)
@Log(title = "用户管理", businessType = BusinessType.UPDATE)
@Log(title = "用户管理", businessType = BusinessType.DELETE)
@Log(title = "用户管理", businessType = BusinessType.EXPORT)
@Log(title = "用户管理", businessType = BusinessType.IMPORT)
```

**位置**: Controller 方法上
**必须添加的操作**: 新增、修改、删除、导出、导入

---

### @RepeatSubmit (防重复提交)
```java
@RepeatSubmit()  // 使用默认间隔 (5秒)
@RepeatSubmit(interval = 10, timeUnit = TimeUnit.SECONDS)  // 自定义间隔
```

**位置**: Controller 方法上
**使用场景**: 所有写操作 (新增、修改、删除)

---

### @Transactional (事务)
```java
@Transactional(rollbackFor = Exception.class)  // 标准用法
@Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)  // 新事务
@Transactional(readOnly = true)  // 只读事务
```

**位置**: Service 方法上
**必须参数**: `rollbackFor = Exception.class`

---

### @Validated (参数校验)
```java
// Controller 类上
@Validated
@RestController
public class SysUserController { }

// 方法参数上
public R add(@Validated @RequestBody SysUserBo user) { }
public R edit(@Validated(Edit.class) @RequestBody SysUserBo user) { }
```

**分组校验**:
```java
public class SysUserBo {
    @NotNull(message = "ID不能为空", groups = {Edit.class})
    private Long userId;

    @NotBlank(message = "用户名不能为空", groups = {Add.class, Edit.class})
    private String userName;
}

// 使用
public R add(@Validated(Add.class) @RequestBody SysUserBo user) { }
public R edit(@Validated(Edit.class) @RequestBody SysUserBo user) { }
```

---

## MyBatis-Plus 注解

### @TableName (表名)
```java
@TableName("sys_user")
public class SysUser { }

// 自动填充
@TableName(value = "sys_user", autoResultMap = true)
public class SysUser { }
```

### @TableId (主键)
```java
@TableId(value = "id", type = IdType.AUTO)  // 数据库自增
@TableId(value = "id", type = IdType.ASSIGN_ID)  // 雪花算法
```

### @TableField (字段)
```java
// 非表字段
@TableField(exist = false)
private List<SysRole> roles;

// 自动填充
@TableField(fill = FieldFill.INSERT)
private LocalDateTime createTime;

@TableField(fill = FieldFill.INSERT_UPDATE)
private LocalDateTime updateTime;
```

---

## 校验注解

### 常用校验
```java
@NotNull(message = "不能为空")
@NotBlank(message = "不能为空")  // 字符串专用
@NotEmpty(message = "不能为空")  // 集合专用

@Size(min = 2, max = 20, message = "长度必须在2-20之间")
@Length(min = 2, max = 20, message = "长度必须在2-20之间")

@Min(value = 0, message = "最小值为0")
@Max(value = 100, message = "最大值为100")

@DecimalMin(value = "0.01", message = "必须大于0")
@DecimalMax(value = "9999.99", message = "不能超过9999.99")

@Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式错误")
@Email(message = "邮箱格式错误")
```

---

## Lombok 注解

### 实体类常用
```java
@Data                           // getter/setter/toString/equals/hashCode
@EqualsAndHashCode(callSuper = true)  // 继承父类的 equals/hashCode
@TableName("sys_user")
public class SysUser extends BaseEntity {
    @TableId(type = IdType.AUTO)
    private Long userId;
}
```

### Service 常用
```java
@Service
@RequiredArgsConstructor  // 生成带 final 字段的构造器
public class SysUserServiceImpl implements ISysUserService {

    private final ISysUserMapper userMapper;  // 自动注入
    private final ISysRoleService roleService;
}
```

---

## 注解组合使用

### Controller 完整示例
```java
@Validated
@RequiredArgsConstructor
@RestController
@RequestMapping("/system/user")
public class SysUserController extends BaseController {

    private final ISysUserService userService;

    @SaCheckPermission("system:user:add")
    @Log(title = "用户管理", businessType = BusinessType.INSERT)
    @RepeatSubmit()
    @PostMapping
    public R<Void> add(@Validated @RequestBody SysUserBo user) {
        return toAjax(userService.insertUser(user));
    }
}
```

### Service 完整示例
```java
@Service
@RequiredArgsConstructor
public class SysUserServiceImpl extends ServicePlusImpl<ISysUserMapper, SysUser, SysUserVo>
    implements ISysUserService {

    private final ISysUserMapper userMapper;

    @Override
    @Transactional(rollbackFor = Exception.class)
    public Boolean insertUser(SysUserBo bo) {
        // 业务逻辑
    }
}
```
