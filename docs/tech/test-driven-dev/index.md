# TDD - 测试驱动开发

## 🎯 TDD 的核心思想

**先写测试，再写代码，最后重构**

口诀：**红 → 绿 → 重构**
- 🔴 **红**：写一个失败的测试
- 🟢 **绿**：写最少的代码让测试通过
- 🔵 **重构**：优化代码，保持测试通过

---

## 📝 实战案例：开发一个"任务标签"功能

假设我们要开发一个新功能：**给任务添加标签（Tags）**

### 需求描述：
- 用户可以为任务创建标签
- 标签有名称和颜色
- 一个任务可以有多个标签
- 标签可以被多个任务共享
- 可以通过标签筛选任务

---

## 🔄 TDD 完整流程演示

### 第一轮：创建标签功能

#### **步骤 1：🔴 写第一个失败的测试**

创建测试文件 `domains/task-tag/tests/task-tag.test.ts`：

```typescript
import { describe, expect, test } from 'vitest';
import { MockContext } from '@bika/domains/__tests__/mock';
import { TaskTagSO } from '../server/task-tag-so';

describe('任务标签功能 - 创建标签', async () => {
  const { user, space } = await MockContext.initUserContext();
  
  test('应该能创建一个标签', async () => {
    // 准备
    const tagData = {
      name: '紧急',
      color: '#FF0000',
      spaceId: space.id,
    };
    
    // 执行
    const tag = await TaskTagSO.create(user, tagData);
    
    // 断言
    expect(tag).toBeDefined();
    expect(tag.name).toBe('紧急');
    expect(tag.color).toBe('#FF0000');
    expect(tag.spaceId).toBe(space.id);
    expect(tag.createdBy).toBe(user.id);
  });
});
```

**运行测试：**
```bash
npx vitest task-tag.test.ts
```

**结果：** ❌ 测试失败（因为 `TaskTagSO` 还不存在）

```
Error: Cannot find module '../server/task-tag-so'
```

---

#### **步骤 2：🟢 写最少的代码让测试通过**

创建 `domains/task-tag/server/task-tag-so.ts`：

```typescript
import { db } from '@bika/server-orm';
import type { UserSO } from '@bika/domains/user/server/user-so';

export class TaskTagSO {
  id: string;
  name: string;
  color: string;
  spaceId: string;
  createdBy: string;
  
  constructor(model: any) {
    this.id = model.id;
    this.name = model.name;
    this.color = model.color;
    this.spaceId = model.spaceId;
    this.createdBy = model.createdBy;
  }
  
  static async create(
    user: UserSO, 
    data: { name: string; color: string; spaceId: string }
  ): Promise<TaskTagSO> {
    // 最简单的实现
    const model = await db.prisma.taskTag.create({
      data: {
        name: data.name,
        color: data.color,
        spaceId: data.spaceId,
        createdBy: user.id,
      },
    });
    
    return new TaskTagSO(model);
  }
}
```

创建数据库模型（Prisma Schema）：

```prisma
// packages/bika-server-orm/prisma/schema.prisma

model TaskTag {
  id        String   @id @default(cuid())
  name      String
  color     String
  spaceId   String
  createdBy String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  @@map("task_tags")
}
```

运行数据库迁移：
```bash
pnpm prisma migrate dev --name add_task_tags
```

**再次运行测试：**
```bash
npx vitest task-tag.test.ts
```

**结果：** ✅ 测试通过！

---

#### **步骤 3：🔵 重构（如果需要）**

目前代码很简单，暂时不需要重构。继续下一个功能。

---

### 第二轮：验证标签名称

#### **步骤 1：🔴 添加验证测试**

在同一个测试文件中添加：

```typescript
test('创建标签时名称不能为空', async () => {
  await expect(
    TaskTagSO.create(user, {
      name: '',  // 空名称
      color: '#FF0000',
      spaceId: space.id,
    })
  ).rejects.toThrow('标签名称不能为空');
});

test('创建标签时颜色必须是有效的十六进制', async () => {
  await expect(
    TaskTagSO.create(user, {
      name: '紧急',
      color: 'invalid-color',  // 无效颜色
      spaceId: space.id,
    })
  ).rejects.toThrow('颜色格式无效');
});
```

**运行测试：** ❌ 失败（没有验证逻辑）

---

#### **步骤 2：🟢 添加验证逻辑**

```typescript
import { ServerError, errors } from '@bika/contents/config/server/error';

export class TaskTagSO {
  // ... 之前的代码 ...
  
  static async create(
    user: UserSO, 
    data: { name: string; color: string; spaceId: string }
  ): Promise<TaskTagSO> {
    // 验证名称
    if (!data.name || data.name.trim() === '') {
      throw new ServerError({
        code: 'INVALID_TAG_NAME',
        message: '标签名称不能为空',
      });
    }
    
    // 验证颜色格式
    const colorRegex = /^#[0-9A-F]{6}$/i;
    if (!colorRegex.test(data.color)) {
      throw new ServerError({
        code: 'INVALID_TAG_COLOR',
        message: '颜色格式无效',
      });
    }
    
    const model = await db.prisma.taskTag.create({
      data: {
        name: data.name.trim(),
        color: data.color.toUpperCase(),
        spaceId: data.spaceId,
        createdBy: user.id,
      },
    });
    
    return new TaskTagSO(model);
  }
}
```

**运行测试：** ✅ 通过！

---

#### **步骤 3：🔵 重构**

提取验证逻辑到单独的方法：

```typescript
export class TaskTagSO {
  // ... 之前的代码 ...
  
  private static validateName(name: string): void {
    if (!name || name.trim() === '') {
      throw new ServerError({
        code: 'INVALID_TAG_NAME',
        message: '标签名称不能为空',
      });
    }
  }
  
  private static validateColor(color: string): void {
    const colorRegex = /^#[0-9A-F]{6}$/i;
    if (!colorRegex.test(color)) {
      throw new ServerError({
        code: 'INVALID_TAG_COLOR',
        message: '颜色格式无效',
      });
    }
  }
  
  static async create(
    user: UserSO, 
    data: { name: string; color: string; spaceId: string }
  ): Promise<TaskTagSO> {
    // 使用提取的验证方法
    this.validateName(data.name);
    this.validateColor(data.color);
    
    const model = await db.prisma.taskTag.create({
      data: {
        name: data.name.trim(),
        color: data.color.toUpperCase(),
        spaceId: data.spaceId,
        createdBy: user.id,
      },
    });
    
    return new TaskTagSO(model);
  }
}
```

**运行测试：** ✅ 仍然通过！重构成功。

---

### 第三轮：给任务添加标签

#### **步骤 1：🔴 写测试**

```typescript
describe('任务标签功能 - 关联任务', async () => {
  const { user, space, member, rootFolder } = await MockContext.initUserContext();
  
  test('应该能给任务添加标签', async () => {
    // 创建一个任务（假设使用 Database）
    const taskDB = await rootFolder.createChildSimple(user, {
      name: '任务列表',
      resourceType: 'DATABASE',
    });
    const database = await taskDB.toResourceSO<DatabaseSO>();
    const record = await database.createRecord(user, member, {
      name: '完成项目文档',
    });
    
    // 创建标签
    const tag = await TaskTagSO.create(user, {
      name: '紧急',
      color: '#FF0000',
      spaceId: space.id,
    });
    
    // 给任务添加标签
    await record.addTag(tag.id);
    
    // 验证
    const tags = await record.getTags();
    expect(tags.length).toBe(1);
    expect(tags[0].id).toBe(tag.id);
    expect(tags[0].name).toBe('紧急');
  });
  
  test('一个任务可以有多个标签', async () => {
    const taskDB = await rootFolder.createChildSimple(user, {
      name: '任务列表',
      resourceType: 'DATABASE',
    });
    const database = await taskDB.toResourceSO<DatabaseSO>();
    const record = await database.createRecord(user, member, {
      name: '完成项目文档',
    });
    
    // 创建多个标签
    const urgentTag = await TaskTagSO.create(user, {
      name: '紧急',
      color: '#FF0000',
      spaceId: space.id,
    });
    
    const importantTag = await TaskTagSO.create(user, {
      name: '重要',
      color: '#FFA500',
      spaceId: space.id,
    });
    
    // 添加多个标签
    await record.addTag(urgentTag.id);
    await record.addTag(importantTag.id);
    
    // 验证
    const tags = await record.getTags();
    expect(tags.length).toBe(2);
    expect(tags.map(t => t.name)).toContain('紧急');
    expect(tags.map(t => t.name)).toContain('重要');
  });
});
```

**运行测试：** ❌ 失败（`record.addTag` 方法不存在）

---

#### **步骤 2：🟢 实现功能**

首先更新数据库模型：

```prisma
model TaskTag {
  id        String   @id @default(cuid())
  name      String
  color     String
  spaceId   String
  createdBy String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  // 关联关系
  records   RecordTag[]
  
  @@map("task_tags")
}

model RecordTag {
  id        String   @id @default(cuid())
  recordId  String
  tagId     String
  createdAt DateTime @default(now())
  
  tag       TaskTag  @relation(fields: [tagId], references: [id], onDelete: Cascade)
  
  @@unique([recordId, tagId])
  @@map("record_tags")
}
```

扩展 RecordSO 类（假设在 `domains/database/server/record-so.ts`）：

```typescript
export class RecordSO {
  // ... 现有代码 ...
  
  async addTag(tagId: string): Promise<void> {
    await db.prisma.recordTag.create({
      data: {
        recordId: this.id,
        tagId: tagId,
      },
    });
  }
  
  async getTags(): Promise<TaskTagSO[]> {
    const recordTags = await db.prisma.recordTag.findMany({
      where: { recordId: this.id },
      include: { tag: true },
    });
    
    return recordTags.map(rt => new TaskTagSO(rt.tag));
  }
  
  async removeTag(tagId: string): Promise<void> {
    await db.prisma.recordTag.deleteMany({
      where: {
        recordId: this.id,
        tagId: tagId,
      },
    });
  }
}
```

**运行测试：** ✅ 通过！

---

#### **步骤 3：🔵 重构**

可以考虑：
- 添加批量操作方法
- 缓存标签数据
- 添加索引优化查询

暂时不需要大的重构。

---

### 第四轮：通过标签筛选任务

#### **步骤 1：🔴 写测试**

```typescript
describe('任务标签功能 - 筛选任务', async () => {
  const { user, space, member, rootFolder } = await MockContext.initUserContext();
  
  test('应该能通过标签筛选任务', async () => {
    // 创建任务列表
    const taskDB = await rootFolder.createChildSimple(user, {
      name: '任务列表',
      resourceType: 'DATABASE',
    });
    const database = await taskDB.toResourceSO<DatabaseSO>();
    
    // 创建标签
    const urgentTag = await TaskTagSO.create(user, {
      name: '紧急',
      color: '#FF0000',
      spaceId: space.id,
    });
    
    // 创建多个任务
    const task1 = await database.createRecord(user, member, { name: '任务1' });
    const task2 = await database.createRecord(user, member, { name: '任务2' });
    const task3 = await database.createRecord(user, member, { name: '任务3' });
    
    // 只给任务1和任务2添加"紧急"标签
    await task1.addTag(urgentTag.id);
    await task2.addTag(urgentTag.id);
    
    // 通过标签筛选
    const urgentTasks = await database.getRecordsByTag(urgentTag.id);
    
    // 验证
    expect(urgentTasks.length).toBe(2);
    expect(urgentTasks.map(t => t.id)).toContain(task1.id);
    expect(urgentTasks.map(t => t.id)).toContain(task2.id);
    expect(urgentTasks.map(t => t.id)).not.toContain(task3.id);
  });
});
```

**运行测试：** ❌ 失败

---

#### **步骤 2：🟢 实现功能**

在 DatabaseSO 中添加方法：

```typescript
export class DatabaseSO {
  // ... 现有代码 ...
  
  async getRecordsByTag(tagId: string): Promise<RecordSO[]> {
    const recordTags = await db.prisma.recordTag.findMany({
      where: { 
        tagId: tagId,
      },
      include: {
        // 假设有关联到 record 表
      },
    });
    
    const recordIds = recordTags.map(rt => rt.recordId);
    
    // 获取完整的 record 数据
    const records = await this.getRecordsByIds(recordIds);
    
    return records;
  }
}
```

**运行测试：** ✅ 通过！

---

## 📊 TDD 流程总结图

```
┌─────────────────────────────────────────┐
│  1. 🔴 写一个失败的测试                    │
│     - 明确需求                           │
│     - 定义接口                           │
│     - 写测试用例                         │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│  2. 🟢 写最少的代码让测试通过               │
│     - 不考虑优化                         │
│     - 只求测试通过                       │
│     - 快速实现                           │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│  3. 🔵 重构代码                           │
│     - 优化结构                           │
│     - 提取重复代码                       │
│     - 保持测试通过                       │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│  4. 🔁 重复以上步骤                       │
│     - 添加新功能                         │
│     - 逐步完善                           │
└─────────────────────────────────────────┘
```

---

## 🎯 TDD 的优势

### ✅ **1. 需求驱动**
先写测试，强迫你思考清楚需求和接口设计

### ✅ **2. 快速反馈**
每次改动都能立即知道是否破坏了现有功能

### ✅ **3. 高测试覆盖率**
自然而然达到高覆盖率，因为代码都是为了通过测试而写的

### ✅ **4. 更好的设计**
为了让代码可测试，会写出更解耦、更模块化的代码

### ✅ **5. 重构信心**
有完整的测试保护，重构时不怕破坏功能

### ✅ **6. 文档作用**
测试本身就是最好的使用文档

---

## 💡 TDD 实践建议

### **对于初学者：**

1. **从小功能开始**
   - 不要一开始就做复杂功能
   - 选择独立的工具函数练习

2. **严格遵循红-绿-重构循环**
   - 不要跳过任何步骤
   - 养成习惯很重要

3. **小步前进**
   - 每次只添加一个小测试
   - 不要一次写太多测试

4. **及时重构**
   - 看到重复代码就重构
   - 不要等到代码很乱才重构

### **常见陷阱：**

❌ **陷阱 1：写太多测试再写代码**
```typescript
// ❌ 错误：一次写10个测试
test('test1', ...)
test('test2', ...)
test('test3', ...)
// ... 然后才开始写代码

// ✅ 正确：一次一个
test('test1', ...)  // 写这个测试
// 写代码让它通过
// 然后才写下一个测试
```

❌ **陷阱 2：为了通过测试写假实现**
```typescript
// ❌ 错误：硬编码返回值
static async create(user, data) {
  return { id: '123', name: '紧急' };  // 假数据
}

// ✅ 正确：真实实现
static async create(user, data) {
  return await db.prisma.taskTag.create({ data });
}
```

❌ **陷阱 3：跳过重构步骤**
```typescript
// ❌ 测试通过了就不管代码质量
// 结果代码越来越乱

// ✅ 测试通过后立即重构
// 保持代码整洁
```

---

## 🚀 实战练习建议

### **练习 1：简单工具函数（入门）**
开发一个字符串处理函数，使用 TDD：
```typescript
// 功能：将字符串转换为 slug 格式
// "Hello World!" → "hello-world"
```

### **练习 2：数据验证（进阶）**
开发一个邮箱验证器：
```typescript
// 功能：验证邮箱格式
// 支持多种规则
// 返回详细的错误信息
```

### **练习 3：业务逻辑（高级）**
开发一个简单的购物车功能：
```typescript
// 功能：
// - 添加商品
// - 删除商品
// - 计算总价
// - 应用折扣
```

---

## 📚 推荐学习路径

1. **第1周**：练习简单的纯函数（工具函数）
2. **第2周**：练习带数据库的 CRUD 操作
3. **第3周**：练习复杂的业务逻辑
4. **第4周**：在实际项目中应用 TDD

---

希望这个详细的 TDD 流程演示对你有帮助！关键是要**多练习**，一开始可能会觉得慢，但熟练后会发现 TDD 能大大提高开发质量和效率。
