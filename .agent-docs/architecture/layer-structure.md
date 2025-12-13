# 分层架构规范

## RuoYi-Vue-Plus 四层架构

```
┌─────────────────────────────────────────┐
│          Controller 层                   │  # 接口层
│  - 接口定义、参数校验、权限控制          │
│  - 依赖: Service                        │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│          Service 层                      │  # 业务层
│  - 业务逻辑、事务管理、数据封装          │
│  - 依赖: Mapper, 其他Service             │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│          Mapper 层                       │  # 数据访问层
│  - SQL 编写、数据库交互                  │
│  - 依赖: 无 (只操作数据库)               │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│          Domain 层                       │  # 领域层
│  - Entity, Bo, Vo, Query               │
│  - 依赖: 无 (纯数据对象)                 │
└─────────────────────────────────────────┘
```

---

## Controller 层职责

### 应该做
✅ 接受HTTP请求，解析参数
✅ 参数校验 (`@Validated`)
✅ 权限控制 (`@SaCheckPermission`)
✅ 调用 Service 层方法
✅ 封装返回结果 (`R<T>`)
✅ 记录操作日志 (`@Log`)

### 不应该做
❌ 直接调用 Mapper
❌ 编写业务逻辑
❌ 处理事务
❌ 直接操作数据库

### 示例
```java
@RestController
@RequestMapping("/system/user")
public class SysUserController extends BaseController {

    private final ISysUserService userService;

    @SaCheckPermission("system:user:list")
    @GetMapping("/list")
    public TableDataInfo<SysUserVo> list(SysUserBo user, PageQuery pageQuery) {
        // ✅ 只负责调用Service
        return userService.selectPageUserList(user, pageQuery);
    }

    @SaCheckPermission("system:user:add")
    @Log(title = "用户管理", businessType = BusinessType.INSERT)
    @PostMapping
    public R<Void> add(@Validated @RequestBody SysUserBo user) {
        // ✅ 参数校验、权限控制、日志记录
        return toAjax(userService.insertUser(user));
    }
}
```

---

## Service 层职责

### 应该做
✅ 实现业务逻辑
✅ 控制事务边界 (`@Transactional`)
✅ 调用 Mapper 操作数据
✅ 调用其他 Service
✅ 数据转换 (Bo ↔ Entity ↔ Vo)
✅ 业务校验

### 不应该做
❌ 处理 HTTP 请求
❌ 权限控制 (应在Controller)
❌ 参数校验 (应在Bo层通过注解)

### 示例
```java
@Service
@RequiredArgsConstructor
public class SysUserServiceImpl implements ISysUserService {

    private final ISysUserMapper userMapper;
    private final ISysRoleService roleService;

    @Override
    @Transactional(rollbackFor = Exception.class)
    public Boolean insertUser(SysUserBo bo) {
        // ✅ 业务校验
        if (!checkUserNameUnique(bo)) {
            throw new ServiceException("用户名已存在");
        }

        // ✅ 数据转换
        SysUser entity = MapstructUtils.convert(bo, SysUser.class);

        // ✅ 多表操作在同一事务内
        userMapper.insert(entity);
        if (ArrayUtil.isNotEmpty(bo.getRoleIds())) {
            roleService.insertUserRole(entity.getUserId(), bo.getRoleIds());
        }

        return true;
    }
}
```

---

## Mapper 层职责

### 应该做
✅ 定义数据访问接口
✅ 编写 SQL (XML)
✅ 使用 MyBatis-Plus 基础方法

### 不应该做
❌ 包含业务逻辑
❌ 调用其他 Service
❌ 处理事务

### 示例
```java
public interface ISysUserMapper extends BaseMapperPlus<SysUser, SysUserVo> {
    // ✅ 只定义数据访问方法
    List<SysUserVo> selectUserList(SysUserBo bo);
}
```

---

## Domain 层

### Entity (实体类)
对应数据库表，使用 MyBatis-Plus 注解
```java
@TableName("sys_user")
public class SysUser extends BaseEntity {
    @TableId(type = IdType.AUTO)
    private Long userId;
    private String userName;
}
```

### Bo (Business Object)
接收前端参数，包含校验注解
```java
public class SysUserBo extends BaseEntity {
    @NotBlank(message = "用户名不能为空")
    private String userName;
}
```

### Vo (View Object)
返回给前端的数据
```java
public class SysUserVo implements Serializable {
    private Long userId;
    private String userName;
    private List<SysRoleVo> roles;  // 可包含关联数据
}
```

---

## 禁止的跨层调用

```java
// ❌ Controller 直接调用 Mapper
@RestController
public class BadController {
    @Autowired
    private ISysUserMapper userMapper;  // 错误！

    @GetMapping("/list")
    public R list() {
        return R.ok(userMapper.selectList(null));  // 错误！
    }
}

// ❌ Mapper 调用 Service
public interface BadMapper extends BaseMapperPlus<SysUser, SysUserVo> {
    default void badMethod() {
        // 错误！Mapper不应有业务逻辑
        userService.doSomething();
    }
}
```

---

## 依赖方向

```
Controller → Service → Mapper → Database
   ↓           ↓
  Bo/Vo      Entity
```

**原则**: 只能向下依赖，不能向上依赖或跨层依赖
