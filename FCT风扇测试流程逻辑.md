# FCT 风扇测试流程逻辑（C# 上位机 + 仪器采集 + PLC 信号联动）

## 1. 整体角色划分

| 角色 | 职责 |
|------|------|
| C# 上位机 | 流程调度、仪器读写、数据判定、结果记录、与 PLC 握手 |
| 仪器层 | 电源、功率计、转速计/示波器、风量风速仪、声级计、耐压仪等，通过 SCPI/串口/Modbus 通信 |
| PLC | 工装夹具动作、到位检测、启动请求、结果接收、安全联锁 |

> 原则：**PLC 管“动”，上位机管“测和判”**。上位机不直接驱动气缸，只发请求位、收状态位。

---

## 2. 主流程状态机

```
Idle → 等待上料到位 → 读条码/选配方 → 初始化仪器
     → 安全项（耐压/绝缘）
     → 上电
     → 功能项（转向/启动）
     → 性能项（电流/功率/转速/风量/噪音/振动）
     → 下电
     → 汇总判定
     → 回传 PLC
     → 存数据
     → 复位等待下一台
```

每个状态必须具备三要素：

- **超时控制**
- **异常分支**
- **安全回退**（下电、复位输出、置安全态）

---

## 3. 与 PLC 的握手逻辑（核心）

### 3.1 信号定义

| 方向 | 信号 | 含义 |
|------|------|------|
| PLC → 上位机 | `TestRequest` | 请求测试（工位就绪） |
| PLC → 上位机 | `FixtureInPlace` | 夹具到位 |
| PLC → 上位机 | `SafetyOK` | 门关/急停/联锁正常 |
| PLC → 上位机 | `Ack` | 收到结果确认 |
| 上位机 → PLC | `Busy` | 测试进行中，禁止 PLC 动作 |
| 上位机 → PLC | `ResultOK` / `ResultNG` | 测试结果 |
| 上位机 → PLC | `TestComplete` | 本次测试结束 |
| 上位机 → PLC | `Fault` | 仪器/系统故障 |

### 3.2 握手时序（双握手，上升沿触发）

```
1. 轮询 PLC：TestRequest == 1 ?
2. 读到 1 → 检查 SafetyOK、FixtureInPlace
3. 置 Busy=1 并确认 TestRequest 已清零 → 进入测试
4. 执行测试流程
5. 写 ResultOK / ResultNG + TestComplete=1
6. 等 PLC 回 Ack=1 → 上位机清 TestComplete、清 Busy
7. 等 PLC 清 Ack → 回到等待状态，准备下一次
```

### 3.3 关键约束

- 每条位信号都要有**超时**（例如 5s 未收到 `Ack` 即报警）。
- 急停或门开：立即中断、下电、写 `Fault`，**不自动恢复**，等人工复位。
- 多工位时每工位独立一套握手位（`Station1_Request`、`Station1_Result`…），或按字打包传输。

---

## 4. 单项测试的通用执行逻辑

所有测试项统一骨架：

```
设定条件（电压 / PWM 占空比 / 负载）
  → 等待稳定（固定延时或稳定判据）
  → 连续采样 N 次（固定周期）
  → 计算（均值 / 最大 / 最小 / 峰峰值 / 中位数）
  → 与规格上下限比对
  → 记录原始值 + 判定
  → 返回 Pass / Fail
```

要点：

- 采样前**丢弃前若干点**，避开上电瞬态。
- 用**中位数或均值**降噪，必要时加滤波。
- 规格支持**双边（min~max）与单边（≤ / ≥）**。
- 不同型号使用不同 **Recipe 配方**：限值、电压点、采样次数全部配置化，不写死在代码里。

---

## 5. 风扇 FCT 测试项与顺序

顺序原则：**安全项 → 功能项 → 性能项**。安全项不过直接 NG，不再继续。

| 顺序 | 测试项 | 采集设备 | 说明 |
|------|--------|----------|------|
| 1 | 耐压 / 绝缘 | 耐压仪 | 安全项，不过则禁止上电 |
| 2 | 启动能力 | 电源 + 电流表 | 额定电压启动、最低启动电压 |
| 3 | 转向 / 极性 | 目视 / 传感器 | 确认送风方向正确 |
| 4 | 启动电流 / 堵转电流 | 电源 + 电流表 | 启动瞬间峰值 |
| 5 | 工作电流 / 功率 | 功率计 | 额定点及多个电压点 |
| 6 | 转速 RPM | 转速计 / FG 信号 / 示波器 | 可测不同 PWM 占空比下的转速 |
| 7 | 风量 / 风压 | 风洞 / 风速仪 | 需等气流稳定后采样 |
| 8 | 噪音 | 声级计 | 注意环境本底噪声与测距 |
| 9 | 振动 | 加速度计 | 取 RMS / 峰值 |
| 10 | 温升 | 温度传感器 | 连续运行后测绕组/轴承温度 |
| 11 | 调速响应 | 示波器 / 上位机 | PWM 阶跃响应时间、稳态误差（可选） |

> 产线节拍紧时，把耗时项（温升、噪音）与快速电性能项分站或分时并行。

---

## 6. 判定与重测逻辑

- 每个测试项独立 Pass/Fail，最后汇总。
- **Fail-fast**：安全项、短路类异常立即中断。
- 重测策略配置化：允许 N 次重测，按“最后一次通过”或“连续 2 次通过”判定；重测次数必须记录，防止掩盖不良。
- NG 分类：`仪器故障` / `产品不良` / `操作异常`，便于追溯。

---

## 7. 数据与异常处理

### 7.1 数据记录字段

| 字段 | 说明 |
|------|------|
| 条码 | 产品唯一标识 |
| 工位号 | Station / LineNo |
| 时间 | 开始/结束时间戳 |
| Recipe / 型号 | 当前配方 |
| 设定条件 | 电压、PWM、负载等 |
| 原始采样 | 全部采样点存档 |
| 计算值 | 均值/极值等 |
| 上下限 | 当前规格 |
| 判定 | Pass / Fail |
| 失败原因 | 失败项与错误码 |

### 7.2 异常处理

- 仪器通信失败：**重试 3 次**，仍失败判为**仪器故障并报警**，不得误判为产品 NG。
- 所有步骤使用 `CancellationToken` + 超时控制。
- `try / finally` 保证异常时也执行：**下电 → 复位输出 → 置安全态 → 释放仪器**。
- 日志分级：操作日志、通信日志、判定日志分开；原始采样单独存档用于事后分析。

---

## 8. C# 代码结构建议

```
ITestItem           // Id、前置条件、Setup / Execute / Teardown、Judge
TestSequence        // 按 Recipe 组装并顺序/并行执行测试项
IPlcClient          // ReadBit / WriteBit / ReadWord，封装握手与超时
IInstrument         // 电源、功率计、转速计… 各自实现，统一 Connect / Set / Read
RecipeProvider      // 按型号加载限值、电压点、采样参数
ResultRepository    // 落库 / 上传 MES
```

设计收益：

- 每个测试项**独立可测、可替换、可配置**。
- 流程编排只负责**顺序、握手、异常、安全**。
- 避免形成几千行的 `RunTest()` 巨型方法。

---

## 9. 典型伪代码（C# 骨架）

```csharp
public async Task RunStationAsync(StationContext ctx, CancellationToken ct)
{
    try
    {
        if (!await _plc.WaitRequestAsync(ctx.StationNo, ct)) return;   // 等 PLC 请求
        if (!_plc.IsSafetyOk()) { _plc.SetFault(); return; }

        _plc.SetBusy(ctx.StationNo, true);

        var recipe = _recipeProvider.Load(ctx.Barcode);
        var items = _sequence.Build(recipe);                          // 安全→功能→性能

        bool allPass = true;
        foreach (var item in items)
        {
            var r = await item.RunAsync(ctx, recipe, ct);             // 含超时与采样
            _resultRepo.Add(ctx.Barcode, item.Id, r);
            if (!r.Pass) { allPass = false; if (item.IsSafety) break; } // Fail-fast
        }

        _plc.SetResult(ctx.StationNo, allPass);
        await _plc.WaitAckAsync(ctx.StationNo, ct);
    }
    catch (OperationCanceledException) { _plc.SetFault(); throw; }
    catch (Exception ex) { _log.Error(ex); _plc.SetFault(); }
    finally
    {
        await _power.OffAsync();          // 下电
        _plc.SetBusy(ctx.StationNo, false);
        _instruments.ForEach(i => i.SafeReset());
    }
}
```

---

## 10. 落地检查清单

- [ ] 每个状态都有超时与异常出口
- [ ] 急停/门开时能立即下电并置 Fault
- [ ] 安全项排在性能和功能项之前
- [ ] 限值、电压点、采样参数全部走 Recipe，不写死
- [ ] 仪器故障与产品不良严格区分
- [ ] 原始采样数据可追溯
- [ ] 多工位互不阻塞、信号位独立
