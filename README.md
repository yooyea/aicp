
# AI-Driven Frontend Interaction Capability Enhancement Package

> A unified solution to empower frontend UI components with AI interaction capabilities through build-time analysis, runtime injection, and a standardized protocol — focusing on Vue + Element Plus.



## Table of Contents

* [Overview](#overview)
* [Background and Goals](#background-and-goals)
* [Design Principles](#design-principles)
* [Core Modules](#core-modules)

  * [Build-time Capability Description Generation](#build-time-capability-description-generation)
  * [Runtime Capability Injection and Registration](#runtime-capability-injection-and-registration)
  * [AI Command Execution Interface](#ai-command-execution-interface)
* [Debugging and Runtime Monitoring Panel](#debugging-and-runtime-monitoring-panel)
* [Unified Protocol: AICP](#unified-protocol-aicp)
* [Element Plus Adaptation](#element-plus-adaptation)
* [Plugin Example (Pseudocode)](#plugin-example-pseudocode)
* [Future Work and Extensions](#future-work-and-extensions)

---

## Overview

As AI evolves, it moves beyond backend algorithms into the frontend to enhance user experience and intelligence. This project proposes a framework to make frontend UI components “AI-interactable” by:

* **Statically analyzing Vue components during build** to extract capabilities and generate unique IDs.
* **Injecting runtime wrappers** to expose all component properties and methods to AI.
* **Defining a unified AI Interaction Capability Protocol (AICP)** for consistent AI control and automation.

Currently focused on Vue + Element Plus, the framework aims to enable AI to accurately identify, understand, and manipulate frontend components.

---

## Background and Goals

### Background

* Modern frontend apps feature numerous, deeply nested interactive components, making AI-driven automation complex.
* AI requires prior knowledge of component hierarchy, state, and events to perform precise UI operations (clicks, inputs, submissions).

### Goals

* Automate extraction and registration of all interactive components’ capabilities at build and runtime.
* Automatically generate unique `data-ai-id` for components without manual work.
* Output a single JSON describing the full component capability tree for runtime usage.
* Support reading and controlling runtime component states (e.g., disabled, selected options, table data).
* Provide extensibility for other frameworks and component libraries.

---

## Design Principles

1. **Build-time Static Analysis:** Parse Vue SFC templates to generate `data-ai-id` and capability metadata.
2. **Runtime Attribute Mounting:** Wrap components to expose all props and methods through `attrs`.
3. **Explicit Enhancement for Custom Components:** Support manual registration and wrapping utilities.
4. **Unified JSON Output:** Single capability tree JSON file for efficient runtime loading and lookup.
5. **Capability-Driven AI Command Planning:** Use the tree structure to build context and execute AI instructions in dependency order.
6. **Initial Focus:** Vue + Element Plus.

---

## Core Modules

### Build-time Capability Description Generation

* Use `@vue/compiler-sfc` and `@vue/compiler-dom` to parse templates.
* Identify Element Plus and marked components; generate unique IDs.
* Extract component type, props, events, model bindings.
* Construct nested capability tree JSON.
* Output to `/public/ai-capabilities.json`.

### Runtime Capability Injection and Registration

* Apply wrappers to Element Plus components at `app.use()`.
* Expose full component states and methods via `attrs`.
* Support manual wrapping and registration for custom components.
* Maintain a global registry for component instances and capability functions (`getValue`, `setValue`, `isVisible`, etc.).

### AI Command Execution Interface

* Load capability JSON at startup, build maps for quick lookup.
* AI queries component tree based on user intent.
* Generate ordered commands respecting dependencies.
* Executor invokes component capabilities to automate UI interactions.

---

## Debugging and Runtime Monitoring Panel

* Floating button at bottom-right toggles a debug panel.
* Panel shows capability tree using Element Plus `el-tree`.
* Nodes highlight based on accessibility state.
* Detailed info displayed on node selection.
* Interactive AI command input to trigger UI actions for testing.

---

## Unified Protocol: AICP (AI Interaction Capability Protocol)

* Defines standardized data structures for AI-component interaction.
* Supports multiple component libraries via protocol adapters.
* Standardizes component IDs, types, model paths, actions, props, children.
* Defines runtime capability function signatures (`getValue`, `setValue`, `isVisible`, `isMounted`).
* Supports nested and dependent component relationships.

---

## Element Plus Adaptation

* Automatically add `data-ai-id` to all Element Plus components during build.
* Wrap components at runtime exposing all props and events.
* Map Element Plus properties into protocol fields for AI understanding.
* Plan to extend support to other component libraries like VXE-Table.

---

## Plugin Example (Pseudocode)

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

*(See full source in the `docs/` folder or source code.)*

---

## Future Work and Extensions

* Support React, Angular, VXE, Ant Design, and other libraries.
* Continuously improve AICP for greater flexibility and extensibility.
* Enhance runtime performance and debugging capabilities.
* Integrate with advanced AI models for better command understanding and complex automation.

---

If you want to contribute or use this project, please check the [Contributing Guide](CONTRIBUTING.md) and [Usage Documentation](docs/USAGE.md).

---

**License:** MIT

---
