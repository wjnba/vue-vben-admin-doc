# PageRender 组件说明文档

## 一、组件概述

**PageRender** 是一个基于配置驱动的动态页面渲染组件，专为医疗健康管理平台设计。该组件采用**元数据驱动架构**，通过 JSON 配置即可生成包含**分类树 + 表格 + 详情描述**的完整业务页面，支持高度自定义和扩展。

### 核心技术栈
- Vue 3.4+ (`<script setup lang="ts">`)
- TypeScript 5.4+
- Ant Design Vue 4.2+
- Pinia 2.1.7
- Vite 5.4+

### 主要特性
- ✅ **配置驱动**：通过 JSON 配置生成完整页面，无需编写模板代码
- ✅ **动态渲染**：支持分类树、表格、表单、弹窗、抽屉等多种组件
- ✅ **表达式系统**：支持 `${}` 表达式和路径格式的动态值解析
- ✅ **跨组件通信**：基于事件总线的组件间通信机制
- ✅ **插槽扩展**：支持作用域插槽的自定义内容扩展
- ✅ **状态共享**：统一的选中状态管理（selectionStore）

---

## 二、组件架构

### 2.1 组件结构

```
PageRender/
├── index.vue                  # 主入口组件
├── types.ts                   # TypeScript 类型定义
├── style.less                 # 全局样式
├── components/                # 子组件目录
│   ├── CategoryTree.vue       # 分类树组件
│   ├── YwtTable.vue           # 表格组件
│   ├── YwtModal.vue           # 弹窗组件
│   ├── YwtDrawer.vue          # 抽屉组件
│   ├── YwtForm.vue            # 表单组件
│   ├── YwtDescription.vue     # 详情描述组件
│   ├── Description/
│   │   └── registerComp.ts    # 详情自定义组件注册入口
│   ├── Form/
│   │   └── registerComp.ts    # 表单组件注册入口
│   └── Table/
│       ├── CellStatusSwitch.vue  # 单元格状态开关组件
│       ├── CellTag.vue           # 单元格标签组件
│       ├── CellCopyText.vue      # 单元格复制文本组件
│       └── registerComp.ts       # 单元格组件注册
├── hooks/                     # 组合式 API
│   ├── useConfigParser.ts     # 配置解析
│   ├── useConfigTransform.tsx # 配置转换
│   ├── useSharedSelection.ts  # 共享状态管理
│   ├── useCustomSlots.ts      # 自定义插槽处理
│   ├── useComponentAction.ts  # 组件动作处理
│   └── useLinkAction.ts       # 页面跳转动作处理
└── utils/                     # 工具函数
    ├── transformHelpers.ts    # 表达式解析和路径解析
    ├── safeEval.ts            # 安全表达式求值器（替代 new Function）
    ├── componentEventBus.ts   # 组件事件总线
    ├── formatHelpers.ts       # 内置格式化工具函数
    └── linkGuard.ts           # Link 动态路由恢复守卫
```

### 2.2 数据流图

```
路由 meta.extraData (JSON)
        ↓
useConfigParser (解析验证)
        ↓
PageRenderConfig (配置对象)
        ↓
┌───────────────────────────────────────┐
│  selectionStore (Provide/Inject)      │
│  - 跨组件状态共享                      │
│  - updateSelection / getSelectionByKey │
└───────────────────────────────────────┘
        ↓
┌──────────────┬────────────────────────┐
│ CategoryTree │     YwtTable           │
│  - 分类管理  │  - 数据表格            │
│  - 树形结构  │  - Tabs 切换           │
│  - 节点操作  │  - 行操作              │
└──────────────┴────────────────────────┘
        ↓
┌──────────────┬────────────────────────┐
│  YwtModal    │      YwtDrawer         │
│  - 弹窗表单  │  - 抽屉表单            │
│  - 弹窗表格  │  - 抽屉表格            │
└──────────────┴────────────────────────┘
```

---

## 三、主入口组件 (index.vue)

### 3.1 功能说明

主入口组件负责：
1. 从路由 `meta.extraData` 中读取 JSON 配置
2. 解析并验证配置结构
3. 提供 `selectionStore` 给子组件共享
4. 渲染分类树和表格布局

### 3.2 Props & 配置

组件无需传入 props，配置来自路由：

```typescript
// 路由配置示例
{
    path: '/example',
        component: () => import('@/views/PageRender.vue'),
    meta: {
    renderMode: 'auto',
        extraData: JSON.stringify({
        categoryTreeConfig: { ... },
        tableConfig: { ... }
    })
}
}
```

### 3.3 插槽系统

支持**透传插槽**，允许父组件向子组件传递自定义内容：

```vue
<PageRender>
  <!-- 传递给分类树的插槽 -->
  <template #tree-header-actions="{ selection }">
    <Button>自定义按钮</Button>
  </template>

  <!-- 传递给表格 name 列的插槽（详见 6.7） -->
  <template #tab1-cell-name="{ record }">
    <a>{{ record.name }}</a>
  </template>
</PageRender>
```

### 3.4 错误处理

配置解析失败时显示错误提示：
- 解析方式（renderMode）
- 错误信息
- 检查提示

---

## 四、类型定义 (types.ts)

### 4.1 核心类型

#### DataSourceConfig - 数据源配置

```typescript
interface DataSourceConfig {
    url?: string;                              // 请求地址
    method?: 'get' | 'post' | 'put' | 'delete'; // 请求方法
    params?: Recordable;                       // 固定参数
    extraParams?: Array<{                      // 动态参数映射
        paramsKey: string;   // 请求参数名
        valueKey: string;    // 值来源路径（支持 ${} 表达式）
    }>;
    getChild?: DataSourceConfig;               // 获取子级数据（树形表格）
    requestOptions?: {
        dictTransform?: Recordable;              // 字典转换
    };
    labelField?: string;                       // 标签字段名
    valueField?: string;                       // 值字段名
    childrenField?: string;                    // 子级字段名
}
```

#### HandleControlConfig - 交互控制配置

```typescript
interface HandleControlConfig {
    component: string;                          // 交互组件类型：Popconfirm | YwtModal | YwtDrawer | Link
    componentProps?: Recordable;                // 组件属性
    slotComponent?: string;                     // 内部插槽组件：YwtForm | YwtTable
    slotComponentProps?: Recordable;            // 插槽组件属性
    dataSource?: {
        initSetting?: DataSourceConfig;           // 初始化数据请求
        updateSetting?: DataSourceConfig;         // 更新数据请求
    };
}
```

**component 可选值**：
| 值 | 说明 | 适用场景 |
|----|------|---------|
| `Popconfirm` | 确认对话框 | 删除/停用等需要二次确认的操作 |
| `YwtModal` | 弹窗 | 表单编辑/新增、详情展示 |
| `YwtDrawer` | 抽屉 | 侧边表单编辑、详情展示 |
| `Link` | 页面跳转 | 打开外部链接或内部路由页面 |

#### ActionConfig - 操作按钮配置

```typescript
interface ActionConfig {
    label: string;                    // 按钮文本（支持 ${} 表达式）
    key: string;                      // 唯一标识
    type?: 'primary' | 'warning' | 'danger' | 'success' | 'default';
    icon?: string;                    // 图标（转换为 preIcon）
    className?: string;               // 自定义类名
    handleControl?: HandleControlConfig; // 交互控制
    disabled?: boolean | string;      // 禁用状态（支持表达式）
}
```

#### CellActionConfig - 行操作按钮配置

```typescript
interface CellActionConfig extends Omit<ActionItem, 'ifShow' | 'disabled'> {
    key: string;
    handleControl?: HandleControlConfig;
    disabled?: boolean | string | ((action: CellActionConfig) => boolean);
    ifShow?: boolean | string | ((action: CellActionConfig) => boolean);
}
```

#### TableTabConfig - 表格标签页配置

```typescript
interface TableTabConfig {
    key: string;                              // 标签页唯一标识
    label: string;                            // 标签文本
    icon?: string;                            // 图标
    disabled?: boolean;                       // 是否禁用
    showCount?: number;                       // 是否显示数量
    count?: number;                           // 数量值
    dataSource?: {
        initSetting?: DataSourceConfig;         // 初始化数据请求
        updateSetting?: DataSourceConfig;       // 更新数据请求
    };
    headerActions?: ActionConfig[];           // 头部操作按钮
    cellActions?: CellActionConfig[];         // 行操作按钮
    componentProps?: {
        columns?: BasicColumn[];                // 列配置
        formConfig?: {
            schemas?: FormSchema[];               // 表单配置
        };
        actionColumn?: Recordable;              // 操作列配置
    };
}
```

#### CategoryTreeConfig - 分类树配置

```typescript
interface CategoryTreeConfig {
    key: string;                              // 树组件标识
    title?: string;                           // 树标题
    headerActions?: ActionConfig[];           // 头部操作按钮
    cellActions?: ActionConfig[];             // 节点操作按钮
    dataSource?: {
        initSetting?: DataSourceConfig;         // 初始化数据请求
        updateSetting?: DataSourceConfig;       // 更新数据请求
    };
}
```

#### TransformContext - 转换上下文

```typescript
interface TransformContext {
    route?: Recordable;     // 路由信息
    rowData?: Recordable;   // 行数据
    formData?: Recordable;  // 表单数据
    selection?: Recordable; // 选中数据
    userInfo?: Recordable;  // 当前用户信息（来自 userStore.getUserInfo）
    [key: string]: any;
}
```

#### MergeCellConfig - 合并单元格配置

```typescript
type MergeCellConfig = boolean | { groupBy?: string };
```

- `true`：相邻行相同值自动合并
- `{ groupBy: 'fieldName' }`：按指定字段分组，组内相邻相同值合并

详见 [6.4 mergeCell - 合并单元格](#mergecell---合并单元格)。

---

## 五、分类树组件 (CategoryTree.vue)

### 5.1 功能特性

- **树形结构展示**：支持无限层级的分类展示
- **审核状态显示**：待审核、审核不通过状态图标提示
- **节点操作**：支持编辑、删除等节点级别操作
- **头部操作**：支持添加分类等全局操作
- **懒加载**：支持子节点懒加载
- **自动选择**：加载后自动选择第一个节点

### 5.2 Props 配置

```typescript
interface Props {
  config: CategoryTreeConfig;
}
```

### 5.3 数据转换

树数据会自动转换为 BasicTree 所需格式：

```typescript
// 输入数据
{
  id: '1',
  name: '分类名称',
  auditStatus: 0,  // 0-待审核，5-审核不通过
  hasChildren: true,
  children: [...]
}

// 转换后
{
  key: '1',           // 来自 valueField 或 id
  title: '分类名称',   // 来自 labelField 或 name
  auditStatus: 0,
  children: [...]
}
```

### 5.4 节点操作按钮

支持四种交互方式：

1. **Popconfirm**：确认对话框
   ```json
   {
     "label": "删除",
     "icon": "ant-design:delete-outlined",
     "className": "delete-btn",
     "handleControl": {
       "component": "Popconfirm",
       "componentProps": {
         "title": "确认删除该分类？"
       },
       "dataSource": {
         "updateSetting": {
           "url": "/api/category/delete",
           "extraParams": [
             { "paramsKey": "id", "valueKey": "${rowData.id}" }
           ]
         }
       }
     }
   }
   ```

2. **YwtDrawer/YwtModal**：打开抽屉或弹窗
   ```json
   {
     "label": "编辑",
     "icon": "ant-design:edit-outlined",
     "className": "edit-btn",
     "handleControl": {
       "component": "YwtDrawer",
       "slotComponent": "YwtForm",
       "slotComponentProps": { ... }
     }
   }
   ```

3. **Link**：页面跳转（外部链接或内部路由）
   ```json
   {
     "label": "查看详情",
     "key": "viewDetail",
     "icon": "ant-design:link-outlined",
     "handleControl": {
       "component": "Link",
       "componentProps": {
         "path": "${'/detail/' + rowData.id}",
         "goType": "push"
       }
     }
   }
   ```

**节点按钮的条件显示**：

通过 `ifShow` 字段控制按钮的显示/隐藏，支持 `${}` 表达式：

```json
{
  "label": "审核",
  "key": "audit",
  "icon": "ant-design:check-circle-outlined",
  "ifShow": "${rowData.auditStatus === 0}",   // 仅待审核状态显示
  "handleControl": {
    "component": "YwtDrawer",
    "slotComponent": "YwtForm",
    ...
  }
}
```

### 5.5 事件

- `@select(node)`：节点选中事件，返回选中的节点数据

### 5.6 事件总线集成

分类树组件使用 `useComponentAction` 自动监听 `comp:action` 事件：

```typescript
// 当收到 reload 动作且 targetKeys 包含树组件的 key 时，自动刷新树数据
useComponentAction(treeKey, { reload: () => loadTree() });
```

其他组件（如表单提交后）可以通过事件总线触发树刷新：

```typescript
componentEventBus.emit('comp:action', {
  action: 'reload',
  targetKeys: ['tree'],  // tree 为分类树配置中的 key
});
```

---

## 六、表格组件 (YwtTable.vue)

### 6.1 功能特性

- **多标签页切换**：支持多个表格标签页
- **动态列配置**：支持表达式控制列显示和格式化
- **行操作**：支持多种行操作按钮
- **树形表格**：支持展开加载子级数据
- **自定义单元格**：支持作用域插槽自定义渲染
- **弹窗/抽屉集成**：自动集成 Modal 和 Drawer

### 6.2 Props 配置

```typescript
interface Props {
  tableConfig: TableConfig;
}

interface TableConfig {
  defaultActive?: string;  // 默认激活的标签页 key
  tabs: TableTabConfig[];  // 标签页配置数组
}
```

### 6.3 标签页配置

```json
{
  "tableConfig": {
    "defaultActive": "tab1",
    "tabs": [
      {
        "key": "tab1",
        "label": "待审核",
        "icon": "ant-design:clock-circle-outlined",
        "showCount": true,
        "count": 10,
        "dataSource": {
          "initSetting": {
            "url": "/api/list",
            "method": "post",
            "params": { "status": "1" },
            "extraParams": [
              { "paramsKey": "categoryId", "valueKey": "${selection.tree.id}" }
            ]
          }
        },
        "headerActions": [
          {
            "label": "批量删除",
            "key": "batchDelete",
            "type": "danger",
            "icon": "ant-design:delete-outlined",
            "disabled": "${!selection.tree || !selection.tree.id}"
          }
        ],
        "cellActions": [
          {
            "label": "编辑",
            "key": "edit",
            "type": "primary",
            "icon": "ant-design:edit-outlined",
            "handleControl": {
              "component": "YwtModal",
              "slotComponent": "YwtForm",
              "slotComponentProps": { ... }
            }
          }
        ],
        "componentProps": {
          "columns": [
            {
              "title": "姓名",
              "dataIndex": "name",
              "ifShow": "${rowData.status === 1}",
              "format": "${rowData.name + ' (' + rowData.age + '岁)'}"
            }
          ],
          "actionColumn": {
            "width": 200,
            "fixed": "right",
            "title": "操作"
          }
        }
      }
    ]
  }
}
```

### 6.4 列配置增强

#### ifShow - 条件显示

```typescript
{
  "dataIndex": "name",
  "ifShow": "${rowData.status === 1}"  // 字符串表达式
}
```

转换后：
```typescript
{
  dataIndex: 'name',
  ifShow: (column: Recordable) => {
    return parseExpression('${rowData.status === 1}', { rowData: column });
  }
}
```

#### format - 格式化显示

支持 `${}` 表达式，自动注入内置格式化工具函数（详见 12.2.7 节）：

```typescript
{
  "dataIndex": "name",
  "format": "${rowData.name + ' (' + rowData.age + '岁)'}"
}
```

**使用内置工具函数**：

```typescript
{
  "dataIndex": "createTime",
  "title": "创建时间",
  "format": "${dateFormat(rowData.createTime, 'YYYY年MM月DD日')}"
}
{
  "dataIndex": "birthday",
  "title": "年龄",
  "format": "${calcAge(rowData.birthday) + '岁'}"
}
{
  "dataIndex": "amount",
  "title": "金额",
  "format": "${'¥' + numberFormat(rowData.amount, 2)}"
}
{
  "dataIndex": "remark",
  "title": "备注",
  "format": "${truncate(rowData.remark, 10)}"
}
```

#### component - 自定义组件

```typescript
{
  "dataIndex": "status",
  "component": "CellStatusSwitch",
  "bindField": "status",
  "triggerActions": [
    { "type": "reload", "targetKeys": ["tab1"] }
  ]
}
```

#### mergeCell - 合并单元格

支持将相邻行中相同值的列合并为一个单元格，适用于分类汇总、分组展示等场景。

**配置方式**：

```typescript
{
  // 简写：相邻行相同值自动合并
  "title": "分类",
  "dataIndex": "category",
  "mergeCell": true
}
```

```typescript
{
  // 完整写法：按指定分组字段合并，只在组内进行合并
  "title": "日期",
  "dataIndex": "date",
  "mergeCell": {
    "groupBy": "category"
  }
}
```

**两种模式对比**：

以如下数据为例：

| 日期 | 分类 | 金额 |
|------|------|------|
| 2026-01-01 | 餐饮 | 30 |
| 2026-01-01 | 餐饮 | 25 |
| 2026-01-01 | 交通 | 50 |
| 2026-01-02 | 餐饮 | 35 |
| 2026-01-02 | 交通 | 60 |
| 2026-01-02 | 交通 | 20 |

- **`mergeCell: true`**：只看当前列值是否与上一行相同，相同则合并。日期列 "2026-01-01" 三行合并，"2026-01-02" 三行合并。
- **`mergeCell: { groupBy: "category" }`**：先按 `category` 分组，分组内相同值再合并。日期列在餐饮组内相同（不合并），交通组内相同（不合并），跨组不合并，合并结果更精准。

> **注意**：
> - 合并仅在同一页数据内生效，不支持跨页合并
> - 树形表格（`isTreeTable`）不支持合并单元格
> - `mergeCell` 与 `component`（自定义组件）互斥，组件列优先
> - 排序、筛选后会自动重新计算合并范围

### 6.5 行操作按钮

支持三种状态：
- **actions**：显示在操作列的按钮（最多 2 个）
- **dropDownActions**：超出 2 个的按钮放入下拉菜单

按钮交互类型：
1. **Popconfirm**：确认对话框
2. **YwtModal**：打开弹窗
3. **YwtDrawer**：打开抽屉
4. **Link**：页面跳转（外部链接或内部路由）

### 6.6 树形表格展开

支持懒加载子级数据：

```json
{
  "dataSource": {
    "initSetting": {
      "url": "/api/parent/list",
      "getChild": {
        "url": "/api/child/list",
        "params": { "type": "child" },
        "extraParams": [
          { "paramsKey": "parentId", "valueKey": "${rowData.id}" }
        ]
      }
    }
  }
}
```

### 6.7 插槽系统

表格组件提供 3 类插槽，均以 `{tabKey}` 作为命名空间前缀，支持为每个标签页独立定制。

#### 列自定义渲染（`{tabKey}-cell-{columnKey}`）

当内置的 `format` 表达式或 `component` 组件无法满足渲染需求时，可通过插槽完全自定义某列的渲染。`columnKey` 对应 JSON 列配置中的 `dataIndex` 或 `key`。

> **注意**：JSON 列配置中必须声明 `slots.customRender` 为列名，插槽才会生效。这是底层 VXE Table 的要求。

**JSON 列配置**：

```json
{
  "columns": [
    {
      "dataIndex": "name",
      "title": "姓名",
      "slots": { "customRender": "name" }
    },
    {
      "dataIndex": "status",
      "title": "状态",
      "slots": { "customRender": "status" }
    }
  ]
}
```

**对应的插槽**：

```vue
<PageRender>
  <!-- name 列 -> 插槽名 tab1-cell-name -->
  <template #tab1-cell-name="{ record, column, index, text }">
    <a>{{ record.name }}</a>
  </template>

  <!-- status 列 -> 插槽名 tab1-cell-status -->
  <template #tab1-cell-status="{ record, column }">
    <ATag :color="record.status === 1 ? 'green' : 'red'">
      {{ record.status === 1 ? '启用' : '禁用' }}
    </ATag>
  </template>
</PageRender>
```

> 插槽名格式固定为 `{tabKey}-cell-{columnKey}`，其中 `columnKey` 必须与 JSON 中 `slots.customRender` 的值一致。

**插槽作用域数据**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `text` | `any` | 当前单元格原始值（`record[column.dataIndex]`） |
| `record` | `Recordable` | 当前行数据 |
| `column` | `BasicColumn` | 当前列配置 |
| `index` | `number` | 当前行索引 |

#### 行操作栏插槽（`{tabKey}-cell-action`）

自定义整个操作列的渲染，覆盖默认的 `TableAction` 组件。

```vue
<PageRender>
  <template #tab1-cell-action="{ record, column, props }">
    <!-- props 包含转换后的 actions 和 dropDownActions -->
    <TableAction v-bind="props" />
    <!-- 也可以完全自定义 -->
    <Button @click="handleCustom(record)">自定义</Button>
  </template>
</PageRender>
```

**插槽作用域数据**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `record` | `Recordable` | 当前行数据 |
| `column` | `BasicColumn` | 操作列配置 |
| `props` | `{ actions, dropDownActions? }` | 经过 `transformRowActions` 转换后的按钮配置 |

#### 头部操作插槽（`{tabKey}-header-actions`）

在标签页头部追加自定义操作按钮，位于配置的 `headerActions` 之后。

```vue
<PageRender>
  <template #tab1-header-actions="{ selection }">
    <Button type="primary" @click="handleBatch">批量导出</Button>
  </template>
</PageRender>
```

**插槽作用域数据**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `selection` | `Recordable` | 当前选中状态（来自 selectionStore） |

#### 完整示例

```vue
<PageRender>
  <!-- 列自定义渲染 -->
  <template #tab1-cell-name="{ record }">
    <a>{{ record.name }}</a>
  </template>
  <template #tab1-cell-status="{ record }">
    <ATag :color="record.status === 1 ? 'green' : 'red'">
      {{ record.status === 1 ? '启用' : '禁用' }}
    </ATag>
  </template>

  <!-- 行操作栏 -->
  <template #tab1-cell-action="{ record, props }">
    <TableAction v-bind="props" />
  </template>

  <!-- 头部操作 -->
  <template #tab1-header-actions>
    <Button>批量导出</Button>
  </template>
</PageRender>
```

> 提示：插槽名称中的 `tab1` 需替换为实际标签页配置中的 `key`。

---

## 七、表单组件 (YwtForm.vue)

### 7.1 功能特性

- **配置驱动**：通过配置生成表单
- **动态数据源**：支持表单字段的数据源请求
- **字段联动**：支持字段间的依赖关系
- **表达式支持**：支持条件显示、动态禁用等表达式
- **自动验证**：提交时自动验证表单

### 7.2 Props 配置

```typescript
interface Props {
    config: {
        key: string;                          // 表单标识
        triggerActions?: Array<{              // 触发刷新动作
            type: string;
            targetKeys: string[];
        }>;
        dataSource?: {
            initSetting?: DataSourceConfig;     // 初始化数据请求
            updateSetting?: DataSourceConfig;   // 更新数据请求
        };
        // BasicForm 的其他配置
        schemas?: FormSchema[];
        labelWidth?: number;
        ...
    };
    rowData?: Recordable;  // 行数据（用于编辑场景）
}
```

### 7.3 表单 Schema 转换

#### show - 条件显示

```typescript
{
    "field": "name",
        "component": "Input",
        "show": "${formData.type === 1}"
}
```

#### ifShow - 条件渲染

```typescript
{
    "field": "age",
        "component": "InputNumber",
        "ifShow": "${formData.hasAge === true}"
}
```

#### dynamicDisabled - 动态禁用

```typescript
{
    "field": "email",
        "component": "Input",
        "dynamicDisabled": "${formData.locked === true}"
}
```

#### dataSource - 动态数据源

支持以下组件的 API 数据源：
- ApiCascader
- ApiRadioGroup
- ApiSelect
- ApiTransfer
- ApiTree
- ApiTreeSelect

```typescript
{
    "field": "categoryId",
        "component": "ApiSelect",
        "componentProps": {
        "dataSource": {
            "initSetting": {
                "url": "/api/category/list",
                    "extraParams": [
                    { "paramsKey": "parentId", "valueKey": "${formData.parentId}" }
                ]
            }
        }
    }
}
```

#### clearBy - 字段清空依赖

```typescript
{
    "field": "subCategoryId",
        "component": "ApiSelect",
        "clearBy": ["categoryId"]  // 当 categoryId 变化时清空 subCategoryId
}
```

### 7.4 数据初始化

表单加载时自动请求数据：

1. **initSetting**：请求额外数据
2. **rowData**：传入的行数据
3. 合并后设置为表单初始值

### 7.5 表单提交

```typescript
async function handleOk() {
    // 1. 验证表单
    const values = await validate();

    // 2. 合并参数
    const params = mergeRequestParams(
        { ...values, ...(updateSetting?.params || {}) },
        updateSetting?.extraParams || [],
        {
            rowData: props.rowData,
            formData: values,
        }
    );

    // 3. 发送请求
    const apiFn = createRequest(updateSetting || {});
    return apiFn(params).then((res) => {
        message.success(res?.message || '操作成功');

        // 4. 触发刷新动作
        triggerActions.value.forEach((act) => {
            componentEventBus.emit('comp:action', {
                action: act.type,
                targetKeys: act.targetKeys,
            });
        });
    });
}
```

### 7.6 插槽系统

支持自定义表单项渲染：

```vue
<YwtForm :config="formConfig">
  <!-- 自定义 username 字段的渲染 -->
  <template #myForm-formItem-username="{ model, schema }">
    <Input v-model:value="model[schema.field]" placeholder="请输入用户名" />
  </template>
</YwtForm>
```

---

## 八、弹窗组件 (YwtModal.vue)

### 8.1 功能特性

- **动态组件**：支持加载 YwtForm、YwtTable 等组件
- **插槽透传**：支持向内部组件透传插槽
- **自动关闭**：提交成功后自动关闭
- **错误处理**：使用 `awaitTo` 处理异步错误
- **状态重置**：关闭时自动重置内部状态（`closeFunc` 返回 Promise）

### 8.2 Props 配置

通过 `useModalInner` 接收参数：

```typescript
{
    actionConfig: HandleControlConfig;  // 操作配置
    row?: Recordable;                    // 行数据
}
```

### 8.3 支持的组件

```typescript
const componentsMap = {
    YwtForm,   // 表单组件
    YwtTable,  // 表格组件
};
```

### 8.4 配置示例

```json
{
  "handleControl": {
    "component": "YwtModal",
    "slotComponent": "YwtForm",
    "componentProps": {
      "width": 800,
      "helpMessage": ["请填写相关信息"]
    },
    "slotComponentProps": {
      "key": "editForm",
      "dataSource": {
        "updateSetting": {
          "url": "/api/update",
          "method": "post"
        }
      },
      "schemas": [...]
    }
  }
}
```

---

## 九、抽屉组件 (YwtDrawer.vue)

### 9.1 功能特性

与 YwtModal 类似，区别在于：
- 使用抽屉形式展示
- 默认宽度 800px
- 支持 `placement` 配置（默认 `right`）
- 其他功能与 Modal 一致（awaitTo 错误处理、closeFunc 状态重置）

### 9.2 配置示例

```json
{
  "handleControl": {
    "component": "YwtDrawer",
    "slotComponent": "YwtForm",
    "componentProps": {
      "width": 600,
      "placement": "right"
    },
    "slotComponentProps": { ... }
  }
}
```

---

## 十、单元格组件 (CellStatusSwitch.vue)

### 10.1 功能特性

- **双向绑定**：使用 `defineModel` 实现 v-model 绑定
- **状态切换**：点击切换开关状态，通过 API 持久化
- **双 URL 切换**：支持启用/停用分别配置不同请求地址（enableUrl / disabledUrl）
- **动态禁用**：支持表达式控制禁用状态
- **事件触发**：切换后通过事件总线触发刷新动作

### 10.2 Props 配置

```typescript
interface Props {
    record?: Recordable;                    // 行数据
    triggerActions?: Array<{                // 触发刷新动作
        type: string;
        targetKeys: string[];
    }>;
    dataSource?: {                          // 数据源配置
        updateSetting?: {
            enableUrl?: string;                 // 启用状态的请求地址
            disabledUrl?: string;               // 停用状态的请求地址
            method?: string;
            params?: Recordable;
            extraParams?: Array<{
                paramsKey: string;
                valueKey: string;
            }>;
        };
    };
    disabled?: string;                      // 禁用表达式（如 "${rowData.locked}"）
}
```

### 10.3 使用示例

```typescript
{
    "dataIndex": "status",
        "component": "CellStatusSwitch",
        "bindField": "status",
        "triggerActions": [
        { "type": "reload", "targetKeys": ["tab1"] }
    ],
        "componentProps": {
        "dataSource": {
            "updateSetting": {
                "enableUrl": "/api/enable",
                    "disabledUrl": "/api/disable",
                    "extraParams": [
                    { "paramsKey": "id", "valueKey": "${rowData.id}" }
                ]
            }
        },
        "disabled": "${rowData.locked === true}"
    }
}
```

---

## 十一、单元格标签组件 (CellTag.vue)

### 11.1 功能特性

- **状态标签**：根据值映射显示不同颜色和文本的标签
- **字典映射**：支持通过 `mapOption` 配置值到颜色、文本的映射关系
- **表达式格式化**：支持 `${}` 表达式格式化标签文本
- **自动配色**：未指定颜色时按预设颜色列表自动分配

### 11.2 Props 配置

```typescript
interface MapOption {
    label: string;                // 显示文本
    value: string | number | boolean; // 匹配值
    color?: string;              // 标签颜色（可选，自动分配）
}

interface Props {
    record?: Recordable;          // 行数据
    mapOption?: MapOption[];      // 值映射配置
    format?: string;              // 格式化表达式（当无映射匹配时使用）
}
```

### 11.3 使用示例

```typescript
{
    "dataIndex": "status",
        "component": "CellTag",
        "bindField": "status",
        "componentProps": {
        "mapOption": [
            { "label": "已启用", "value": 1, "color": "green" },
            { "label": "已停用", "value": 0, "color": "red" }
        ]
    }
}
```

### 11.4 颜色分配规则

- 优先使用 `mapOption` 中指定的 `color`
- 未指定颜色时，按值在数组中的索引依次使用预设颜色列表：
  `['blue', 'green', 'red', 'orange', 'purple', 'cyan', 'lime', 'pink']`

---

## 十二、详情描述组件 (YwtDescription.vue)

### 12.1 功能特性

- **配置驱动**：通过 JSON 配置生成详情描述页面
- **动态数据源**：支持从 API 加载详情数据，自动合并 `rowData`
- **表达式支持**：支持 `show` 条件显示表达式
- **自定义组件**：支持在详情项中使用 `CellTag`、`CellCopyText` 等自定义组件渲染
- **命名空间插槽**：支持按字段自定义渲染，自动合并 `rowData` 和 API 数据
- **跨组件通信**：数据加载完成后自动触发 `triggerActions`
- **前后插槽**：支持 `header`/`footer` 插槽进行内容扩展

### 12.2 Props 配置

```typescript
interface Props {
    config: {
        key: string;                          // 组件唯一标识
        schema: Array<{                       // 详情项配置
            field: string;                      // 数据字段路径
            label: string;                      // 显示标签
            span?: number;                      // 合并列数
            labelMinWidth?: number;             // 标签最小宽度
            contentMinWidth?: number;           // 内容最小宽度
            show?: string;                      // 条件显示表达式（如 "${rowData.status !== '0'}"）
            component?: string;                 // 自定义组件名（如 "CellTag"、"CellCopyText"）
            componentProps?: Recordable;        // 自定义组件属性
            format?: string;                    // 格式化表达式（自定义组件时使用）
        }>;
        dataSource?: {
            initSetting?: DataSourceConfig;     // 初始化数据请求
        };
        triggerActions?: Array<{              // 触发刷新动作
            type: string;
            targetKeys: string[];
        }>;
        title?: string;                       // 标题
        useCollapse?: boolean;                // 是否使用折叠容器
        column?: number | Recordable;         // 列数
        bordered?: boolean;                   // 是否显示边框
        size?: 'small' | 'default' | 'middle';// 尺寸
        collapseOptions?: Recordable;         // 折叠容器配置
    };
    rowData?: Recordable;  // 行数据
}
```

### 12.3 Schema 配置项

每个详情项通过 `schema` 数组配置：

```typescript
{
    "field": "name",           // 数据字段，支持 lodash get 路径
        "label": "姓名",           // 显示标签
        "span": 1,                 // 合并列数（可选）
        "labelMinWidth": 100,      // 标签最小宽度（可选）
        "contentMinWidth": 200,    // 内容最小宽度（可选）
        "show": "${rowData.status !== '0'}", // 条件显示（可选，${} 表达式）
        "component": "CellTag",    // 自定义组件（可选）
        "componentProps": {...},   // 组件属性（可选）
    "format": "..."            // 格式化表达式（可选，组件内使用）
}
```

#### show - 条件显示

支持 `${}` 表达式，动态控制详情项是否显示：

```typescript
{
    "field": "approver",
        "label": "审批人",
        "show": "${rowData.status === '1'}"
}
```

#### component - 自定义组件

复用 `Description/registerComp.ts` 中注册的组件，目前支持的组件：

| 组件名 | 说明 | 来源 |
|--------|------|------|
| `CellTag` | 状态标签（支持颜色映射） | `Table/CellTag.vue` |
| `CellCopyText` | 复制文本 | `Table/CellCopyText.vue` |
| `CellStatusSwitch` | 状态开关 | `Table/CellStatusSwitch.vue` |

```typescript
{
    "field": "status",
        "label": "状态",
        "component": "CellTag",
        "componentProps": {
        "mapOption": [
            { "label": "启用", "value": "1", "color": "green" },
            { "label": "禁用", "value": "0", "color": "red" }
        ]
    }
}
```

### 12.4 数据初始化

组件加载数据流程：

1. 接收 `config` 和 `rowData` props
2. 解构配置中的 `key`、`schema`、`dataSource`、`triggerActions` 等
3. 通过 `transformDescSchemas` 转换 schema（解析 `show` 表达式，处理自定义组件）
4. 调用 `setDescProps` 注入 schema 和 DescriptionProps
5. 如果有 `dataSource.initSetting`，自动发起 API 请求获取额外数据
6. 合并 `rowData` 和 API 数据，通过 `setDescProps` 注入 `data`
7. 数据加载完成后自动触发 `triggerActions` 事件总线

### 12.5 数据流

```
props.config (JSON 配置)
        ↓
  解构 key / schema / dataSource / triggerActions
        ↓
  transformDescSchemas (转换 schema)
    ├── 解析 show 表达式为函数
    ├── 处理 component 自定义组件 → h() 渲染
    └── 绑定 modelValue 双向绑定
        ↓
  setDescProps({ schema, ...descProps })
        ↓
  initData()
    ├── 有 initSetting.url → API 请求
    ├── 合并 props.rowData + API data
    └── setDescProps({ data })
        ↓
  emitTriggerActions() (触发跨组件通信)
```

### 12.6 插槽系统

提供三类插槽，均以 `{key}-desc-` 为命名空间前缀。

#### 字段自定义渲染（`{key}-desc-{field}`）

当自定义组件无法满足需求时，可通过命名插槽完全自定义某个详情项的渲染：

```vue
<PageRender>
  <template #userDetail-desc-status="{ value, data }">
    <a-tag :color="value === '1' ? 'green' : 'red'">
      {{ value === '1' ? '启用' : '禁用' }}
    </a-tag>
  </template>
</PageRender>
```

> 插槽名格式：`{config.key}-desc-{schema.field}`
> 作用域数据：`{ value: any, data: Recordable }`

#### 头部插槽（`{key}-desc-header`）

在详情组件上方插入自定义内容：

```vue
<PageRender>
  <template #userDetail-desc-header="{ data }">
    <div class="mb-4">用户：{{ data.name }}</div>
  </template>
</PageRender>
```

#### 底部插槽（`{key}-desc-footer`）

在详情组件下方插入自定义内容：

```vue
<PageRender>
  <template #userDetail-desc-footer>
    <Button type="primary">编辑</Button>
  </template>
</PageRender>
```

### 12.7 配置示例

```json
{
  "key": "userDetail",
  "title": "用户信息",
  "column": 2,
  "bordered": false,
  "schema": [
    { "field": "name", "label": "姓名" },
    { "field": "phone", "label": "手机号" },
    { "field": "email", "label": "邮箱" },
    { "field": "deptName", "label": "所属部门" },
    {
      "field": "status",
      "label": "状态",
      "component": "CellTag",
      "componentProps": {
        "mapOption": [
          { "label": "启用", "value": "1", "color": "green" },
          { "label": "禁用", "value": "0", "color": "red" }
        ]
      }
    },
    { "field": "createTime", "label": "创建时间", "span": 2 }
  ],
  "dataSource": {
    "initSetting": {
      "url": "/api/user/detail",
      "params": { "id": "${rowData.id}" }
    }
  },
  "triggerActions": [
    { "type": "reload", "targetKeys": ["userTable"] }
  ]
}
```

---

## 十三、Hooks 详解

### 13.1 useConfigParser

**功能**：解析和验证路由配置

**返回值**：
```typescript
{
    autoJson: Ref<PageRenderConfig>;  // 解析后的配置
    isValid: Ref<boolean>;             // 是否有效
    errorMsg: Ref<string | null>;      // 错误信息
    renderMode: string;                // 渲染模式
}
```

**工作流程**：
1. 从 `route.meta.extraData` 读取 JSON 字符串
2. 安全解析 JSON（parseJsonSafe）
3. 验证配置结构（isValidConfig）
4. 深度合并配置（deepMerge）
5. 监听路由变化自动重新解析

### 13.2 useConfigTransform

**功能**：配置转换和参数处理

**核心方法**：

#### buildContext

构建完整的上下文对象，默认包含 route、selection、userInfo，并自动注入内置格式化工具函数（详见 12.2.7）：

```typescript
function buildContext(overrideContext: Partial<TransformContext> = {}): TransformContext {
    return {
        route,                                     // 当前路由信息
        selection: selectionStore?.selection || {}, // 跨组件选中数据
        userInfo: userStore.getUserInfo || {},      // 当前登录用户信息
        ...formatHelpers,                          // 注入内置格式化工具函数
        ...overrideContext,                        // 调用方传入的上下文（优先级最高）
    };
}
```

内置工具函数通过 `formatHelpers` 注入，可在 format、ifShow、disabled、show、dynamicDisabled 等所有支持表达式的字段中使用（见下节）。

#### mergeRequestParams

合并请求参数：

```typescript
function mergeRequestParams(
    params: Recordable,
    extraParams: Array<{ paramsKey: string; valueKey: string; ifShow?: string }>,
    context: TransformContext = {}
): Recordable

// 使用示例
const params = mergeRequestParams(
    { status: '1' },
    [{ paramsKey: 'id', valueKey: 'selection.tree.id' }],
    { selection: { tree: { id: '123' } } }
);
// 结果：{ status: '1', id: '123' }
```

**增强特性**：
- `extraParams` 支持 `ifShow` 字段，值为表达式时根据上下文判断是否添加该参数
- 自动通过 `buildContext` 合并 route、selection、userInfo 到上下文

#### resolveExpression

解析字符串中的 `${}` 表达式，替换为上下文中的实际值：

```typescript
function resolveExpression(
    expression: string,
    context: TransformContext
): any

// 处理逻辑
// 1. 纯 ${} 包裹：提取变量路径，从 context 中取值
// 2. 字符串包含 ${}：替换所有变量为实际值
// 3. 路径格式（不带 ${}）：直接取值
// 4. 其他情况：原样返回

// 使用示例
resolveExpression('${rowData.name}', { rowData: { name: '张三' } });
// 返回：'张三'

resolveExpression('selection.tree.id', { selection: { tree: { id: '123' } } });
// 返回：'123'

resolveExpression('用户: ${rowData.name} (${rowData.age}岁)', { rowData: { name: '张三', age: 25 } });
// 返回：'用户: 张三 (25岁)'

resolveExpression('普通文本', {});
// 返回：'普通文本'
```

#### createRequest

创建 API 请求函数：

```typescript
function createRequest(config: DataSourceConfig) {
    return (params?: Recordable) => {
        const url = config.url || '';
        const method = config.method || 'post';
        const requestOptions = config.requestOptions || {};
        return defHttp.request({ url, method, params }, requestOptions);
    };
}

// 使用示例
const apiFn = createRequest({
    url: '/api/user/list',
    method: 'post'
});
apiFn({ page: 1 }).then(res => { ... });
```

#### formatHelpers - 内置格式化工具函数

内置工具函数通过 `buildContext` 自动注入到表达式上下文中，可在 format、ifShow、disabled、show、dynamicDisabled 等所有支持 `${}` 表达式的字段中使用。

**函数列表**：

| 函数 | 说明 | 默认值 | 示例 |
|------|------|--------|------|
| `dateFormat(value, formatStr?)` | 日期格式化 | `'YYYY-MM-DD'` | `dateFormat(rowData.createTime, 'YYYY年MM月DD日')` |
| `calcAge(birthday)` | 计算年龄（基于当前日期） | - | `calcAge(rowData.birthday)` |
| `relativeTime(value)` | 相对时间描述 | - | `relativeTime(rowData.createTime)` → `"3分钟前"` |
| `now(formatStr?)` | 当前时间 | `'YYYY-MM-DD HH:mm:ss'` | `now('YYYY-MM-DD')` |
| `numberFormat(value, decimals?)` | 数字格式化 | `2` 位小数 | `numberFormat(rowData.amount, 2)` |
| `empty(value, placeholder?)` | 空值占位 | `'--'` | `empty(rowData.phone, '未填写')` |
| `truncate(value, maxLen, suffix?)` | 截断字符串 | `'...'` | `truncate(rowData.remark, 10)` |

**使用示例**：

```typescript
{
    "title": "创建时间",
        "dataIndex": "createTime",
        "format": "${dateFormat(rowData.createTime, 'YYYY-MM-DD HH:mm')}"
}
{
    "title": "年龄",
    "dataIndex": "birthday",
    "format": "${calcAge(rowData.birthday) + '岁'}"
}
{
    "title": "更新时间",
    "dataIndex": "updateTime",
    "format": "${relativeTime(rowData.updateTime)}"
}
{
    "title": "金额",
    "dataIndex": "amount",
    "format": "${'¥' + numberFormat(rowData.amount) + '元'}"
}
{
    "title": "备注",
    "dataIndex": "remark",
    "format": "${truncate(rowData.remark, 20)}"
}
```

这些函数也可在 ifShow、disabled 等表达式中使用：

```typescript
{
    "title": "审核人",
        "dataIndex": "auditor",
        "ifShow": "${!empty(rowData.auditor)}"  // 有审核人时才显示该列
}
```

#### transformHeaderActions

转换头部操作按钮：

```typescript
function transformHeaderActions(
    actions: ActionConfig[],
    context: TransformContext
): ActionConfig[]

// 处理内容
// 1. icon -> preIcon
// 2. 解析 disabled 表达式
```

#### transformColumns

转换表格列配置：

```typescript
function transformColumns(columns: any[]): BasicColumn[]

// 处理内容
// 1. ifShow 表达式转换为函数
// 2. format 表达式转换为函数
// 3. component 组件渲染
```

#### transformRowActions

转换行操作按钮：

```typescript
function transformRowActions(
    actions: CellActionConfig[],
    context: TransformContext,
    handleAction?: (actionConfig: HandleControlConfig, row: Recordable) => void
): CellActionConfig[]

// 处理内容
// 1. type 转换为 link
// 2. ghost 属性计算
// 3. disabled 和 ifShow 表达式解析
// 4. handleControl 转换为 onClick 或 popConfirm
```

#### transformFormSchemas

转换表单 Schema：

```typescript
function transformFormSchemas(schemas: FormSchema[]): FormSchema[]

// 处理内容
// 1. show/ifShow/dynamicDisabled 表达式转换
// 2. dataSource 转换为组件 API
// 3. componentProps 动态生成
```

**字段联动 - clearBy**：

Schema 支持 `clearBy` 字段，指定依赖字段列表。当依赖字段的值变化时，自动清空当前字段的值：

```typescript
{
    "field": "subCategoryId",
        "component": "ApiSelect",
        "clearBy": ["categoryId"]  // 当 categoryId 变化时清空 subCategoryId
}
```

**dataSource 自动注入 API**：

以下组件会自动获取 API 数据源，并注入动态参数：
- `ApiCascader`：使用 `initFetchParams` 注入参数
- `ApiRadioGroup`、`ApiSelect`、`ApiTransfer`、`ApiTree`、`ApiTreeSelect`：使用 `params` 注入参数

#### transformDescSchemas

转换 Description 详情 schema：

```typescript
function transformDescSchemas(
    schemas: DescSchemaItem[],
    context: TransformContext
): DescItem[]

// 处理内容
// 1. show 表达式转换为函数
// 2. component 自定义组件渲染（复用 descComponentMap 中的组件）
// 3. modelValue 双向绑定
```

**自定义组件渲染**：

```typescript
{
    "field": "status",
        "label": "状态",
        "component": "CellTag",
        "componentProps": {
        "mapOption": [
            { "label": "启用", "value": "1", "color": "green" },
            { "label": "禁用", "value": "0", "color": "red" }
        ]
    }
}
```

支持的组件列表详见 `Description/registerComp.ts`，目前包含：
- `CellTag` — 状态标签
- `CellCopyText` — 复制文本
- `CellStatusSwitch` — 状态开关

### 13.3 useSharedSelection

**功能**：跨组件状态共享

**返回值**：
```typescript
{
    selection: Reactive<Recordable>;           // 选中数据集合
    updateSelection: (key, data) => void;      // 更新选中数据
    getSelectionByKey: (key) => Recordable;    // 获取选中数据
    clearSelection: (key?) => void;            // 清空选中数据
}
```

**使用示例**：

```typescript
// 在 index.vue 中提供
const { selection, updateSelection, getSelectionByKey, clearSelection } = useSharedSelection();
provide('selectionStore', { selection, updateSelection, getSelectionByKey, clearSelection });

// 在子组件中注入
const { updateSelection } = inject('selectionStore');
updateSelection('tree', selectedNode);

// 访问选中数据
const treeSelection = getSelectionByKey('tree');
```

### 13.4 useCustomSlots

**功能**：处理自定义插槽

**返回值**：
```typescript
{
    slotKeys: ComputedRef<string[]>;     // 匹配的插槽键名
    replaceSlotKey: (key) => string;     // 替换插槽键名
    renderSlots: (scopeData) => VNode[]; // 渲染插槽
}
```

**使用示例**：

```typescript
const slots = createNamespacedSlots(useSlots());
const cellSlots = slots.withPrefix(() => activeKey.value + '-cell-');

// 过滤出 tab1-cell- 开头的插槽
// slotKeys.value = ['tab1-cell-name', 'tab1-cell-status', 'tab1-cell-action']

// 替换插槽键名（去掉前缀，传入 BasicTable 内部使用）
replaceSlotKey('tab1-cell-name');  // 返回 'name'
```

### 13.5 useComponentAction

**功能**：处理组件间动作通信。通过事件总线订阅 `comp:action` 事件，根据 `targetKeys` 匹配当前组件 key 来决定是否触发对应的动作。

**参数**：
```typescript
function useComponentAction(
    componentKey: MaybeRefOrGetter<string>,  // 当前组件的 key（支持 ref）
    actionMap: Record<string, (payload?: any) => void> // 动作映射表
)
```

**工作原理**：
1. 监听 `componentEventBus` 的 `comp:action` 事件
2. 事件载荷包含 `{ action, targetKeys, payload }`
3. 当 `targetKeys` 包含当前组件的 key（或 `'all'`）时，执行对应的 action 处理器
4. 支持动态 key 变化（如表格 Tab 切换），自动解绑旧 key 并绑定新 key
5. 组件卸载时自动解绑

**使用示例**：

```typescript
// 在 YwtTable.vue 中注册
useComponentAction(activeKey, {
    reload: () => reload()       // 收到 reload 动作时刷新表格
});

// 在 YwtForm.vue 中提交成功后触发
componentEventBus.emit('comp:action', {
    action: 'reload',
    targetKeys: ['tab1'],       // 只刷新 tab1 对应的表格
    payload: {}                  // 可选额外参数
});
```

### 13.6 useLinkAction

**功能**：处理 Link 类型的操作按钮，支持打开外部链接和内部路由页面。

**核心方法**：

```typescript
function handleLinkAction(
    actionConfig: HandleControlConfig,
    row?: Recordable
): void
```

**工作流程**：

```
用户点击 Link 按钮
        ↓
解析 path 表达式（支持 ${} 表达式）
        ↓
┌──────────────────────────────────────┐
│ 判断是否为外部链接（isHttpUrl）       │
├────────────────┬─────────────────────┤
│   是           │    否               │
│   ↓           │    ↓                │
│ window.open() │  检查路由是否已注册   │
│               │  ├─ 已注册 → 直接跳转 │
│               │  └─ 未注册 → 动态添加 │
│               │     路由并跳转       │
└────────────────┴─────────────────────┘
```

**支持两种跳转方式**：
1. **外部链接**：使用 `window.open()` 打开
2. **内部路由**：使用 `router.push()` 或 `router.replace()` 跳转

**刷新持久化**：

注册内部路由时，会自动将路由配置保存到 `sessionStorage`（key 为 `LINK_DYNAMIC_ROUTES`），配合 `linkGuard.ts` 路由守卫，**刷新页面后动态路由不会丢失**。

**配置示例**：

```json
// 外部链接
{
  "label": "查看文档",
  "key": "viewDoc",
  "icon": "ant-design:link-outlined",
  "handleControl": {
    "component": "Link",
    "componentProps": {
      "path": "https://example.com/docs",
      "target": "_blank"
    }
  }
}

// 内部路由 - 静态路径
{
  "label": "查看详情",
  "key": "viewDetail",
  "handleControl": {
    "component": "Link",
    "componentProps": {
      "path": "/detail/123",
      "goType": "push",
      "title": "详情页面"
    }
  }
}

// 内部路由 - 动态路径（支持表达式）
{
  "label": "查看详情",
  "key": "viewDetail",
  "handleControl": {
    "component": "Link",
    "componentProps": {
      "path": "${'/detail/' + rowData.id}",
      "goType": "push",
      "query": {
        "type": "${rowData.type}",
        "name": "${rowData.name}"
      },
      "title": "详情页面",
      "currentActiveMenu": "/parent-menu"
    }
  }
}
```

**componentProps 参数说明**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `path` | `string` | 是 | 页面路径，支持 `${}` 表达式 |
| `target` | `string` | 否 | 外部链接打开方式，默认 `_blank` |
| `goType` | `'push' \| 'replace'` | 否 | 路由跳转方式，默认 `push` |
| `title` | `string` | 否 | 页面标题（动态注册路由时使用） |
| `query` | `Recordable` | 否 | URL 查询参数，支持 `${}` 表达式 |
| `extraData` | `object \| string` | 否 | 传递给目标页面的配置数据 |
| `currentActiveMenu` | `string` | 否 | 当前激活的菜单路径 |

**动态路由注册**：

当目标路由未注册时，自动创建一个隐藏菜单的路由配置，并将配置持久化到 `sessionStorage`：

```typescript
const STORAGE_KEY = 'LINK_DYNAMIC_ROUTES';

function handleLinkAction(actionConfig, row) {
    // ... 解析 path 和 extraData ...

    if (!routeExists) {
        const routeName = `Link_${resolvedPath.replace(/[/?&=#.]/g, '_')}`;
        const serializedExtraData =
            typeof extraData === 'object' ? JSON.stringify(extraData) : extraData;
        const menuPath = currentActiveMenu || route.path;

        // 1. 动态注册路由
        router.addRoute({
            path: '',
            name: `${routeName}Parent`,
            component: LAYOUT,
            redirect: resolvedPath,
            meta: { hideMenu: true, single: true, title: componentProps?.title || '' },
            children: [{
                path: resolvedPath,
                name: routeName,
                component: () => import('@/views/auto-render/index.vue'),
                meta: {
                    extraData: serializedExtraData,
                    renderMode: RenderModeEnum.AUTO,
                    hideMenu: true,
                    title: componentProps?.title || '',
                    currentActiveMenu: menuPath,
                },
            }],
        });

        // 2. 持久化到 sessionStorage（刷新后可恢复）
        const routes = JSON.parse(sessionStorage.getItem(STORAGE_KEY) || '{}');
        routes[resolvedPath] = {
            extraData: serializedExtraData,
            title: componentProps?.title,
            currentActiveMenu: menuPath,
        };
        sessionStorage.setItem(STORAGE_KEY, JSON.stringify(routes));
    }

    // ... 跳转 ...
}
```

---

## 十四、工具函数

### 14.1 transformHelpers.ts

**功能**：提供表达式解析、取值、API 请求创建等核心工具函数。

**安全特性**：使用递归下降解析器（safeEval）替代 `new Function`，无代码注入风险，可通过政府项目安全扫描。

#### resolveExpression

取值 + 字符串插值，失败时原样返回。

```typescript
function resolveExpression(expression: string, context: TransformContext): any
```

**处理流程**：
1. 字符串完全包裹 `${...}` → 提取内部 JS 表达式用 `safeEval` 求值
2. 混合文本插值（如 `"用户: ${rowData.name}"`）→ 逐个替换 `${expr}` 部分
3. 路径格式字符串（以 `selection.`、`rowData.` 等开头）→ 尝试 JS 求值
4. 其他普通字符串 → 原样返回

**使用示例**：
```typescript
resolveExpression('${rowData.name}', { rowData: { name: '张三' } });
// 返回：'张三'

resolveExpression('rowData.price * rowData.count', { rowData: { price: 100, count: 20 } });
// 返回：2000

resolveExpression('用户: ${rowData.name}', { rowData: { name: '张三' } });
// 返回：'用户: 张三'

resolveExpression('普通文本', {});
// 返回：'普通文本'
```

**支持的路径前缀**（路径格式字符串会自动注入这些上下文）：
- `selection` - 选中数据
- `rowData` - 行数据
- `route` - 路由信息
- `formData` - 表单数据
- `userInfo` - 当前用户信息

#### parseExpression

执行任意 JS 表达式，支持调用 context 中的工具函数，失败返回 `defaultValue`。

```typescript
function parseExpression(
    expression: string | boolean | undefined | null,
    context: TransformContext,
    defaultValue: any = undefined  // 默认 undefined，而非 true
): any
```

**处理流程**：
1. 如果是布尔值，直接返回
2. 如果字符串包含 `${...}` 包裹，先剥离 `${` 和 `}`
3. 使用 `safeEval`（递归下降解析器）执行表达式，替代 `new Function`
4. 解析失败时返回 `defaultValue`

**与 resolveExpression 的区别**：
- `resolveExpression` 侧重"取值"，非表达式字符串会原样返回
- `parseExpression` 侧重"执行"，非表达式字符串会执行失败返回 `defaultValue`

**使用示例**：
```typescript
parseExpression("status === '1'", { rowData: { status: '1' } }, false);
// 返回：true

parseExpression('selection.tree.id', { selection: { tree: { id: '123' } } });
// 返回：'123'

parseExpression('dateFormat(rowData?.creteTime)', { rowData: { creteTime: '2026-01-15' }, dateFormat: fn });
// 返回：'2026-01-15'

parseExpression("${rowData.name + '-' + rowData.code}", { rowData: { name: '张三', code: '001' } }, '');
// 返回：'张三-001'

parseExpression("rowData.sex === '1' ? '男' : '女'", { rowData: { sex: '1' } });
// 返回：'男'
```

**支持的完整语法**：
```
属性访问        rowData.id、selection.categoryTree.id
可选链          rowData?.creteTime、selection?.areaTree?.areaLevel
方括号下标      arr[0]、obj['key']、arr?.[0]
比较            ===、!==、==、!=、>、>=、<、<=
逻辑            &&、||、!
三元            condition ? trueVal : falseVal
函数调用        dateFormat(rowData?.creteTime)
算术            +、-、*、/、%、()
字面量          数字、'单引号'、"双引号"、true、false、null、undefined
```

#### createRequest

创建 API 请求函数：

```typescript
function createRequest(config: DataSourceConfig): (params?: Recordable) => Promise<any>

// 示例
const apiFn = createRequest({
    url: '/api/user/list',
    method: 'post'
});
apiFn({ page: 1 }).then(res => { ... });
```

#### mergeRequestParams

合并请求参数，支持动态表达式解析和条件过滤：

```typescript
function mergeRequestParams(
    params: Recordable,
    extraParams: Array<{ paramsKey: string; valueKey: string; ifShow?: string }>,
    context: TransformContext
): Recordable
```

**处理流程**：
1. 复制基础参数
2. 遍历 extraParams，用 `parseExpression` 计算 `ifShow` 条件
3. 对通过条件过滤的参数，用 `resolveExpression` 解析 `valueKey`
4. 值为 `undefined` 时自动跳过，不污染请求参数

### 14.2 safeEval.ts

**功能**：安全表达式求值器，使用递归下降解析器替代 `new Function`，无代码注入风险，可通过政府项目安全扫描。

**设计原理**：

```
表达式字符串 "rowData.sex === '1' ? '男' : '女'"
        ↓
tokenize()  →  [{IDENTIFIER}, {DOT}, {IDENTIFIER}, {OPERATOR:==}, ...]
        ↓
parseExpr() → parseTernary → parseLogicalOr → parseEquality → ...
        ↓
求值结果  →  "男"
```

**词法解析（tokenize）**：
将表达式拆分为有意义的 token 单元：

| Token | 示例 | 说明 |
|-------|------|------|
| `NUMBER` | `123`, `3.14` | 数字 |
| `STRING` | `'男'`, `"key"` | 字符串（支持单引号和双引号） |
| `BOOLEAN` | `true`, `false` | 布尔值 |
| `NULL` / `UNDEFINED` | `null`, `undefined` | 空值 |
| `IDENTIFIER` | `rowData`, `dateFormat` | 变量名/函数名 |
| `DOT` / `OPTIONAL_CHAIN` | `.`, `?.` | 属性访问 |
| `LBRACKET` / `RBRACKET` | `[`, `]` | 方括号下标 |
| `OPERATOR` | `===`, `&&`, `+`, `!` 等 | 运算符 |

**语法解析（递归下降）**：
```
expression  → ternary
ternary     → logical_or ("?" expression ":" expression)?
logical_or  → logical_and ("||" logical_and)*
logical_and → equality ("&&" equality)*
equality    → relational (("==" | "!=" | "===" | "!==") relational)*
relational  → additive ((">" | ">=" | "<" | "<=") additive)*
additive    → multiplicative (("+" | "-") multiplicative)*
multiplicative → unary (("*" | "/" | "%") unary)*
unary       → ("!" | "-") unary | member
member      → primary (("?." identifier | "." identifier | "[" expression "]" | "?.[" expression "]") )*
primary     → NUMBER | STRING | BOOLEAN | NULL | UNDEFINED | IDENTIFIER | "(" expression ")"
```

**错误日志**：
表达式求值失败时，控制台会自动打印 `[safeEval]` 组日志（同表达式同阶段最多 3 次防刷屏）：

```
[safeEval] 求值异常
  表达式: /market/social-organization/edit
  上下文: { rowData: { ... } }
  详情: { error: "意外的 token", pos: 0, currentToken: ... }
```

### 13.3 componentEventBus.ts

**功能**：组件事件总线

**API**：
```typescript
componentEventBus.on(event, handler);    // 监听事件
componentEventBus.off(event, handler);   // 移除监听
componentEventBus.emit(event, payload);  // 触发事件
```

**使用场景**：
- 表单提交后刷新表格
- 单元格切换后刷新数据
- 跨层级组件通信

### 13.4 linkGuard.ts

**功能**：Link 动态路由恢复守卫。在路由 `beforeEach` 阶段拦截匹配到 404 的路由，从 `sessionStorage` 恢复之前注册的 Link 动态路由，解决刷新页面后动态路由丢失的问题。

**工作原理**：

```
页面刷新/直接访问 Link 路由
        ↓
router.beforeEach (createLinkGuard)
        ↓
匹配到 404 路由？
├── 否 → 正常放行
└── 是 → 检查 sessionStorage
    ├── 无配置 → 正常放行 (继续 404)
    └── 有配置 → 动态注册路由 + replace 重定向
                  ↓
            重新匹配 → 命中新注册的路由
```

**核心代码**：

```typescript
export function createLinkGuard(router: Router) {
    router.beforeEach((to, from, next) => {
        // 只有匹配到 404 路由时才检查
        const isPageNotFound = to.matched.some((r) =>
            r.name?.toString().startsWith(PAGE_NOT_FOUND_NAME),
        );
        if (!isPageNotFound) {
            next();
            return;
        }

        // 从 sessionStorage 恢复路由配置
        const storedRoutes = JSON.parse(sessionStorage.getItem(STORAGE_KEY) || '{}');
        const config = storedRoutes[to.path];

        if (!config) {
            next();
            return;
        }

        // 动态注册路由（与 useLinkAction 相同的注册逻辑）
        router.addRoute({
            path: '',
            name: `${routeName}Parent`,
            component: LAYOUT,
            redirect: to.path,
            meta: { hideMenu: true, single: true, title: config?.title || '' },
            children: [
                {
                    path: to.path,
                    name: routeName,
                    component: () => import('@/views/auto-render/index.vue'),
                    meta: {
                        extraData: config?.extraData,
                        renderMode: 'AUTO',
                        hideMenu: true,
                        title: config?.title || '',
                        currentActiveMenu: config?.currentActiveMenu,
                    },
                },
            ],
        });

        // replace 触发路由重新匹配
        next({ path: to.path, query: to.query, replace: true });
    });
}
```

**数据持久化**：

`useLinkAction` 在注册路由的同时，将路由配置持久化到 `sessionStorage`：

```typescript
const STORAGE_KEY = 'LINK_DYNAMIC_ROUTES';

// 注册路由时保存配置
const routes = JSON.parse(sessionStorage.getItem(STORAGE_KEY) || '{}');
routes[resolvedPath] = {
    extraData: serializedExtraData,
    title: componentProps?.title,
    currentActiveMenu: menuPath,
};
sessionStorage.setItem(STORAGE_KEY, JSON.stringify(routes));
```

**注册方式**：

在路由守卫中注册（已在 `src/router/guard/index.ts` 中引入）：

```typescript
import { createLinkGuard } from '@/components/PageRender/utils/linkGuard';

export function createGuard(router: Router) {
    createLinkGuard(router);
    // ... 其他守卫
}
```

---

## 十五、完整配置示例

### 15.1 基础配置

```json
{
  "categoryTreeConfig": {
    "key": "tree",
    "title": "科室分类",
    "headerActions": [
      {
        "label": "添加科室",
        "key": "add",
        "type": "primary",
        "icon": "ant-design:plus-outlined",
        "handleControl": {
          "component": "YwtModal",
          "slotComponent": "YwtForm",
          "slotComponentProps": {
            "key": "addForm",
            "dataSource": {
              "updateSetting": {
                "url": "/api/department/add",
                "method": "post"
              }
            },
            "schemas": [
              {
                "field": "name",
                "label": "科室名称",
                "component": "Input",
                "required": true
              }
            ]
          }
        }
      }
    ],
    "cellActions": [
      {
        "label": "编辑",
        "key": "edit",
        "icon": "ant-design:edit-outlined",
        "className": "edit-btn",
        "handleControl": {
          "component": "YwtDrawer",
          "slotComponent": "YwtForm",
          "slotComponentProps": {
            "key": "editForm",
            "dataSource": {
              "initSetting": {
                "url": "/api/department/get",
                "extraParams": [
                  { "paramsKey": "id", "valueKey": "${rowData.id}" }
                ]
              },
              "updateSetting": {
                "url": "/api/department/update",
                "method": "post",
                "extraParams": [
                  { "paramsKey": "id", "valueKey": "${rowData.id}" }
                ]
              }
            },
            "schemas": [
              {
                "field": "name",
                "label": "科室名称",
                "component": "Input",
                "required": true
              }
            ]
          }
        }
      },
      {
        "label": "删除",
        "key": "delete",
        "icon": "ant-design:delete-outlined",
        "className": "delete-btn",
        "handleControl": {
          "component": "Popconfirm",
          "componentProps": {
            "title": "确认删除该科室？"
          },
          "dataSource": {
            "updateSetting": {
              "url": "/api/department/delete",
              "method": "post",
              "extraParams": [
                { "paramsKey": "id", "valueKey": "${rowData.id}" }
              ]
            }
          }
        }
      }
    ],
    "dataSource": {
      "initSetting": {
        "url": "/api/department/tree",
        "method": "post",
        "labelField": "name",
        "valueField": "id",
        "childrenField": "children"
      }
    }
  },
  "tableConfig": {
    "defaultActive": "tab1",
    "tabs": [
      {
        "key": "tab1",
        "label": "全部医生",
        "icon": "ant-design:user-outlined",
        "dataSource": {
          "initSetting": {
            "url": "/api/doctor/list",
            "method": "post",
            "params": { "status": "1" },
            "extraParams": [
              { "paramsKey": "departmentId", "valueKey": "${selection.tree.id}" }
            ]
          }
        },
        "headerActions": [
          {
            "label": "添加医生",
            "key": "add",
            "type": "primary",
            "icon": "ant-design:plus-outlined",
            "disabled": "${!selection.tree || !selection.tree.id}",
            "handleControl": {
              "component": "YwtModal",
              "slotComponent": "YwtForm",
              "slotComponentProps": {
                "key": "addDoctorForm",
                "dataSource": {
                  "updateSetting": {
                    "url": "/api/doctor/add",
                    "method": "post",
                    "extraParams": [
                      { "paramsKey": "departmentId", "valueKey": "${selection.tree.id}" }
                    ]
                  }
                },
                "schemas": [
                  {
                    "field": "name",
                    "label": "医生姓名",
                    "component": "Input",
                    "required": true
                  },
                  {
                    "field": "title",
                    "label": "职称",
                    "component": "ApiSelect",
                    "componentProps": {
                      "dataSource": {
                        "initSetting": {
                          "url": "/api/dict/title",
                          "labelField": "name",
                          "valueField": "code"
                        }
                      }
                    },
                    "required": true
                  }
                ]
              }
            }
          }
        ],
        "cellActions": [
          {
            "label": "编辑",
            "key": "edit",
            "type": "primary",
            "icon": "ant-design:edit-outlined",
            "handleControl": {
              "component": "YwtModal",
              "slotComponent": "YwtForm",
              "slotComponentProps": {
                "key": "editDoctorForm",
                "triggerActions": [
                  { "type": "reload", "targetKeys": ["tab1"] }
                ],
                "dataSource": {
                  "initSetting": {
                    "url": "/api/doctor/get",
                    "extraParams": [
                      { "paramsKey": "id", "valueKey": "${rowData.id}" }
                    ]
                  },
                  "updateSetting": {
                    "url": "/api/doctor/update",
                    "method": "post",
                    "extraParams": [
                      { "paramsKey": "id", "valueKey": "${rowData.id}" }
                    ]
                  }
                },
                "schemas": [
                  {
                    "field": "name",
                    "label": "医生姓名",
                    "component": "Input",
                    "required": true
                  },
                  {
                    "field": "status",
                    "label": "状态",
                    "component": "RadioGroup",
                    "componentProps": {
                      "options": [
                        { "label": "启用", "value": "1" },
                        { "label": "禁用", "value": "0" }
                      ]
                    }
                  }
                ]
              }
            }
          },
          {
            "label": "删除",
            "key": "delete",
            "type": "danger",
            "icon": "ant-design:delete-outlined",
            "handleControl": {
              "component": "Popconfirm",
              "componentProps": {
                "title": "确认删除该医生？"
              },
              "dataSource": {
                "updateSetting": {
                  "url": "/api/doctor/delete",
                  "method": "post",
                  "extraParams": [
                    { "paramsKey": "id", "valueKey": "${rowData.id}" }
                  ]
                }
              }
            }
          }
        ],
        "componentProps": {
          "columns": [
            {
              "title": "姓名",
              "dataIndex": "name",
              "width": 150
            },
            {
              "title": "职称",
              "dataIndex": "title",
              "width": 120,
              "format": "${dictTransform.title[rowData.title] || rowData.title}"
            },
            {
              "title": "状态",
              "dataIndex": "status",
              "width": 100,
              "component": "CellStatusSwitch",
              "bindField": "status",
              "triggerActions": [
                { "type": "reload", "targetKeys": ["tab1"] }
              ]
            },
            {
              "title": "创建时间",
              "dataIndex": "createTime",
              "width": 180,
              "format": "${dayjs(rowData.createTime).format('YYYY-MM-DD HH:mm:ss')}"
            }
          ],
          "actionColumn": {
            "width": 200,
            "fixed": "right",
            "title": "操作"
          }
        }
      }
    ]
  }
}
```

---

## 十六、最佳实践

### 16.1 配置组织

1. **模块化配置**：将复杂配置拆分为多个文件
2. **常量提取**：提取重复使用的配置片段
3. **注释说明**：为关键配置添加注释

### 15.2 表达式使用

1. **优先使用路径格式**：`selection.tree.id` 比 `${selection.tree.id}` 更简洁
2. **复杂计算使用表达式**：`${!selection.tree || !selection.tree.id}`
3. **避免复杂逻辑**：复杂逻辑应在组件中处理

### 16.3 性能优化

1. **懒加载子节点**：树形表格使用 `getChild` 配置
2. **避免重复请求**：使用 `loadedNodes` 记录已加载节点
3. **合理设置缓存**：表单数据适当缓存

### 15.4 错误处理

1. **配置验证**：使用 `isValidConfig` 验证配置结构
2. **错误提示**：提供友好的错误信息
3. **降级处理**：配置失败时显示错误界面

### 16.5 调试技巧

1. **控制台日志**：关键位置添加 `console.log`
2. **网络请求**：检查请求参数是否正确
3. **配置打印**：打印解析后的配置对象

---

## 十七、常见问题

### Q1: 如何传递复杂参数？

使用 `extraParams` 数组：

```json
{
  "extraParams": [
    { "paramsKey": "id", "valueKey": "${rowData.id}" },
    { "paramsKey": "type", "valueKey": "selection.tree.type" },
    { "paramsKey": "status", "valueKey": "${formData.status || '1'}" }
  ]
}
```

### Q2: 如何实现字段联动？

使用 `clearBy` 配置：

```json
{
  "schemas": [
    {
      "field": "categoryId",
      "component": "ApiSelect",
      "componentProps": {
        "dataSource": { ... }
      }
    },
    {
      "field": "subCategoryId",
      "component": "ApiSelect",
      "clearBy": ["categoryId"],
      "componentProps": {
        "dataSource": {
          "initSetting": {
            "extraParams": [
              { "paramsKey": "parentId", "valueKey": "${formData.categoryId}" }
            ]
          }
        }
      }
    }
  ]
}
```

### Q3: 如何自定义单元格渲染？

使用作用域插槽。需要两步：

**1. JSON 列配置中声明 `slots.customRender`**：

```json
{
  "columns": [
    {
      "dataIndex": "name",
      "title": "姓名",
      "slots": { "customRender": "name" }
    },
    {
      "dataIndex": "status",
      "title": "状态",
      "slots": { "customRender": "status" }
    }
  ]
}
```

**2. 在 Vue 模板中提供同名插槽**，插槽名格式为 `{tabKey}-cell-{columnKey}`：

```vue
<PageRender>
  <!-- status 列 -> 插槽名 tab1-cell-status -->
  <template #tab1-cell-status="{ record }">
    <Tag :color="record.status === '1' ? 'green' : 'red'">
      {{ record.status === '1' ? '启用' : '禁用' }}
    </Tag>
  </template>

  <!-- name 列 -> 插槽名 tab1-cell-name -->
  <template #tab1-cell-name="{ record }">
    <a>{{ record.name }}</a>
  </template>
</PageRender>
```

> 注意：`slots.customRender` 的值必须与 `{tabKey}-cell-` 后面的 `columnKey` 一致，否则插槽不会生效。

### Q4: 如何触发跨组件刷新？

使用 `triggerActions`：

```json
{
  "triggerActions": [
    { "type": "reload", "targetKeys": ["tab1"] }
  ]
}
```

### Q5: 如何处理禁用状态？

使用 `disabled` 表达式：

```json
{
  "disabled": "${!selection.tree || !selection.tree.id}"
}
```

或函数形式：

```typescript
{
    "disabled": (action) => {
        return !selection.value?.tree?.id;
    }
}
```

---

## 十八、更新日志

### 18.1 v1.2.0 (2026-06-17)

- ✅ 新增 `YwtDescription.vue` 详情描述组件，支持配置驱动的详情渲染
- ✅ 新增 `Description/registerComp.ts`，独立管理详情自定义组件注册
- ✅ `useConfigTransform` 新增 `transformDescSchemas` 方法
- ✅ 模板支持 `header`/`footer` 命名空间插槽，扩展详情组件前后内容
- ✅ 更新 `PageRender` 组件架构图，增加 Description 相关节点

### 18.2 v1.1.0 (2026-06-09)

- ✅ 新增 `useLinkAction` hook，支持 Link 页面跳转动作（外部链接 + 内部路由）
- ✅ 新增 `CellTag` 单元格标签组件，支持状态颜色映射
- ✅ 新增 `useComponentAction` 事件总线机制，支持跨层级组件刷新
- ✅ 新增分类树节点按钮 `ifShow` 条件显示支持
- ✅ 新增表单 `clearBy` 字段清空依赖
- ✅ 优化弹窗/抽屉错误处理，使用 `awaitTo` 替代 try-catch
- ✅ 优化状态重置，`closeFunc` 返回 Promise

### 18.2 v1.0.0 (2026-06-05)

- ✅ 初始版本发布
- ✅ 支持分类树 + 表格布局
- ✅ 支持配置驱动渲染
- ✅ 支持表达式系统
- ✅ 支持弹窗和抽屉
- ✅ 支持表单组件
- ✅ 支持跨组件通信
- ✅ 支持作用域插槽

---

## 十九、维护者

- **开发团队**: 天府惠民 前端团队
- **最后更新**: 2026-06-09
- **技术栈**: Vue 3 + TypeScript + Vite + Ant Design Vue

---

## 附录：快速参考

### 表达式语法

| 格式 | 示例 | 说明 |
|------|------|------|
| 路径格式 | `selection.tree.id` | 直接访问上下文数据 |
| 模板字符串 | `${rowData.name}` | 支持复杂表达式 |
| 布尔表达式 | `${!selection.tree}` | 返回布尔值 |
| 计算表达式 | `${rowData.age + 1}` | 支持数学运算 |

### 上下文对象

| 对象 | 说明 | 示例 |
|------|------|------|
| `selection` | 选中数据 | `selection.tree.id` |
| `rowData` | 行数据 | `rowData.name` |
| `formData` | 表单数据 | `formData.status` |
| `route` | 路由数据 | `route.query.type` |

### 组件映射

| 组件名 | 用途 | 位置 |
|--------|------|------|
| `CellStatusSwitch` | 状态开关 | Table/CellStatusSwitch.vue |
| `CellTag` | 状态标签 | Table/CellTag.vue |
| `YwtForm` | 表单组件 | Form/YwtForm.vue |
| `YwtTable` | 表格组件 | Table/YwtTable.vue |
| `YwtModal` | 弹窗组件 | YwtModal.vue |
| `YwtDrawer` | 抽屉组件 | YwtDrawer.vue |

### Hooks 映射

| Hook | 用途 | 文件 |
|------|------|------|
| `useConfigParser` | 配置解析和验证 | hooks/useConfigParser.ts |
| `useConfigTransform` | 配置转换和参数处理 | hooks/useConfigTransform.tsx |
| `useSharedSelection` | 跨组件状态共享 | hooks/useSharedSelection.ts |
| `useCustomSlots` | 自定义插槽处理 | hooks/useCustomSlots.ts |
| `useComponentAction` | 组件间动作通信 | hooks/useComponentAction.ts |
| `useLinkAction` | 页面跳转处理 | hooks/useLinkAction.ts |

### Utils 映射

| 工具 | 用途 | 文件 |
|------|------|------|
| `transformHelpers` | 表达式解析和路径解析 | utils/transformHelpers.ts |
| `safeEval` | 安全表达式求值器（替代 new Function） | utils/safeEval.ts |
| `formatHelpers` | 内置格式化工具函数（日期、年龄、数字格式化等） | utils/formatHelpers.ts |
| `componentEventBus` | 组件事件总线 | utils/componentEventBus.ts |
| `linkGuard` | Link 动态路由恢复守卫 | utils/linkGuard.ts |

### 常用原子类

| 类名 | 说明 |
|------|------|
| `flex-center` | Flex 居中 |
| `flex-between` | Flex 两端对齐 |
| `flex-start` | Flex 起始对齐 |
| `gap-2` | 间距 0.5rem |
| `mr-2` | 右边距 0.5rem |
| `text-base` | 基础字体 |
