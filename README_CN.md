

# AI 驱动的前端交互能力增强包

> 通过构建时分析、运行时注入和统一协议，赋能前端 UI 组件 AI 交互能力的完整解决方案 — 当前聚焦 Vue + Element Plus。


## 目录

* [概述](#概述)
* [背景与目标](#背景与目标)
* [设计原则](#设计原则)
* [核心模块](#核心模块)

  * [构建时能力描述生成](#构建时能力描述生成)
  * [运行时能力注入与注册](#运行时能力注入与注册)
  * [AI 指令执行接口](#ai-指令执行接口)
* [调试与运行时监控面板](#调试与运行时监控面板)
* [统一协议：AICP](#统一协议aicp)
* [Element Plus 适配](#element-plus-适配)
* [插件示例（伪代码）](#插件示例伪代码)
* [未来工作与扩展](#未来工作与扩展)

---

## 概述

随着 AI 技术的发展，AI 不再仅限于后端算法，也逐步进入前端领域，提升用户体验和智能化水平。本项目提出一个框架，使前端 UI 组件具备“AI 可交互”能力，具体做法为：

* **构建阶段静态分析 Vue 组件**，提取能力信息并生成唯一标识。
* **运行时注入包装器**，暴露组件的所有属性和方法给 AI 使用。
* **定义统一的 AI 交互能力协议（AICP）**，实现 AI 对组件的统一控制和自动化。

目前聚焦 Vue + Element Plus，目标使 AI 能准确识别、理解并操作前端组件。

---

## 背景与目标

### 背景

* 现代前端应用包含大量且嵌套复杂的交互组件，AI 驱动自动化难度较大。
* AI 需要提前获知组件的层级、状态和事件，才能精准执行点击、填值、提交等操作。

### 目标

* 实现自动化提取与注册页面所有交互组件的能力，覆盖构建与运行时。
* 自动为组件生成唯一的 `data-ai-id`，避免人工指定。
* 输出单一 JSON 文件，描述完整能力树供运行时使用。
* 支持读取和控制组件运行时状态（如禁用、选项、数据等）。
* 便于后续支持其他框架与组件库。

---

## 设计原则

1. **构建时静态分析**：解析 Vue SFC 模板，自动生成 `data-ai-id` 和能力元数据。
2. **运行时属性挂载**：包装组件，暴露所有属性和方法。
3. **自定义组件显式增强**：支持手动注册包装。
4. **统一 JSON 输出**：生成能力树 JSON，提升运行时加载和查询效率。
5. **能力树驱动 AI 指令规划**：基于树结构构建上下文，执行依赖顺序指令。
6. **初期聚焦 Vue + Element Plus。**

---

## 核心模块

### 构建时能力描述生成

* 使用 `@vue/compiler-sfc` 和 `@vue/compiler-dom` 解析模板。
* 识别 Element Plus 及带有标识的组件，生成唯一 ID。
* 抽取类型、属性、事件、模型绑定等信息。
* 构建嵌套能力树 JSON。
* 输出至 `/public/ai-capabilities.json`。

### 运行时能力注入与注册

* 在 `app.use()` 时批量包装 Element Plus 组件。
* 通过 `attrs` 暴露组件所有状态和方法。
* 支持自定义组件手动包装注册。
* 维护全局注册中心，管理实例及能力函数（如 `getValue`、`setValue`、`isVisible` 等）。

### AI 指令执行接口

* 启动时加载能力 JSON，构建快速查询结构。
* AI 根据用户意图检索组件树。
* 生成有依赖顺序的执行指令集。
* 执行器按序调用组件能力完成自动交互。

---

## 调试与运行时监控面板

* 页面右下角悬浮按钮，点击展开调试面板。
* 使用 Element Plus `el-tree` 展示能力树。
* 节点根据可访问状态高亮。
* 选中节点显示详细属性、事件和状态。
* 面板内含 AI 指令输入框，实时触发 UI 操作调试。

---

## 统一协议：AICP

* 规范 AI 与前端组件交互的统一数据格式和接口。
* 支持多组件库统一接入。
* 定义组件 ID、类型、模型路径、动作列表、属性字典、子节点等字段。
* 统一运行时能力函数签名。
* 支持多层嵌套及依赖关系表达。

---

## Element Plus 适配

* 构建时自动添加 `data-ai-id`。
* 运行时包装暴露所有属性与事件。
* 将 Element Plus 属性映射到协议字段，保障 AI 理解。
* 计划支持更多组件库如 VXE-Table。

---

## 插件示例（伪代码）

```typescript
// vite-plugin-ai-capability.ts
import type { Plugin, ResolvedConfig } from 'vite';
import { parse } from '@vue/compiler-sfc';
import { parse as parseTemplate } from '@vue/compiler-dom';
import fs from 'fs';
import path from 'path';
import * as globModule from 'glob';

interface AICapability {
  id: string;
  type: string;
  filePath: string;
  modelPath?: string | null;
  actions: string[];
  props: Record<string, any>;
  children: AICapability[];
}

/**
 * AI Capability Description Generator Plugin
 * Used for static analysis of Vue components at build time,
 * generating AI capability description JSON
 */
export default function createAICapabilityPlugin(options = { enableDevMode: true }): Plugin {
  const aiCapabilityMap = new Map<string, AICapability>();
  const aiCapabilityTree: AICapability[] = [];
  let config: ResolvedConfig;

  return {
    name: 'vite-plugin-ai-capability',
    
    configResolved(resolvedConfig) {
      config = resolvedConfig;
    },
    
    async buildStart() {
      if (options.enableDevMode === false && !config.isProduction) return;
      await generateAICapabilities(config, aiCapabilityMap, aiCapabilityTree);
    },
    
    handleHotUpdate({ file }) {
      if (file.endsWith('.vue') && options.enableDevMode !== false) {
        setTimeout(() => {
          generateAICapabilities(config, aiCapabilityMap, aiCapabilityTree);
        }, 100);
      }
    }
  };
}

function traverseAST(
  nodes: any[], 
  filePath: string, 
  capabilityMap: Map<string, AICapability>, 
  capabilityTree: AICapability[], 
  parentId: string | null = null
) {
  if (!nodes) return;
  
  for (const node of nodes) {
    if (node.type !== 1) continue; // Only process ELEMENT nodes
    
    const isElComponent = node.tag?.startsWith('el-') || ['ElButton', 'ElInput'].includes(node.tag);
    const isVxeComponent = node.tag?.startsWith('vxe-') || ['VxeTable', 'VxeButton'].includes(node.tag);
    const aiIdProp = node.props?.find(p => p.name === 'data-ai-id');
    const hasAiId = !!aiIdProp;
    
    if (isElComponent || isVxeComponent || hasAiId) {
      const customId = hasAiId ? aiIdProp.value.content : null;
      const componentId = customId || `${filePath.replace(/\//g, '_')}_${node.tag}_${Date.now().toString(36)}`;
      
      const componentProps: Record<string, any> = {};
      const events: Record<string, boolean> = {};
      let modelValue: string | null = null;
      
      if (node.props) {
        for (const prop of node.props) {
          if (prop.type === 6) { // ATTRIBUTE
            componentProps[prop.name] = prop.value?.content || true;
          } else if (prop.type === 7 && prop.name === 'on') { // EVENT
            events[prop.arg.content] = true;
          } else if (prop.type === 7 && prop.name === 'model') { // MODEL binding
            modelValue = prop.arg?.content || 'modelValue';
          }
        }
      }
      
      const capability: AICapability = {
        id: componentId,
        type: node.tag,
        filePath,
        modelPath: modelValue,
        actions: Object.keys(events),
        props: componentProps,
        children: []
      };
      
      capabilityMap.set(componentId, capability);
      
      if (parentId) {
        const parent = findCapabilityById(capabilityTree, parentId);
        if (parent) {
          parent.children.push(capability);
        } else {
          capabilityTree.push(capability);
        }
      } else {
        capabilityTree.push(capability);
      }
      
      if (node.children?.length) {
        traverseAST(node.children, filePath, capabilityMap, capabilityTree, componentId);
      }
    } else if (node.children?.length) {
      traverseAST(node.children, filePath, capabilityMap, capabilityTree, parentId);
    }
  }
}

function findCapabilityById(tree: AICapability[], id: string): AICapability | null {
  for (const node of tree) {
    if (node.id === id) return node;
    if (node.children?.length) {
      const found = findCapabilityById(node.children, id);
      if (found) return found;
    }
  }
  return null;
}

async function generateAICapabilities(config: ResolvedConfig, aiCapabilityMap: Map<string, AICapability>, aiCapabilityTree: AICapability[]) {
  console.log('Starting AI capability description JSON generation...');
  aiCapabilityMap.clear();
  aiCapabilityTree.length = 0;
  
  const vueFiles = globModule.sync('src/**/*.vue', { cwd: config.root });
  for (const file of vueFiles) {
    const filePath = path.join(config.root, file);
    const source = fs.readFileSync(filePath, 'utf-8');
    try {
      const { descriptor } = parse(source);
      if (!descriptor.template) continue;
      const templateAST = parseTemplate(descriptor.template.content);
      traverseAST(templateAST.children, file, aiCapabilityMap, aiCapabilityTree);
    } catch (err) {
      console.error(`Failed to parse file ${file}:`, err);
    }
  }
  
  const aiCapabilityJSON = JSON.stringify({
    tree: aiCapabilityTree,
    map: Object.fromEntries(aiCapabilityMap)
  }, null, 2);
  
  const publicDir = path.join(config.root, 'public');
  if (!fs.existsSync(publicDir)) {
    fs.mkdirSync(publicDir, { recursive: true });
  }
  
  fs.writeFileSync(path.join(publicDir, 'ai-capabilities.json'), aiCapabilityJSON);
  
  console.log('AI capability description JSON generated successfully at public/ai-capabilities.json');
}

```

---

## 未来工作与扩展

* 支持 React、Angular、VXE、Ant Design 等更多框架和库。
* 持续完善 AICP，提升通用性与扩展能力。
* 优化运行时性能和调试体验。
* 结合更智能的 AI 模型，实现复杂指令理解与自动化。

---

欢迎贡献和使用！详情请查阅 [CONTRIBUTING.md](CONTRIBUTING.md) 和 [使用文档](docs/USAGE.md)。

---

**许可证：** MIT

---
