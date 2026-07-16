# LaserGRBL G-code 生成器 — 架构文档

版本 v0.1 · 2026-07-15

---

## 1. 目标与设计原则

构建一个 **Web 端的 LaserGRBL G-code 生成器**,支持栅格图 / 矢量图 / 参数化几何三类输入,具备:

- **最短路径 / 最省空程**的工具路径规划
- **可调参数**:激光功率 S、进给速度 F、分图层、passes、模式(M4/M3)、DPI/线间距等
- **接入 Claude API** 做自然语言配参、材料建议、参数化几何、诊断

### 核心原则(重要)

1. **算法归算法,LLM 归 LLM。** 最短路径是确定性优化问题(TSP 类),用算法求解——快、便宜、可复现。Claude 只负责"理解意图、翻译参数、生成几何、诊断",**不参与逐点路径计算,也不直接输出 G-code**。
2. **Claude 输出结构化 JSON,不输出 G-code。** 由确定性引擎把 JSON 编译成 G-code,保证机器安全与可复现。
3. **客户端为主,API 按需。** Claude API 只放在低频交互(配参 / 生成几何 / 诊断),不进热路径,控制单位成本。
4. **图层 = 加工工序。** 每层独立参数与加工顺序;切透永远最后。

---

## 2. 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                        浏览器前端 (SPA)                        │
│                                                               │
│  上传/输入   →  预览画布   →  参数面板   →  下载 .nc          │
│  (raster/     (toolpath      (S/F/图层/     (生成结果)         │
│   vector/      + 耗时预估)     passes...)                      │
│   parametric)                                                 │
└───────┬─────────────────────────────────────────┬────────────┘
        │                                          │
        │ (低频) 自然语言配参 / 生成几何 / 诊断      │ (高频) 本地转换
        ▼                                          ▼
┌──────────────────┐                    ┌──────────────────────────┐
│  Claude API 层    │  ── 结构化 JSON ──▶ │   转换核心 (Converter Core) │
│  (BFF/Serverless) │                    │                          │
│  - 意图理解        │                    │  1. 输入解析 (Parser)      │
│  - 参数翻译        │                    │  2. 工具路径规划 (Planner)  │
│  - 几何生成        │                    │  3. 路径优化 (Optimizer)    │
│  - 报错诊断        │                    │  4. G-code 编译 (Emitter)   │
└──────────────────┘                    └──────────────────────────┘
```

### 部署形态两选一

| 形态 | 说明 | 适用 |
|---|---|---|
| **纯客户端 + Serverless BFF** | 转换核心用 JS/WASM 跑在浏览器;仅 Claude 调用走一个轻量 Serverless(隐藏 API key)| 个人/爱好者首选:免服务器、隐私好、静态托管 |
| **客户端 + Python 后端** | UI 在浏览器,转换核心用 Python(Pillow/numpy/ezdxf)| 需要重度栅格处理、批量作业 |

> 说明:**API key 绝不能放前端。** 即便纯客户端,也必须用一个最小 Serverless 函数(Vercel/Cloudflare Workers)代理 Claude 请求。

---

## 3. 数据流(端到端)

```
输入文件/文字
   │
   ▼
[Parser] ──► 内部统一模型 (IR: paths[] + rasters[])
   │
   ├─(可选)─► [Claude: 意图理解] ──► 图层归类 + 参数建议 (JSON)
   │
   ▼
[Planner] ──► 每层工具路径 (raster scanlines / vector contours)
   │
   ▼
[Optimizer] ──► 空程最小化 (TSP + 2-opt) + 进入点/方向优化
   │
   ▼
[Emitter] ──► GRBL G-code (.nc) + 耗时预估
   │
   ▼
预览 + 下载
```

### 统一中间表示 (IR)

所有输入解析后归一到同一模型,后续阶段与输入类型解耦:

```
IR {
  contours: [ { points: [[x,y],...], closed: bool, layerId } ]   // 矢量/参数化
  rasters:  [ { bitmap, bbox, dpi, layerId } ]                    // 栅格
  bounds:   { minX, minY, maxX, maxY }
}
```

---

## 4. 模块拆分

### 4.1 Parser(输入解析)

| 输入 | 库(JS 侧) | 库(Python 侧) | 产出 |
|---|---|---|---|
| 栅格 (PNG/JPG) | Canvas API | Pillow / numpy | rasters[] |
| 矢量 (SVG) | svg 解析 / paper.js | svgpathtools | contours[] |
| 矢量 (DXF) | dxf-parser | ezdxf | contours[] |
| 参数化 | 内建几何生成器 | 内建 | contours[] |

要点:SVG 曲线(贝塞尔/圆弧)按弦高误差 **flatten 成折线**(容差可配,如 0.05mm),统一为 points。

### 4.2 Planner(工具路径规划)

- **矢量路径**:沿轮廓走。切割 vs 线雕不同(切割 M3 恒定 + passes;线雕 M4 动态)。
- **栅格扫描**:逐行扫描,灰度→S 映射。双向扫描 + overscan。
- **参数化**:生成几何后当矢量处理。

栅格关键参数:
- **线间距 / DPI**:决定分辨率与耗时
- **Overscan**:每行两端外延 2–5mm,让激光头到达全速后再出光,避免边缘过烧
- **双向扫描回差补偿**:消除"拉链状"错边

### 4.3 Optimizer(路径优化)—— 核心

**目标:最小化空程 (G0 travel)。** 见第 5 节算法细节。

### 4.4 Emitter(G-code 编译)

薄序列化层,把规划好的动作编译成 GRBL 文本。可配方言:

- S 上限(`$30`:1000 或 255)自动换算
- M3 恒定 / M4 动态
- 小数精度(建议 3 位)
- 去冗余点、共线合并(减小文件体积,防串口卡顿)

---

## 5. 路径优化算法(不用 LLM)

### 5.1 问题建模

把每条独立轮廓 / 每段扫描线当作一个"作业节点",求访问所有节点、总空程最短的顺序——**开放式 TSP(Travelling Salesman)**。

### 5.2 求解流程

```
1. 构建节点集:每条闭合轮廓 = 一个节点
   (记录其所有可能的"进入点";闭合路径可从任意点切入)
2. 初始解:最近邻 (Nearest Neighbor) 从当前激光头位置贪心构造
3. 局部优化:2-opt / Or-opt 迭代改善,直到收敛或超时
4. 联合优化进入点 + 遍历方向:
   - 对每条轮廓,选择使"上一条终点→本条进入点"空程最短的切入点
   - 允许反转轮廓方向
5. 同层内合并可连续加工的路径,减少抬笔
```

### 5.3 复杂度与工程取舍

- 节点数 < 2000:最近邻 + 2-opt 毫秒级到亚秒级,足够。
- 节点数很大:限制 2-opt 邻域(仅对空间近邻做交换,用 KD-tree 加速),或设时间预算。
- **分层求解**:每层独立跑 TSP,层间按加工顺序(见 §6)串接,切透层永远最后。

### 5.4 为什么不用 Claude 算路径

| 维度 | 算法 (2-opt) | Claude API |
|---|---|---|
| 速度 | 毫秒级 | 秒级 |
| 成本 | 免费(本地) | 按 token 计费 |
| 可复现 | 完全确定 | 每次可能不同 |
| 规模 | 数千节点 | 上下文受限 |

结论:路径优化 100% 用算法。Claude 的价值在别处(§7)。

---

## 6. 参数与图层模型

### 6.1 图层 = 加工工序

每层携带独立参数,并有明确加工顺序。典型三层:

| 顺序 | 图层 | 模式 | 典型参数 |
|---|---|---|---|
| 1 | 浅雕文字 | M4 动态 | 低 S、高 F、1 pass |
| 2 | 线雕轮廓 | M4 动态 | 中 S、中 F、1 pass |
| 3 | 切透 | M3 恒定 | 高 S、低 F、多 passes |

> **安全铁律:切透层永远最后。** 否则工件切下后掉落 / 位移,后续加工全部错位。

### 6.2 可调参数清单

**全局**
- 机器 S 上限 `$30`(1000 / 255)· 激光模式 `$32=1`
- 单位/坐标原点 · 加工原点(左下 / 中心)
- 框选预览(G0 走边框对位)· 气泵/风扇(M7/M8)
- 全局速度倍率

**每层**
- S(功率)· F(进给,mm/min,**注意不是 mm/s**)
- passes(重复次数)· 模式(M3/M4)
- 栅格:DPI/线间距、overscan、单/双向、灰度→功率(min/max/gamma)
- 矢量:切入点策略、方向、kerf 补偿

---

## 7. Claude API 调用点(差异化价值)

Claude **只在这些低频、非热路径的场景**被调用,且**统一输出结构化 JSON**:

| 场景 | 输入 | 输出 (JSON) |
|---|---|---|
| **自然语言配参** | "3mm 椴木板切透 + 文字浅雕" | 图层配置 + S/F/passes |
| **图层语义归类** | 图元列表 + 用户意图 | 每个图元 → layerId + 参数 |
| **参数化几何生成** | "生成 50mm 校准标尺" | 几何 contours 描述 |
| **材料测试图** | "椴木,S 20–100%,F 1000–6000" | S×F 方阵参数 |
| **报错诊断** | 现象("烧焦"/"切不透") + 当前参数 | 调整建议 |

### 7.1 关键实现约束

- **强制结构化输出**:用 tool use / JSON mode,让 Claude 返回严格符合 §8 schema 的 JSON。
- **不接受自由文本 G-code**:Claude 的输出必须经过校验后,再由 Emitter 编译。
- **成本控制**:仅在用户主动触发(点"智能配参"/"诊断")时调用;结果缓存到本地预设库。

### 7.2 调用示意(伪代码)

```python
# Serverless BFF —— 隐藏 API key
resp = claude.messages.create(
    model="claude-sonnet-5",
    max_tokens=1024,
    tools=[LAYER_CONFIG_TOOL],          # 强制结构化
    tool_choice={"type": "tool", "name": "emit_layer_config"},
    messages=[{"role": "user", "content": user_intent}],
)
layer_config = resp.tool_use.input     # 严格 JSON
validate(layer_config, SCHEMA)         # 校验后才交给 Emitter
```

---

## 8. JSON Schema(Claude 与引擎的契约)

引擎与 Claude 之间用这套 schema 通信。Claude 产出它,Emitter 消费它。

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "LaserJob",
  "type": "object",
  "required": ["units", "maxSpindle", "origin", "layers"],
  "properties": {
    "units":      { "enum": ["mm"], "default": "mm" },
    "maxSpindle": { "type": "integer", "enum": [255, 1000], "default": 1000,
                    "description": "机器 $30,用于 S 换算" },
    "laserMode":  { "type": "boolean", "default": true,
                    "description": "对应 $32=1" },
    "origin":     { "enum": ["bottom-left", "center"], "default": "bottom-left" },
    "frameProbe": { "type": "boolean", "default": true,
                    "description": "加工前 G0 走边框对位" },
    "air":        { "type": "boolean", "default": false,
                    "description": "M7/M8 气泵" },
    "layers": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "name", "mode", "power", "feed", "order"],
        "properties": {
          "id":    { "type": "string" },
          "name":  { "type": "string" },
          "order": { "type": "integer",
                     "description": "加工顺序,切透层数值最大(最后)" },
          "mode":  { "enum": ["engrave", "cut"],
                     "description": "engrave→M4 动态;cut→M3 恒定" },
          "power": { "type": "number", "minimum": 0, "maximum": 100,
                     "description": "功率百分比,编译时换算到 S" },
          "feed":  { "type": "number", "minimum": 1,
                     "description": "F,mm/min" },
          "passes":{ "type": "integer", "minimum": 1, "default": 1 },
          "raster": {
            "type": "object",
            "properties": {
              "dpi":         { "type": "number", "default": 254 },
              "overscan":    { "type": "number", "default": 3 },
              "bidirectional": { "type": "boolean", "default": true },
              "minPower":    { "type": "number", "default": 0 },
              "maxPower":    { "type": "number", "default": 100 },
              "gamma":       { "type": "number", "default": 1.0 }
            }
          }
        }
      }
    }
  }
}
```

功率换算:`S = round(power/100 * maxSpindle)`。例如 power=80、maxSpindle=1000 → `S800`。

---

## 9. G-code 输出规范与示例

### 9.1 文件结构

```
Header:  G21 (mm) · G90 (绝对) · G17 · [M7 气泵]
Body:    逐层;每层 G0 定位(S0) → G1 加工(S/F)
Footer:  M5 (激光关) · [M9 气泵关] · G0 回原点
```

### 9.2 矢量切割片段(M3 恒定,cut 层)

```gcode
G21
G90
M8              ; 气泵开(可选)
; ---- Layer: 切透 (order 3) ----
G0 X10.000 Y10.000 S0
M3
G1 X60.000 Y10.000 S1000 F1200
G1 X60.000 Y40.000 S1000 F1200
G1 X10.000 Y40.000 S1000 F1200
G1 X10.000 Y10.000 S1000 F1200
M5
M9
G0 X0.000 Y0.000
```

### 9.3 线雕片段(M4 动态,engrave 层)

```gcode
; ---- Layer: 线雕 (order 2) ----
G0 X20.000 Y20.000 S0
M4                          ; 动态功率:随速度缩放,防拐角过烧
G1 X50.000 Y20.000 S300 F3000
G1 X50.000 Y50.000 S300 F3000
M5
```

> **M4 vs M3:** M4 动态模式下,功率随实际速度自动缩放,在起停/拐角减速处功率同步下降,避免烧焦——雕刻优先用 M4;切割用 M3 恒定。

---

## 10. 你可能还没想到的功能(建议排期)

| 功能 | 价值 | 优先级 |
|---|---|---|
| **材料测试图生成器**(S×F 方阵) | 一键找最佳参数,激光玩家刚需 | 高 |
| **参数预设库**(材料→参数,存取) | 复用,减少 API 调用 | 高 |
| **路径预览 + 耗时预估**(优化前后对比) | 直观体现价值 | 高 |
| 功率软换算(自动适配 $30) | 防功率标定错 | 高 |
| 首件安全:框选对位 / 暂停点 | 防废件 | 中 |
| 断点续雕(从某行恢复) | 大件断电救援 | 中 |
| 批量作业 / 拼版(nesting) | 省料 | 低 |

---

## 11. 技术选型建议

| 层 | 纯客户端方案 | 后端方案 |
|---|---|---|
| 前端 | React/Vue + Canvas | 同左 |
| 矢量解析 | svgpathtools(WASM)/ paper.js | svgpathtools + ezdxf |
| 栅格处理 | Canvas + WASM | Pillow + numpy |
| 路径优化 | JS 实现 2-opt(或 WASM)| Python + 可选 OR-Tools |
| Claude 代理 | Serverless(Cloudflare Workers)| FastAPI 端点 |
| 托管 | 静态站 + Serverless | 容器 |

个人/爱好场景优先 **纯客户端 + Serverless BFF**:免服务器、隐私好、单位成本最低。

---

## 12. 成本与安全(投资视角)

- **Claude 只进低频交互**,不进逐帧生成的热路径;否则每次生成都烧 token,单位经济性不成立。
- 结果缓存到本地预设库,复用替代重复调用。
- **API key 只在服务端**,前端永不出现。
- 首件安全靠框选对位 + 加工顺序约束(切透最后),防止贵重材料报废。

---

## 附:里程碑建议

1. **MVP**:矢量输入 → TSP 优化 → M3/M4 编译 → 下载 .nc(无 Claude)
2. **v1**:加栅格输入 + 图层参数面板 + 预览/耗时
3. **v2**:接 Claude(自然语言配参 + 诊断)+ 材料测试图 + 预设库
4. **v3**:断点续雕、拼版、批量
```