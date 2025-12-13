# 事务管理最佳实践

## 核心原则

### 黄金法则
```
事务应该尽可能小，只包裹必须保证原子性的操作
```

---

## 基础使用

### 标准注解
```java
@Transactional(rollbackFor = Exception.class)
public void saveUser(SysUserBo bo) {
    // 数据库操作
}
```

**必须参数**: `rollbackFor = Exception.class`
**原因**: Spring 默认只回滚 RuntimeException，加上此参数可捕获所有异常

---

## 事务传播行为

### REQUIRED (默认)
```java
@Transactional(propagation = Propagation.REQUIRED)
public void methodA() {
    methodB();  // 加入 methodA 的事务
}
```
如果当前存在事务，则加入该事务；如果不存在，则创建新事务

### REQUIRES_NEW
```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void methodB() {
    // 始终开启新事务，挂起外部事务
}
```
**使用场景**: 记录日志、发送通知等，即使主业务失败也要成功

### NOT_SUPPORTED
```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void readOnlyMethod() {
    // 以非事务方式执行
}
```
**使用场景**: 纯查询操作，不需要事务

---

## 避免大事务

### ❌ 错误示例
```java
@Transactional(rollbackFor = Exception.class)
public void bigTransaction() {
    // 1. 查询大量数据
    List<User> users = userMapper.selectList(null);

    // 2. 复杂计算 (耗时)
    for (User user : users) {
        complexCalculation(user);
    }

    // 3. 外部调用 (不可控)
    thirdPartyService.notify(users);

    // 4. 数据库操作
    userMapper.batchUpdate(users);
}
```

**问题**:
- 事务时间长，锁表时间长
- 外部调用失败导致整体回滚
- 并发性能差

### ✅ 正确示例
```java
public void optimizedTransaction() {
    // 1. 查询 (无事务)
    List<User> users = userMapper.selectList(null);

    // 2. 复杂计算 (无事务)
    for (User user : users) {
        complexCalculation(user);
    }

    // 3. 只在必要时开启事务
    batchUpdateWithTransaction(users);

    // 4. 异步通知 (无事务)
    asyncNotify(users);
}

@Transactional(rollbackFor = Exception.class)
private void batchUpdateWithTransaction(List<User> users) {
    // 仅数据库操作在事务内
    userMapper.batchUpdate(users);
}
```

---

## 事务失效场景

### 1. 方法不是 public
```java
// ❌ 事务失效
@Transactional
private void saveUser() { }

// ✅ 正确
@Transactional
public void saveUser() { }
```

### 2. 同类内部调用
```java
public class UserService {
    // ❌ 事务失效
    public void methodA() {
        this.methodB();  // 直接调用，事务失效
    }

    @Transactional
    public void methodB() { }

    // ✅ 解决方案1: 注入自身
    @Autowired
    private UserService self;

    public void methodA() {
        self.methodB();  // 通过代理调用
    }

    // ✅ 解决方案2: 使用 AopContext
    public void methodA() {
        ((UserService) AopContext.currentProxy()).methodB();
    }
}
```

### 3. 异常被捕获
```java
// ❌ 事务失效
@Transactional
public void saveUser() {
    try {
        userMapper.insert(user);
    } catch (Exception e) {
        // 异常被吞了，事务不会回滚
        log.error("error", e);
    }
}

// ✅ 正确
@Transactional(rollbackFor = Exception.class)
public void saveUser() {
    try {
        userMapper.insert(user);
    } catch (Exception e) {
        log.error("error", e);
        throw new ServiceException("保存失败");  // 重新抛出
    }
}
```

---

## 嵌套事务

### 场景示例
```java
@Service
public class OrderService {
    @Autowired
    private LogService logService;

    @Transactional(rollbackFor = Exception.class)
    public void createOrder(Order order) {
        // 1. 保存订单
        orderMapper.insert(order);

        // 2. 记录日志 (独立事务，即使订单失败也要记录)
        logService.saveLog(order);

        // 3. 扣减库存
        stockService.deduct(order.getProductId(), order.getQuantity());
    }
}

@Service
public class LogService {
    @Transactional(propagation = Propagation.REQUIRES_NEW,
                   rollbackFor = Exception.class)
    public void saveLog(Order order) {
        // 开启新事务，独立提交
        logMapper.insert(new Log(order));
    }
}
```

---

## 分布式事务

### Seata AT 模式
```java
@GlobalTransactional(rollbackFor = Exception.class)
public void distributedTransaction() {
    // 订单服务
    orderService.createOrder(order);

    // 库存服务 (远程调用)
    stockService.deduct(order.getProductId());

    // 积分服务 (远程调用)
    pointsService.add(order.getUserId(), order.getAmount());
}
```

---

## Checklist

开启事务前确认:
- [ ] 是否真的需要事务？(只读操作不需要)
- [ ] 事务范围是否最小？
- [ ] 是否包含外部调用？(应移出事务)
- [ ] 是否有长时间计算？(应移出事务)
- [ ] 异常是否会被正确抛出？
- [ ] 是否为 public 方法？
- [ ] 是否通过代理调用？
