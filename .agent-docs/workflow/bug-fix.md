# 后端 Bug 修复流程

## 概述
后端 Bug 的影响范围更大，可能涉及数据一致性、安全性、性能等关键问题，修复时必须格外谨慎。

---

## Bug 分类

### 1. 逻辑错误
- 业务逻辑实现错误
- 条件判断错误
- 数据计算错误

### 2. 性能问题
- SQL 查询慢
- 内存泄漏
- 高并发下性能下降

### 3. 数据异常
- 数据不一致
- 事务回滚失败
- 脏数据产生

### 4. 安全漏洞
- SQL 注入
- 权限绕过
- 敏感信息泄露

---

## 定位步骤

### Step 1: 日志分析
```bash
# 查看应用日志
tail -f logs/ruoyi.log

# 查看错误日志
grep "ERROR" logs/ruoyi-error.log

# 查看 SQL 日志（如开启）
grep "==>  Preparing:" logs/ruoyi.log
```

**关键信息**：
- 异常堆栈信息
- SQL 执行记录
- 请求参数
- 用户 ID 和租户 ID

### Step 2: 断点调试
1. 在可疑代码处设置断点
2. 模拟问题场景
3. 逐步跟踪变量值
4. 检查方法调用链

### Step 3: SQL 分析
```sql
-- 查看执行计划
EXPLAIN SELECT * FROM sys_user WHERE ...;

-- 检查索引使用情况
SHOW INDEX FROM sys_user;

-- 分析慢查询日志
```

---

## 修复规范

### 事务完整性检查
```java
// ❌ 错误：事务边界不清晰
public void updateUser(SysUserBo user) {
    userMapper.updateById(user);  // 无事务保护
    roleMapper.deleteByUserId(user.getUserId());
    roleMapper.insertUserRole(user.getUserId(), user.getRoleIds());
}

// ✅ 正确：使用事务注解
@Transactional(rollbackFor = Exception.class)
public void updateUser(SysUserBo user) {
    userMapper.updateById(user);
    roleMapper.deleteByUserId(user.getUserId());
    roleMapper.insertUserRole(user.getUserId(), user.getRoleIds());
}
```

### 数据一致性检查
```java
// ✅ 修复前先检查数据状态
public void deleteUser(Long userId) {
    // 检查是否有关联数据
    if (orderService.countByUserId(userId) > 0) {
        throw new ServiceException("该用户存在订单记录，无法删除");
    }
    userMapper.deleteById(userId);
}
```

### 权限检查
```java
// ✅ 确保权限注解存在
@SaCheckPermission("system:user:remove")
@Log(title = "用户管理", businessType = BusinessType.DELETE)
public R<Void> remove(@PathVariable Long[] userIds) {
    // 检查数据权限
    userService.checkUserDataScope(userIds);
    return toAjax(userService.deleteUserByIds(userIds));
}
```

---

## 回归测试

### 功能测试
- [ ] 修复的功能正常
- [ ] 相关业务不受影响
- [ ] 边界条件处理正确

### 数据验证
- [ ] 数据库数据一致
- [ ] 事务正确回滚/提交
- [ ] 无脏数据产生

### 性能测试
- [ ] SQL 执行时间正常
- [ ] 内存占用合理
- [ ] 并发情况下稳定

---

## 常见问题与解决方案

### 事务问题
| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 事务不回滚 | `@Transactional` 只捕获 RuntimeException | 添加 `rollbackFor = Exception.class` |
| 事务失效 | 方法内部调用 | 使用 `AopContext.currentProxy()` 或注入自身 |
| 大事务超时 | 事务范围过大 | 拆分事务，只在必要处使用 |

### SQL 性能问题
| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 查询慢 | 缺少索引 | 添加合适的索引 |
| N+1 查询 | 循环调用 Mapper | 使用批量查询或 JOIN |
| 全表扫描 | WHERE 条件未使用索引 | 优化 WHERE 条件 |

### 权限问题
| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 权限绕过 | 缺少权限注解 | 添加 `@SaCheckPermission` |
| 数据越权 | 未检查数据权限 | 使用 `checkDataScope` 方法 |
| 租户隔离失效 | 未使用租户拦截器 | 检查多租户配置 |
