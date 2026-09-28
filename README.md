# PANext BFF

[![Deploy to Cloudflare Workers](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/lonemoonspace/PANext-BFF/tree/main/bff)

一个跑在 Cloudflare Workers 免费版上的个人通勤助理后端：替手机按日程轮询挪威的公共数据源，把结果聚合成一个 JSON 给 Android App，并在服务端准点推送通知。

- **数据源**：Entur（火车 / 公交实时班次）、MET Norway（天气）、Google Routes（驾车路况）、football-data.org（皇马赛程）
- **通知**：列车异常、早间简报、皇马开赛 / 终场、车票到期四类，经 Firebase Cloud Messaging 推送，「至多一次」（宁可漏发，不重复）
- **多设备**：第一台设备用认领码认领服务器，其余设备用 10 分钟有效的配对码只读加入；设置在服务器上同步，带乐观锁
- **管理界面**：配置 Cloudflare Access 后在浏览器里查看调度与数据源状态、管理设备、改设置（校验 Access JWT，防跨站请求）
- **技术栈**：TypeScript、Hono、Drizzle ORM + D1、zod v4；测试用 vitest + `@cloudflare/vitest-pool-workers`（跑在真实 Workers 运行时）

## 目录

```
bff/
  CONTRACT.md     接口、表结构、调度与推送的完整规格
  DEPLOY.md       部署与接入指南（从零开始）
  src/            Worker 源码（src/contract/ 是 zod 契约）
  test/           测试与录制的真实响应
  admin-ui/       管理界面（原生 ES module，免构建）
  migrations/     D1 迁移
contracts/
  golden/         金标准用例：Android App（Kotlin）与本后端（TS）共用同一批输入与期望输出，保证两边行为一致
```

## 快速开始

```powershell
pnpm -C bff install
pnpm -C bff typecheck
pnpm -C bff test
```

点上面的按钮可以一键部署到你自己的 Cloudflare 账号（免费版即可）；完整步骤（含按钮做不到的部分：设置机密、App 认领）见 [bff/DEPLOY.md](bff/DEPLOY.md)。

## 关于这个仓库

这是从一个私有项目导出的**快照**，只包含后端与共用的金标准用例；配套的 Android App 不在这里。文档与注释里提到的 `app/…` 路径、`BRIEF.md` 等开发流程文档，都属于那个私有项目。

问题与建议欢迎提 Issue；这个仓库不接受直接合并，改动会在私有项目里完成后随下一次快照同步过来。
