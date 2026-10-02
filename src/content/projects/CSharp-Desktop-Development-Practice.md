---
title: "一切的开始——Winform超市管理程序"
slug: CSharp-Desktop-Development-Practice
published: 2026-07-15
order: 100
description: "第一次尝试使用 Winform 来开发桌面应用程序"
status: "published"
tags:
  - C#
  - Winform
  - Json
link:
  - label: "GitHub"
    icon: "fa7-brands:github"
    value: "https://github.com/songkimi/CSharp-Desktop-Development-Practice"
---

# WinForm+Json 超市管理系统
C# WinForm桌面程序，使用Json做数据持久化存档。

## 功能
- 初始化向导：指引第一次打开项目的用户创建超市存档
- 管理员模式：提供全面的后台管理功能，包括：
  - 1.商品管理：查看，添加，补货商品。
  - 2.库存预警：当商品库存过低时，系统会自动发出提醒
  - 3.账单查询：按日期范围查看历史销售记录和订单详细
  - 4.销量浏览：将每月的销量和收入总计在卡片中进行方便浏览
- 顾客购物模式：选购商品、加入购物车并完成结算
- 多套存档：切换/新建存档

## 运行方式
1. 使用Visual Studio打开解决方案 `.sln`
2. 目标框架：.NET 10
3. 直接编译运行即可

## 技术点
- 开发语言：C#
- 框架：.NET 10.0 & Windows Forms (WinForms)
- UI设计：原生WinForms控件以及自定义用户控件（UserControl）
- 数据存储：`System.Text.Json` 文件持久化
- 设计模式：观察者模式（用于UI事件广播）

## 首次使用
程序完成后，如果没有找到可用的超市存档，会引导用户新建存档。

## 最后
本项目是个人学习练习项目，由于本人专业为测绘大类，这个项目只是单纯用于c#基础练习并且不再考虑更新，欢迎其他新手作为练习。也欢迎 fork 和 star。
