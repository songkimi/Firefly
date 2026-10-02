---
title: "基于WPF的超市管理系统"
slug: SupermarketApp
published: 2026-09-11
order: 100
description: "WPF + MVVM + EF Core 实现的超市管理系统"
image: "images/SupermarketApp.png"
status: "published"
tags:
  - C#
  - dotnet
  - efCore
  - mvvm
  - SQL server
  - WPF
link:
  - label: "GitHub"
    icon: "fa7-brands:github"
    value: "https://github.com/songkimi/SupermarketApp"
---

# 超市管理系统（WPF + MVVM + EF Core）

一个基于 .NET 10 / WPF 的超市管理桌面应用，采用 MVVM 分层架构与 EF Core 数据访问，
覆盖商品管理、收银结算（事务扣库存）、经营报表、管理员权限与操作审计日志等完整业务闭环。

由早期 WinForm + JSON 版本重写而来：数据层从 JSON 文件升级为 EF Core + SQL Server，界面从 WinForm 升级为 WPF + MVVM。

## 功能

| 模块 | 说明 |
|---|---|
| 登录认证 | 密码使用 SHA256 + 每用户盐哈希存储，统一错误提示防止用户名枚举 |
| 商品管理 | 增删改查、关键字搜索、按分类筛选（IQueryable 链式查询，条件在数据库侧执行） |
| 收银结算 | 内存购物车，单事务完成「校验库存 → 扣减库存 → 写入订单主表与明细（含成交价快照）」，库存不足整体回滚 |
| 经营报表 | 月度营收、热销 Top5（LINQ 聚合 + LiveCharts2 柱状图） |
| 管理员管理 | 层级权限模型（Rank 1~5，高等级兼容低等级功能） |
| 操作审计 | OperationLog 表记录"谁在何时执行了什么操作"，支持按月份查询 |
| 运行日志 | Serilog 按天滚动文件 + 全局异常捕获（UI 线程、后台任务、程序级） |

## 技术栈与架构

```
.NET 10 · WPF · CommunityToolkit.Mvvm · EF Core 10 (SQL Server) · LiveCharts2 · Serilog · HandyControl

View (XAML / UserControl)          HandyControl 皮肤统一外观，侧边导航切换页面
   |
   | 数据绑定 / 命令
   v
ViewModel (ObservableObject)       异步命令，业务逻辑不写在 code-behind
   |
   | 构造函数注入（依赖注入）
   v
Service (接口 + 实现)               业务规则、事务、数据访问
   |
   v
EF Core -> SQL Server (LocalDB)
```

关键设计：

- 依赖注入：在 `App.OnStartup` 中搭建容器，ViewModel 只依赖接口，便于替换实现与单元测试
- 异步全链路：EF Core 的 `ToListAsync` / `SaveChangesAsync` 与异步事务，界面不阻塞
- 事务原子性：收银操作在单个事务内完成库存扣减与订单落库，失败整体回滚
- 密码安全：数据库只保存哈希值与盐，明文密码仅存在于用户输入瞬间
- 审计留痕：业务操作在 Service 层统一埋点写入日志表，界面可按月查询
- 界面与交互：HandyControl 提供控件模板与主题（品牌色通过覆盖资源键注入），侧边栏导航切换页面，操作反馈使用非阻塞的 Growl 通知替代弹窗；表格保持行虚拟化（有界高度），避免大数据量下的渲染卡顿

## 运行方式

环境要求：.NET 10 SDK（或 Visual Studio 2026）

```bash
git clone https://github.com/songkimi/SupermarketApp
cd SupermarketApp
dotnet run
```

- 首次运行自动建库并写入初始数据（含 4 条示例商品）。内置两个演示账号，权限等级不同：

  | 账号 | 密码 | 等级 | 可用功能 |
  |---|---|---|---|
  | `admin` | `123456` | 1 店员 | 收银 |
  | `BOSS` | `888888` | 5 超管 | 全部功能，含管理员授权 |

- 数据库使用 SQL Server LocalDB（`(localdb)\MSSQLLocalDB`），连接字符串见 `appsettings.json`
- 运行日志输出到 `bin/.../logs/app-YYYYMMDD.log`

### 打包为可执行程序

```bash
dotnet publish -c Release
```

- 产物在 `bin/Release/net10.0-windows/publish/`，双击其中的 `SupermarketApp.exe` 即可运行
- 这是**框架依赖**发布：目标机器需安装 .NET 10 桌面运行时，并具备 SQL Server LocalDB（随 Visual Studio 或 SQL Server Express 安装）
- 若要让没装 .NET 的机器也能直接运行，改用自包含发布（体积约 150 MB）：
  `dotnet publish -c Release --self-contained -r win-x64`

## 目录结构

```
SupermarketApp/
├── App.xaml(.cs)          入口：依赖注入容器、建库与初始化数据、登录后打开主窗口
├── LoginWindow.xaml       登录窗口
├── MainWindow.xaml        主窗口（HandyControl 侧边导航 + 内容区）
├── Views/                 页面组件（UserControl）
│   ├── GoodsView          商品管理
│   ├── SalesView          收银
│   ├── ReportView         报表
│   ├── AdminView          管理员管理
│   └── LogView            操作日志
├── ViewModels/            各页面的 ViewModel
├── Services/              业务服务（接口 + 实现）
│   ├── IAuthService / IGoodsService / ISalesService
│   ├── IReportService / IAdminService / ILogService
│   └── PasswordHasher     密码哈希工具
├── Models/                实体与数据模型
├── Data/                  DbContext 与初始化数据（Seed）
├── Themes/Styles.xaml     项目自定义资源键（品牌色等，由 HandyControl 模板引用）
├── docs/screenshots/      界面截图（本 README 引用）