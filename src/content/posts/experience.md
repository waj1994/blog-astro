---
title: 那些吃过的苦，踩过的坑
published: 2025-07-24
draft: false
tags: [坑]
toc: true
description: 开发中遇到的问题点记录，方便后续遇到查阅。
---

### vue3 中报`TypeError: Cannot read properties of null (reading 'emitsOptions')`

场景：vue3 项目二次封装组件，props 字段类型是组件，同时声明 slot，props 作为 slot 默认渲染内容，在传递 h 函数渲染的组件为 props 参数时报错

原因（询问 AI）：

> 因为 Vue
> 3 的 slot 机制会自动处理 vnode 的生命周期和上下文，确保插槽内容在父组件 props 变化时能正确地重新渲染和挂载。而你直接用 h 创建 vnode 并通过 props 传递时，Vue 没法自动追踪这些 vnode 的依赖和生命周期，导致 props 变化时，旧的 vnode 可能被复用或丢失上下文，访问到 null 的 component 实例，从而报 “emitsOptions” 的错误。

解决方案：使用 jsx 声明组件或者使用单组件
