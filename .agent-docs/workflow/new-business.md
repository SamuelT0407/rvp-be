# 创建全新业务模块

## 概述
从零创建新业务模块时，严格按照以下顺序执行，确保架构清晰、代码规范。

---

## Phase 1: 数据库设计

### Step 1.1: 设计表结构

```sql
-- 示例：创建商品管理表
CREATE TABLE biz_product (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键',
    product_code VARCHAR(50) NOT NULL COMMENT '商品编码',
    product_name VARCHAR(200) NOT NULL COMMENT '商品名称',
    category_id BIGINT COMMENT '分类ID ',
    price DECIMAL(10,2) NOT NULL COMMENT '价格',
    stock INT DEFAULT 0 COMMENT '库存',
    status CHAR(1) DEFAULT '0' COMMENT '状态(0正常 1停用)',
    remark VARCHAR(500) COMMENT '备注',

    create_dept BIGINT COMMENT '创建部门',
    create_by BIGINT COMMENT '创建者',
    create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    update_by BIGINT COMMENT '更新者',
    update_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',

    UNIQUE KEY uk_product_code (product_code),
    KEY idx_category_id (category_id),
    KEY idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品表';
```

### Step 1.2: 字段设计规范
- **主键**: 统一使用 `BIGINT` 自增
- **必备字段**: `create_dept`, `create_by`, `create_time`, `update_by`, `update_time`
- **逻辑删除**: 如需软删除，添加 `del_flag CHAR(1) DEFAULT '0'`
- **多租户**: 如需租户隔离，添加 `tenant_id VARCHAR(20)`
- **状态字段**: 统一用 `CHAR(1)`, '0'正常, '1'停用
- **时间字段**: 使用 `DATETIME` 类型

---

## Phase 2: 创建模块结构

### Step 2.1: 创建目录
```
ruoyi-modules/
└── ruoyi-{business}/           # 例如: ruoyi-product
    ├── pom.xml
    └── src/main/java/org/dromara/{business}/
        ├── controller/         # Controller 层
        ├── service/           # Service 接口
        │   └── impl/          # Service 实现
        ├── mapper/            # Mapper 接口
        ├── domain/            # Domain 对象
        │   ├── bo/            # Business Object
        │   ├── vo/            # View Object
        │   └── entity/        # 实体类 (或直接放 domain 下)
        └── listener/          # 监听器 (如导入监听器)
    └── src/main/resources/
        └── mapper/            # MyBatis XML
```

### Step 2.2: 配置 pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <parent>
        <artifactId>ruoyi-modules</artifactId>
        <groupId>org.dromara</groupId>
        <version>${revision}</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>ruoyi-product</artifactId>
    <description>商品管理模块</description>

    <dependencies>
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>ruoyi-common-mybatis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>ruoyi-common-web</artifactId>
        </dependency>
    </dependencies>
</project>
```

---

## Phase 3: Domain 层

### Step 3.1: 创建实体类
```java
package org.dromara.product.domain;

import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;
import lombok.EqualsAndHashCode;
import org.dromara.common.mybatis.core.domain.BaseEntity;
import java.math.BigDecimal;

/**
 * 商品对象 biz_product
 */
@Data
@EqualsAndHashCode(callSuper = true)
@TableName("biz_product")
public class BizProduct extends BaseEntity {

    @TableId(value = "id", type = IdType.AUTO)
    private Long id;

    private String productCode;

    private String productName;

    private Long categoryId;

    private BigDecimal price;

    private Integer stock;

    private String status;

    private String remark;
}
```

### Step 3.2: 创建 BO
```java
package org.dromara.product.domain.bo;

import io.github.linpeilie.annotations.AutoMapper;
import jakarta.validation.constraints.*;
import lombok.Data;
import lombok.EqualsAndHashCode;
import org.dromara.common.mybatis.core.domain.BaseEntity;
import org.dromara.product.domain.BizProduct;
import java.math.BigDecimal;

/**
 * 商品业务对象
 */
@Data
@EqualsAndHashCode(callSuper = true)
@AutoMapper(target = BizProduct.class, reverseConvertGenerate = false)
public class BizProductBo extends BaseEntity {

    @NotNull(message = "ID不能为空", groups = {Edit.class})
    private Long id;

    @NotBlank(message = "商品编码不能为空")
    @Size(max = 50, message = "商品编码长度不能超过50")
    private String productCode;

    @NotBlank(message = "商品名称不能为空")
    @Size(max = 200, message = "商品名称长度不能超过200")
    private String productName;

    @NotNull(message = "价格不能为空")
    @DecimalMin(value = "0.01", message = "价格必须大于0")
    private BigDecimal price;

    private Integer stock;

    private String status;

    private String remark;
}
```

### Step 3.3: 创建 VO
```java
package org.dromara.product.domain.vo;

import com.alibaba.excel.annotation.ExcelProperty;
import io.github.linpeilie.annotations.AutoMapper;
import lombok.Data;
import org.dromara.product.domain.BizProduct;
import java.io.Serial;
import java.io.Serializable;
import java.math.BigDecimal;

/**
 * 商品视图对象
 */
@Data
@AutoMapper(target = BizProduct.class)
public class BizProductVo implements Serializable {

    @Serial
    private static final long serialVersionUID = 1L;

    @ExcelProperty(value = "ID")
    private Long id;

    @ExcelProperty(value = "商品编码")
    private String productCode;

    @ExcelProperty(value = "商品名称")
    private String productName;

    @ExcelProperty(value = "价格")
    private BigDecimal price;

    @ExcelProperty(value = "库存")
    private Integer stock;

    @ExcelProperty(value = "状态")
    private String status;
}
```

---

## Phase 4: Mapper 层

### Step 4.1: 创建 Mapper 接口
```java
package org.dromara.product.mapper;

import org.dromara.common.mybatis.core.mapper.BaseMapperPlus;
import org.dromara.product.domain.BizProduct;
import org.dromara.product.domain.vo.BizProductVo;

/**
 * 商品Mapper接口
 */
public interface BizProductMapper extends BaseMapperPlus<BizProduct, BizProductVo> {

}
```

### Step 4.2: 创建 Mapper XML (如需自定义 SQL)
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="org.dromara.product.mapper.BizProductMapper">

    <!-- 自定义 SQL 示例 -->
    <select id="selectProductListWithCategory" resultType="BizProductVo">
        SELECT p.*, c.category_name
        FROM biz_product p
        LEFT JOIN biz_category c ON p.category_id = c.id
        WHERE p.del_flag = '0'
    </select>

</mapper>
```

---

## Phase 5: Service 层

### Step 5.1: 创建 Service 接口
```java
package org.dromara.product.service;

import org.dromara.common.mybatis.core.page.PageQuery;
import org.dromara.common.mybatis.core.page.TableDataInfo;
import org.dromara.common.mybatis.core.service.IServicePlus;
import org.dromara.product.domain.BizProduct;
import org.dromara.product.domain.bo.BizProductBo;
import org.dromara.product.domain.vo.BizProductVo;

/**
 * 商品Service接口
 */
public interface IBizProductService extends IServicePlus<BizProduct, BizProductVo> {

    /**
     * 查询分页
     */
    TableDataInfo<BizProductVo> selectPageProductList(BizProductBo bo, PageQuery pageQuery);

    /**
     * 新增商品
     */
    Boolean insertProduct(BizProductBo bo);

    /**
     * 修改商品
     */
    Boolean updateProduct(BizProductBo bo);

    /**
     * 删除商品
     */
    Boolean deleteProductByIds(Long[] ids);
}
```

### Step 5.2: 创建 Service 实现
```java
package org.dromara.product.service.impl;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import lombok.RequiredArgsConstructor;
import org.dromara.common.mybatis.core.page.PageQuery;
import org.dromara.common.mybatis.core.page.TableDataInfo;
import org.dromara.common.mybatis.core.service.ServicePlusImpl;
import org.dromara.product.domain.BizProduct;
import org.dromara.product.domain.bo.BizProductBo;
import org.dromara.product.domain.vo.BizProductVo;
import org.dromara.product.mapper.BizProductMapper;
import org.dromara.product.service.IBizProductService;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * 商品Service实现
 */
@Service
@RequiredArgsConstructor
public class BizProductServiceImpl extends ServicePlusImpl<BizProductMapper, BizProduct, BizProductVo>
    implements IBizProductService {

    @Override
    public TableDataInfo<BizProductVo> selectPageProductList(BizProductBo bo, PageQuery pageQuery) {
        LambdaQueryWrapper<BizProduct> lqw = buildQueryWrapper(bo);
        return baseMapper.selectVoPage(pageQuery.build(), lqw);
    }

    private LambdaQueryWrapper<BizProduct> buildQueryWrapper(BizProductBo bo) {
        return new LambdaQueryWrapper<BizProduct>()
            .like(StringUtils.isNotBlank(bo.getProductName()), BizProduct::getProductName, bo.getProductName())
            .eq(StringUtils.isNotBlank(bo.getStatus()), BizProduct::getStatus, bo.getStatus());
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public Boolean insertProduct(BizProductBo bo) {
        BizProduct entity = MapstructUtils.convert(bo, BizProduct.class);
        return baseMapper.insert(entity) > 0;
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public Boolean updateProduct(BizProductBo bo) {
        BizProduct entity = MapstructUtils.convert(bo, BizProduct.class);
        return baseMapper.updateById(entity) > 0;
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public Boolean deleteProductByIds(Long[] ids) {
        return baseMapper.deleteByIds(List.of(ids)) > 0;
    }
}
```

---

## Phase 6: Controller 层

```java
package org.dromara.product.controller;

import cn.dev33.satoken.annotation.SaCheckPermission;
import jakarta.validation.constraints.NotNull;
import lombok.RequiredArgsConstructor;
import org.dromara.common.core.domain.R;
import org.dromara.common.log.annotation.Log;
import org.dromara.common.log.enums.BusinessType;
import org.dromara.common.mybatis.core.page.PageQuery;
import org.dromara.common.mybatis.core.page.TableDataInfo;
import org.dromara.common.web.core.BaseController;
import org.dromara.product.domain.bo.BizProductBo;
import org.dromara.product.domain.vo.BizProductVo;
import org.dromara.product.service.IBizProductService;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

/**
 * 商品管理Controller
 */
@Validated
@RequiredArgsConstructor
@RestController
@RequestMapping("/product/product")
public class BizProductController extends BaseController {

    private final IBizProductService productService;

    /**
     * 查询列表
     */
    @SaCheckPermission("product:product:list")
    @GetMapping("/list")
    public TableDataInfo<BizProductVo> list(BizProductBo bo, PageQuery pageQuery) {
        return productService.selectPageProductList(bo, pageQuery);
    }

    /**
     * 获取详情
     */
    @SaCheckPermission("product:product:query")
    @GetMapping("/{id}")
    public R<BizProductVo> getInfo(@PathVariable @NotNull Long id) {
        return R.ok(productService.selectVoById(id));
    }

    /**
     * 新增
     */
    @SaCheckPermission("product:product:add")
    @Log(title = "商品管理", businessType = BusinessType.INSERT)
    @PostMapping()
    public R<Void> add(@Validated @RequestBody BizProductBo bo) {
        return toAjax(productService.insertProduct(bo));
    }

    /**
     * 修改
     */
    @SaCheckPermission("product:product:edit")
    @Log(title = "商品管理", businessType = BusinessType.UPDATE)
    @PutMapping()
    public R<Void> edit(@Validated @RequestBody BizProductBo bo) {
        return toAjax(productService.updateProduct(bo));
    }

    /**
     * 删除
     */
    @SaCheckPermission("product:product:remove")
    @Log(title = "商品管理", businessType = BusinessType.DELETE)
    @DeleteMapping("/{ids}")
    public R<Void> remove(@PathVariable Long[] ids) {
        return toAjax(productService.deleteProductByIds(ids));
    }
}
```

---

## Phase 7: 配置权限

### 菜单 SQL
```sql
-- 添加菜单
INSERT INTO sys_menu (menu_name, parent_id, order_num, path, component,
                      is_frame, is_cache, menu_type, visible, status,
                      perms, icon, create_by, create_time)
VALUES ('商品管理', 0, 4, 'product', null, 1, 0, 'M', '0', '0',
        null, 'shopping', 'admin', sysdate());

-- 添加按钮权限
INSERT INTO sys_menu VALUES (NULL, '商品查询', @menuId, 1, '', '', 1, 0, 'F', '0', '0', 'product:product:query', '#', 'admin', sysdate(), '', NULL, '');
INSERT INTO sys_menu VALUES (NULL, '商品新增', @menuId, 2, '', '', 1, 0, 'F', '0', '0', 'product:product:add', '#', 'admin', sysdate(), '', NULL, '');
INSERT INTO sys_menu VALUES (NULL, '商品修改', @menuId, 3, '', '', 1, 0, 'F', '0', '0', 'product:product:edit', '#', 'admin', sysdate(), '', NULL, '');
INSERT INTO sys_menu VALUES (NULL, '商品删除', @menuId, 4, '', '', 1, 0, 'F', '0', '0', 'product:product:remove', '#', 'admin', sysdate(), '', NULL, '');
INSERT INTO sys_menu VALUES (NULL, '商品导出', @menuId, 5, '', '', 1, 0, 'F', '0', '0', 'product:product:export', '#', 'admin', sysdate(), '', NULL, '');
```

---

## Checklist

- [ ] 数据库表已创建
- [ ] Domain 对象 (Entity/Bo/Vo) 已创建
- [ ] Mapper 接口已创建
- [ ] Mapper XML 已创建 (如需要)
- [ ] Service 接口和实现已创建
- [ ] Controller 已创建
- [ ] 权限注解已添加
- [ ] 日志注解已添加
- [ ] 菜单和权限已配置
- [ ] 单元测试已编写
