# 权限控制最佳实践

## RuoYi-Vue-Plus 权限体系

### 三层权限控制
1. **接口权限**: 通过 `@SaCheckPermission` 控制
2. **数据权限**: 通过自定义注解和拦截器控制
3. **字段权限**: 敏感字段脱敏

---

## 接口权限 (@SaCheckPermission)

### 基础使用
```java
@RestController
@RequestMapping("/system/user")
public class SysUserController {

    // 单个权限
    @SaCheckPermission("system:user:query")
    @GetMapping("/list")
    public R<List<SysUserVo>> list() { }

    // AND 逻辑 (同时拥有两个权限)
    @SaCheckPermission(value = {"system:user:query", "system:user:export"}, mode = SaMode.AND)
    @PostMapping("/export")
    public void export() { }

    // OR 逻辑 (拥有任一权限即可)
    @SaCheckPermission(value = {"system:user:query", "system:dept:query"}, mode = SaMode.OR)
    @GetMapping("/all")
    public R<List<SysUserVo>> all() { }
}
```

### 权限命名规范
```
{模块}:{功能}:{操作}

示例:
- system:user:query   # 用户查询
- system:user:add     # 用户新增
- system:user:edit    # 用户修改
- system:user:remove  # 用户删除
- system:user:export  # 用户导出
- system:user:import  # 用户导入
```

---

## 数据权限

### 使用方式
```java
@Service
public class SysUserServiceImpl implements ISysUserService {

    @Override
    @DataScope(deptAlias = "d", userAlias = "u")
    public List<SysUserVo> selectUserList(SysUserBo user) {
        // 查询时会自动拼接数据权限 SQL
        return baseMapper.selectUserList(user);
    }
}
```

### 数据权限级别
1. **全部数据权限**: 不限制
2. **自定义数据权限**: 指定部门
3. **本部门数据权限**: 仅本部门
4. **本部门及以下数据权限**: 本部门及子部门
5. **仅本人数据权限**: 只能查看自己的数据

### 手动检查数据权限
```java
@Service
@RequiredArgsConstructor
public class SysUserServiceImpl implements ISysUserService {

    private final ISysDataScopeService dataScopeService;

    public void updateUser(Long userId) {
        // 检查是否有数据权限
        dataScopeService.checkUserDataScope(userId);

        // 有权限才能操作
        userMapper.updateById(user);
    }
}
```

---

## 多租户隔离

### 自动隔离
RuoYi-Plus 通过 MyBatis 拦截器自动为 SQL 添加租户条件：
```sql
-- 原始 SQL
SELECT * FROM sys_user WHERE status = '0'

-- 自动添加租户过滤
SELECT * FROM sys_user WHERE tenant_id = '000000' AND status = '0'
```

### 跨租户查询
```java
// 需要查询所有租户数据时
TenantHelper.ignore(() -> {
    return userMapper.selectList(null);  // 忽略租户隔离
});

// Lambda 方式
List<SysUser> users = TenantHelper.ignore(() ->
    userMapper.selectList(null)
);
```

### 超级管理员判断
```java
if (LoginHelper.isSuperAdmin()) {
    // 超级管理员逻辑
    TenantHelper.clearDynamic();  // 清除动态租户
}
```

---

## 角色权限设计

### RBAC 模型
```
User (用户) → UserRole (用户角色关联) → Role (角色) → RoleMenu (角色菜单关联) → Menu (菜单/权限)
```

### 示例
```java
// 检查用户是否有某个角色
@SaCheckRole("admin")
@GetMapping("/admin")
public R adminOnly() { }

// 检查是否有某个权限
@SaCheckPermission("system:user:remove")
@DeleteMapping("/{ids}")
public R remove(@PathVariable Long[] ids) { }

// 代码中手动检查
if (StpUtil.hasPermission("system:user:add")) {
    // 有权限的逻辑
}
```

---

## 常见权限场景

### 1. 用户只能操作自己的数据
```java
@Override
public void updateProfile(SysUserBo bo) {
    Long currentUserId = LoginHelper.getUserId();

    if (!currentUserId.equals(bo.getUserId())) {
        throw new ServiceException("无权修改他人信息");
    }

    userMapper.updateById(MapstructUtils.convert(bo, SysUser.class));
}
```

### 2. 防止越权操作
```java
@Override
public void deleteUser(Long userId) {
    // 1. 检查是否是超管 (超管不能删)
    if (LoginHelper.isSuperAdmin(userId)) {
        throw new ServiceException("不能删除超级管理员");
    }

    // 2. 检查是否是自己 (不能删自己)
    if (LoginHelper.getUserId().equals(userId)) {
        throw new ServiceException("不能删除自己");
    }

    // 3. 检查数据权限
    checkUserDataScope(userId);

    // 4. 执行删除
    userMapper.deleteById(userId);
}
```

### 3. 部门负责人权限
```java
public void updateDept(SysDeptBo bo) {
    // 检查是否有权限修改该部门
    checkDeptDataScope(bo.getDeptId());

    deptMapper.updateById(MapstructUtils.convert(bo, SysDept.class));
}

private void checkDeptDataScope(Long deptId) {
    SysDept dept = deptMapper.selectById(deptId);
    if (dept == null) {
        throw new ServiceException("部门不存在");
    }

    // 检查是否是本部门或子部门
    if (!dataScopeService.checkDeptDataScope(deptId)) {
        throw new ServiceException("没有权限访问该部门数据");
    }
}
```

---

## 安全检查清单

每个需要权限控制的接口确认:
- [ ] 是否添加 `@SaCheckPermission` 注解？
- [ ] 权限字符串是否符合命名规范？
- [ ] 是否检查数据权限？
- [ ] 是否防止越权操作？
- [ ] 是否考虑多租户隔离？
- [ ] 超级管理员是否有特殊处理？
- [ ] 是否有敏感操作日志？
