---
title: "我收集的各种公有 NuGet 源"
publishDate: 2019-02-27 12:27:27 +0800
date: 2019-03-12 12:41:49 +0800
categories: dotnet nuget
position: knowledge
---

本文收集我发现的各种公共 NuGet 源。

---

<div id="toc"></div>

## 如何添加本文介绍的 NuGet 源？

请参见：

- [全局或为单独的项目添加自定义的 NuGet 源](/post/add-custom-nuget-source)

## 官方 NuGet 源

- 官方源
    - <https://api.nuget.org/v3/index.json>
- 官方离线本地源
    - `C:\Program Files (x86)\Microsoft SDKs\NuGetPackages\`

官方网站：<https://www.nuget.org/>

- dotnet feed
    - 包含 [.NET Foundation](https://github.com/dotnet) 中各种项目的日构建包 
    - <https://dotnetfeed.blob.core.windows.net/dotnet-core/index.json>

## NuGet 镜像

- 华为云 huaweicloud <https://mirrors.huaweicloud.com/>
    - <https://repo.huaweicloud.com/repository/nuget/v3/index.json>

- Telerik NuGet
    - <https://nuget.telerik.com/nuget>

## 其他 NuGet 源

- [MyGet](https://dotnet.myget.org/gallery)
    - 这是一个很激进的 NuGet 源，包含各种日构建包（其中包括 .NET Standard 或者 .NET Core 等库的日构建版本），所以如果你希望尝试最新的 API 最新的功能，最好设置此 NuGet 源。
    - <https://dotnet.myget.org/F/dotnet-core/api/v3/index.json>

## 私有 NuGet 源

- [Installing private NuGet packages · Gemfury Dev Center](https://gemfury.com/help/nuget-server/)
    - 这是私有 NuGet 源，你可以在上面取得独立的 NuGet 链接，上面只有自己私有的 NuGet 包

## NuGet 网站

呃……这部分只是 NuGet 网站而已，你可以在这里浏览 NuGet 包的各种信息，但是它不提供源。

- [FuGet Gallery - https://www.fuget.org/](https://www.fuget.org/)
    - 此项目开源：[praeclarum/FuGetGallery: An alternative web UI for browsing nuget packages](https://github.com/praeclarum/FuGetGallery)
