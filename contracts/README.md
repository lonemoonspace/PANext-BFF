# contracts

Android App 与 BFF 共享的冻结契约。修改必须经过 Opus 关口，提交信息含 `[gate]`（CI 检查）。

- `golden/`：金标准用例（输入 → 期望输出的 JSON），Kotlin 测试与 BFF 的 TypeScript 测试读取同一批文件。由 T4.1 生成。
- 接口与数据形状的权威定义在 `bff/CONTRACT.md` 与 `bff/src/contract/`。

## 金标准文件格式（`golden/*.json`）

每个文件对应**一个被测函数**（或紧密相关的一组函数，用 `fn` 字段在 `input` 里区分），文件名为
`<kebab 主题>.<kebab 函数>.json`（例如 `ticket-policy.evaluate.json`）。

顶层结构：

```json
{
  "subject": "TicketPolicy.evaluate",
  "kotlin": "app/src/main/java/com/personalassistant/app/domain/TicketPolicy.kt",
  "cases": [
    {
      "name": "明天到期，白天首次检查只发 soon",
      "from": ["TicketPolicyTest.expiring tomorrow notifies soon once"],
      "input": {
        "transitPassUntil": "2026-09-24T23:59",
        "parkingPassUntil": "",
        "now": "2026-09-23T09:00:00+02:00",
        "previousKeys": []
      },
      "expected": {
        "notifications": [
          { "kind": "TRANSIT", "title": "乘车月票明天到期", "body": "有效期至 9月24日 23:59，记得在 Ruter 续买，并回到设置更新截止时间。" }
        ],
        "newKeys": ["TRANSIT:2026-09-24T23:59:soon"]
      }
    }
  ]
}
```

字段：

- `subject`：被测函数的显示名，Kotlin 与 TS 的 `describe` 都用它分组
- `kotlin`：对应的 Kotlin 源文件路径（供人工核对，运行器不解析它）
- `cases[]`：
  - `name`：用例名，**文件内唯一**（重复直接判文件无效）
  - `from`：对应的 Kotlin 测试，写成 `["类名.方法名", ...]`；新写的用例（Kotlin 本无对应测试）写 `[]`
  - `input`：喂给被测函数的输入，形状由该文件的被测函数决定（见各任务卡里的表格）
  - `expected`：**完整**输出，逐字段精确相等（不是部分断言）；输出对象必须列出全部字段

约定：

- 时刻一律写带偏移的 ISO 字符串（`2026-09-23T09:00:00+02:00` 或 `...Z`）；两端运行器按被测函数的需要把它换成
  `ZonedDateTime`（Oslo）/ `LocalDateTime`（Oslo 本地）/ `Instant` / `Date`。纯日期写 `yyyy-MM-dd`
- 枚举写成名字字符串（如 `"MORNING"`）；集合写成**按字符串升序排序**的数组；可空字段在 `expected` 里显式写 `null`
  （不省略）
- `input` 里可以省略「被测函数本身就有默认值」的字段，两端解码时都按各自的默认值补齐；`expected` 不允许省略字段
- 需要表示「外部调用抛异常」时，把该处的值写成 `{ "$error": "说明文字" }`（供有外部依赖注入的用例使用，如
  `TrainRepository.refresh`）
- 需要注入外部数据的用例（如 `TrainRepository.refresh`），外部调用按请求参数查表返回：表项的值为数据、`null`（无结果）
  或 `{ "$error": "..." }`（抛异常）。查不到的请求必须让该用例失败（「意外调用」），且这个失败不能被被测函数自身的
  异常处理吞掉：Kotlin 抛 `AssertionError` 或先记录、调用结束后再断言；TS 同理。查表用的对象键集合要严格校验，
  多余键让用例失败
- `expected.calls` 按字符串升序记录每一次外部调用（重复保留），并带上请求参数，例如
  `fetchBoth:<aStopId>,<bStopId>@<from>`、`fastestTrip:<from>-><to>@<at>`、`fetchStop:<stopId>@<from>`；时刻为 Oslo 偏移的
  ISO（与 `TimeUtils.isoOffset` 一致）
- 文件格式：2 空格缩进、LF 换行、文件末尾保留一个换行；仓库层面用 `.gitattributes` 固定
  `contracts/golden/*.json text eol=lf`，避免 Windows 端签出后行尾变化导致两端字节不一致
- 文件没有用例、或用例名重复：两端运行器都必须直接判文件无效（不是把某条用例判失败）

### `watched-line.status.json` 的补充格式说明

- 被测函数是 `WatchedLinePolicy.status`（Kotlin：`domain/WatchedLinePolicy.kt`；TS：`domain/watched-line.ts` 的同名函数，T9.3 接入）。
- `input.config`：`{ lineCode, stopA: { id, name }, stopB: { id, name } }`，对应 `UserSettings` 的六个 `watchedLine*` 字段。
- `input.callsA` / `input.callsB`：**Entur 原始形状**（不是 `WatchedLinePolicy.BusCall`），字段与
  `estimatedCalls { realtime aimedDepartureTime expectedDepartureTime cancellation destinationDisplay { frontText }
  serviceJourney { transportMode journeyPattern { line { publicCode } } quays { stopPlace { id name } } } }` 一致——
  这样同一批用例可以直接喂给两端各自的 Entur 响应解析器（Kotlin 的 `StopDeparture`、TS 的 `entur.ts` schema），
  两端各自按生产代码里同样的映射规则转成内部的 `BusCall` 再调用 `status`。省略的字段按各自解析器的默认值补齐
  （例如缺 `quays` 时按空列表处理）。
- `input.now`：带偏移的 ISO 字符串，两端换算到 Oslo 时区。
- `expected`：完整的 `BusStatus`（`boards`、`updatedAt`、`lineCode`），`updatedAt` 由测试运行器传入固定值
  （不是从 `input` 推导），两端运行器需要传同一个固定值才能让 `expected` 一致。

## 两端运行器

- Kotlin：`app/src/test/java/com/personalassistant/app/golden/Golden.kt` 的 `Golden.runGolden(file, run)`；
  金标准目录由系统属性 `golden.dir` 传入（`app/build.gradle.kts` 的 `testOptions.unitTests` 配置）。支持
  `-PgoldenRecord=true` 录制模式：把被测函数的实际输出写回 `expected`，其余字段原样保留
- TypeScript：`bff/test/golden/harness.ts` 的 `runGolden(file, run)`；测试文件用静态导入
  `import file from "../../../contracts/golden/xxx.json"` 读取，`describe(subject)` 下每条用例一个 `it(name)`

两端比较都在「解析后的结构」层面做（Kotlin 用 `JsonElement` 相等，TS 用
`expect(JSON.parse(JSON.stringify(actual))).toEqual(expected)`），因此浮点数、`undefined`
与类实例等语言差异不应出现在 `expected` 里——出现即视为格式错误。

## 录制流程（写新金标准文件时）

1. 先写好 `input`（`expected` 随便填一个占位值，格式对即可）
2. 跑 Kotlin 录制模式（`-PgoldenRecord=true`，只跑该文件对应的测试类），把实际输出写回 `expected`
3. **逐条人工核对**录制出的 `expected` 是否与对应 Kotlin 测试的断言一致（尤其是 `from` 里列出的用例）
4. 以非录制模式再跑一遍，确认全绿；TS 侧接入后同样要全绿
