# ParGen 完整链路：从工程师的需求到 ECU 里运行的代码

## 为什么需要这套系统？

博世的超声波泊车辅助系统（USS / ParkPilot）使用同一套代码部署到几十种不同的车型——宝马、奔驰、大众、通用、比亚迪等。每种车的传感器位置不同、车身尺寸不同、客户要求不同，但核心算法逻辑是一样的。

差异通过**参数化**处理。例如"前方障碍物报警距离"：

| 客户 | 值 |
|---|---|
| 宝马 | 800mm |
| 奔驰 | 600mm |
| 通用 | 1000mm |

不是给每个客户写一份不同的代码，而是同一份代码里读取一个参数值，不同车型加载不同的参数值。

整个链路解决的问题：**怎么从工程师脑中的"这个参数设成 800mm" → 最终变成 ECU 芯片里的一个 `Float32` 值**。

---

## 完整链路（7 个阶段）

```
阶段1        阶段2        阶段3         阶段4           阶段5         阶段6          阶段7
工程师       工程师       SHARCC工具     ParGen工具      C编译器       刷写工具       ECU运行
定义参数  →  设值/标定  →  合成XML    →  生成C代码    →  编译固件    →  烧录         →  车上跑
 (PDL)       (DCM)       (→XML)        (→.c .h .hex)   (→binary)     (→Flash/NVM)
```

---

### 阶段 1：工程师定义参数（PDL 文件）

**谁做的**：软件架构师 / 算法工程师

**做什么**：用 PDL（Parameter Definition Language）定义每个参数的"元数据"——名字叫什么、什么类型、什么单位、取值范围、属于哪个功能模块。

**具体例子**（PDL 文件）：

```java
package rbp.ehy.map;

public class MAP_ADS
{
  /**
   * 前方最高速度限制阈值
   * @UNIT mm
   */
  public Type_UInt16 SpeedLimitUpperThresholdFront;

  /**
   * 是否使用滤波后的距离差
   */
  public Type_Boolean UseFiltDeltaDist;
}
```

**类比**：PDL 就像数据库的**表结构定义**（`CREATE TABLE`）——只说有哪些字段、什么类型，不包含具体数据。

---

### 阶段 2：标定工程师设值（DCM 文件）

**谁做的**：标定工程师（在实际车上测试后调整参数值）

**做什么**：在 DCM 文件里给每个参数填写具体的数值。

**具体例子**（DCM 文件，给宝马 HC25 车型标定）：

```
FESTWERT rbp.ehy.map.MAP_ADS.SpeedLimitUpperThresholdFront
    EINHEIT_W "mm"
    WERT 800.0
END

FESTWERT rbp.ehy.map.MAP_ADS.UseFiltDeltaDist
    TEXT "true"
END

FESTWERTEBLOCK rbp.ehy.map.MAP_AmpFilterPdc_Veh.AmpTresh_ampMapFront 6
    WERT 105.0 115.0 115.0 125.0 135.0 145.0
END
```

**类比**：DCM 就像数据库的 `INSERT INTO` 语句——往已定义好的表结构里填数据。

**另外还有 PMAVARDEF 文件**：定义"哪个 DCM 文件对应哪个车型变体"。比如：

- `MAP_ApplClassProject_HC25.dcm` → 归属 `ASIC_HC25`（Major Variant）
- `MAP_ApplClassProject_EU.dcm` → 归属 `ASIC_EU`

---

### 阶段 3：SHARCC 工具合成参数集 XML

**谁做的**：自动化工具（SHARCC）

**输入**：PDL + DCM + PMAVARDEF（阶段 1 和 2 的产物）

**输出**：一个标准格式的参数集 XML（如 `ASIC_EU.xml`）

**做了什么**：把分散在多个 PDL、DCM、PMAVARDEF 文件中的信息，合并成一个结构化的 XML：

```xml
<ParameterSet>
  <Info><Version>1.0</Version>...</Info>
  <MajorVariantList>
    <MajorVariant>MajorVariant_A</MajorVariant>    <!-- 来自 PMAVARDEF -->
  </MajorVariantList>
  <ComponentList>
    <Component>
      <Name>rbp_subsys_comp_i</Name>               <!-- 来自 PDL 的 package -->
      <DefineList>
        <Define>
          <Name>scalarParameter_comp_i</Name>       <!-- 来自 PDL 的参数定义 -->
          <Type>Float32</Type>                      <!-- 来自 PDL 的类型 -->
          <Value>2.0</Value>                        <!-- 来自 DCM 的 WERT -->
        </Define>
      </DefineList>
      <ParameterClusterList>
        <ParameterCluster>
          <Name>classB</Name>                       <!-- 来自 PDL 的 class 名 -->
          <MemoryArea>ROM</MemoryArea>              <!-- 来自 PDL 的标注 -->
          <ParameterDefinitionList>
            <ParameterDefinition>
              <Name>scalarParameter_classB</Name>
              <Type>Float32</Type>
            </ParameterDefinition>
          </ParameterDefinitionList>
        </ParameterCluster>
      </ParameterClusterList>
    </Component>
  </ComponentList>
  <VariantImlementationList>
    <VariantImlementation>
      <ParameterValueList>
        <ParameterValue>2.0</ParameterValue>        <!-- DCM 的值，按参数顺序排列 -->
        <ParameterValue>true</ParameterValue>
      </ParameterValueList>
    </VariantImlementation>
  </VariantImlementationList>
</ParameterSet>
```

**类比**：SHARCC 就像一个 ETL 工具——从多个数据源抽取、转换、加载到一个统一的格式中。

---

### 阶段 4：ParGen 工具生成 C 代码

**谁做的**：ParGen（`pargen.exe`）

**输入**：

```
-s=segment_Chorus.xml          ← 段定义（内存布局）
-c=ASIC_EU_XCP.xml             ← 配置（大小端、对齐、输出路径）
    └─ <InputFiles>
         <File>ASIC_EU.xml     ← 参数集（阶段3的产物）
```

加载前先校验：用 `doc/ParkPilotParGenParameterSet_*.xsd` 验证参数集 XML 格式是否合法。

**输出和对应关系**：

| 输出文件 | 阶段3的哪部分 → 阶段4怎么变 |
|---|---|
| `rbp_par_switches_gen.h` | `<DefineList>` 中的 `<Define>` → C 的 `#define` 宏 |
| `rbp_par_struct_gen.h` | `<ParameterCluster>` + `<ParameterDefinition>` → C 的 `typedef struct` |
| `rbp_par_define_gen.h` | 所有 cluster 的索引 → `#define rbp_Idx_classB_du16 ((UInt16)(0))` |
| `rbp_par_access_gen.h` | 每个参数的读取方式 → `#define rbp_par_get_classB_scalarParameter_classB_f32 (...)` |
| `rbp_par_main_gen.c` | `<ParameterValueList>` 中的具体值 → ROM 常量表初始化 |
| `rbp_par_nvm_gen.c` | `MemoryArea=NVM_INIT` 的 cluster → NVM 读写函数 |
| `rbp_cal_section_gen.hex` | 所有 ROM 参数值 → Intel HEX 二进制（直接烧录用） |
| `gen_par_ST_CHORUS_*.xml` | 所有 NVM 参数值 → RAW 数据（给 NVM 管理工具用） |

#### 完整追踪例子 1：Define 参数

```
DCM 文件:
    FESTWERT rbp.ehy.map.MAP_ADS.SpeedLimitUpperThresholdFront
        WERT 800.0

      ↓ SHARCC 合成

参数集 XML:
    <Define>
      <Name>SpeedLimitUpperThresholdFront</Name>
      <Type>UInt16</Type>
      <Value>800</Value>
    </Define>

      ↓ ParGen 生成

rbp_par_switches_gen.h:
    #define rbp_SpeedLimitUpperThresholdFront_du16 ((UInt16)(800))

rbp_par_access_gen.h:
    #define rbp_par_get_SpeedLimitUpperThresholdFront_u16 \
        rbp_SpeedLimitUpperThresholdFront_du16
```

#### 完整追踪例子 2：ROM cluster 的值

```
DCM:  WERT 2.0    (scalarParameter_classB)
      WERT true   (logicalParameter_classB)
      WERT 1.0 2.0 3.0  (arrayParameter_classB)

      ↓ SHARCC → XML → ParGen

rbp_par_main_gen.c:
    {
      2.0f,   // scalarParameter_classB_f32      ← 就是 DCM 里的 2.0
      true,   // logicalParameter_classB_b       ← 就是 DCM 里的 true
      0,      // enumParameter_classB_u8
      10,     // datatypeParameter_classB_u16
      { 1.0f, 2.0f, 3.0f },  // arrayParameter_classB_pf32  ← DCM 里的数组
      1.0f,   // protectedParameter_classB_f32
      20.0f   // contParameter_virtual_classB_f32
    }
```

---

### 阶段 5：C 编译器编译固件

**谁做的**：交叉编译器（如 GHS / GCC for ARM / Renesas）

**输入**：ParGen 生成的 `.c` `.h` 文件 + 应用层代码（算法）+ BSW 底层代码

**做什么**：把所有 C 代码编译成 ECU 芯片能执行的机器码二进制文件。

应用层代码怎么使用生成的参数？直接通过宏：

```c
// 应用层算法代码（工程师手写的）
Float32 threshold = rbp_par_get_SpeedLimitUpperThresholdFront_u16;

if (obstacle_distance < threshold) {
    trigger_warning_beep();
}
```

工程师**不需要知道**参数值是 800 还是 600，也不需要知道它存在 ROM 还是 NVM。宏会自动解析到正确的位置。

---

### 阶段 6：刷写到 ECU

**谁做的**：产线刷写设备 / 标定工程师的诊断工具

**刷什么**：

- **编译出的固件二进制** → 写入 ECU 的程序 Flash（代码区）
- **`rbp_cal_section_gen.hex`** → 写入 ECU 的标定 Flash（CAL 区）—— 这是 ROM 参数的原始二进制
- **`gen_par_ST_CHORUS_*.xml`** 通过 MCM 工具 → 写入 ECU 的 NVM/EEPROM

---

### 阶段 7：ECU 运行

ECU 启动后：

1. ROM 参数直接从 Flash 读取（`rbp_par_main_gen.c` 里的常量表已经在编译时嵌入了）
2. NVM 参数通过 `rbp_par_nvm_gen.c` 里的读取函数从 EEPROM 加载到 RAM
3. 应用层代码通过 `rbp_par_access_gen.h` 的宏访问参数值
4. 泊车辅助系统根据这些参数值来决定行为

---

## 完整全景图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        工程师的工作（人工）                               │
│                                                                         │
│   PDL 文件                  DCM 文件              PMAVARDEF 文件         │
│   (参数结构)                (参数值)              (变体映射)              │
│   "这个参数叫什么            "宝马的值=800"         "HC25.dcm → ASIC_HC25" │
│    什么类型、什么单位"       "奔驰的值=600"                               │
└──────┬──────────────────────┬──────────────────────┬────────────────────┘
       │                      │                      │
       └──────────────────────┼──────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     SHARCC 工具（自动）                                  │
│                                                                         │
│   合并 PDL + DCM + PMAVARDEF → 生成参数集 XML                           │
│   输出: ASIC_EU.xml (3.8MB, 包含所有参数定义和值)                        │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
       ┌───────────────────────┼───────────────────────┐
       │                       │                       │
       ▼                       ▼                       ▼
 segment_Chorus.xml    ASIC_EU_XCP.xml           ASIC_EU.xml
 (段定义)              (配置 + 引用→)  ──────→   (参数集)
       │                       │
       └───────────┬───────────┘
                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     ParGen 工具（自动）                                  │
│                                                                         │
│  1. XSD 校验参数集 XML 格式                                              │
│  2. 反序列化 XML → C# 数据模型对象                                       │
│  3. 遍历对象，生成各类 C 代码                                            │
│                                                                         │
│  输出:                                                                   │
│  ├── PAR_CODE/  (.c .h 文件，给编译器)                                   │
│  ├── CAL/       (.hex 文件，给刷写工具)                                  │
│  ├── RAW/       (.xml 文件，给 NVM 管理工具)                             │
│  └── LOG/       (日志，给工程师排查问题)                                  │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
         .c .h 文件       .hex 文件        NVM .xml 文件
              │                │                │
              ▼                │                │
┌──────────────────────┐       │                │
│   C 编译器 (GHS等)    │       │                │
│   + 应用层算法代码    │       │                │
│   + BSW 底层代码      │       │                │
│   → 固件二进制        │       │                │
└──────────┬───────────┘       │                │
           │                   │                │
           ▼                   ▼                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       ECU 芯片                                          │
│                                                                         │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
│   │ 程序 Flash    │  │ 标定 Flash   │  │   NVM/EEPROM │                  │
│   │ (固件代码)    │  │ (CAL hex)    │  │  (NVM 数据)  │                  │
│   │              │  │              │  │              │                  │
│   │ 算法代码通过  │  │ ROM 参数的    │  │ 可在线更新的  │                  │
│   │ 宏读取参数 ──→│←─│ 二进制值     │  │ 参数值       │                  │
│   └──────────────┘  └──────────────┘  └──────────────┘                  │
│                                                                         │
│                    泊车辅助系统运行                                       │
│               "前方 800mm 有障碍物，滴滴滴"                               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 实习生的职责

### 在链路中的位置

```
阶段1        阶段2        阶段3              阶段4              阶段5-7
工程师       工程师       SHARCC工具          ParGen工具          编译/刷写/运行
定义参数  →  设值/标定  →  DCM→XML合成     →  XML→C代码生成    →  ...
 (PDL)       (DCM)       
                          ▲ 任务 1           ▲ 任务 2
                          用Python替代        用Python重写C#
                          SHARCC工具          ParGen工具
```

### 任务 1：替代 SHARCC 工具（阶段 3）

**现状**：SHARCC 是一个现有的闭源工具，用来把 DCM + PDL + PMAVARDEF → 参数集 XML。

**成果**：编写了 `WORKFLOW_DCM_TO_XML.md`（707 行），详细设计了如何用 Python 从零实现这个转换流程，包括：

- 解析 DCM 文件（提取参数值）
- 解析 PDL 文件（提取参数元数据：类型、描述、单位）
- 解析 PMAVARDEF 文件（提取变体映射关系）
- 合成符合 XSD schema 的参数集 XML

### 任务 2：用 Python 重写 ParGen（阶段 4）

**现状**：ParGen 是 C#（.NET Framework 4.0）写的，只能在 Windows 上原生运行。

**成果**：`pargen_py/` 目录——一个完整的 Python 版 ParGen，包含：

| 目录 | 对应 C# 源文件 | Python 实现 |
|---|---|---|
| `models/` | 数据模型 | `parameter_set.py`、`component.py`、`cluster.py`、`parameter.py`、`configuration.py`、`segment.py`、`data_types.py` |
| `core/` | 核心功能 | `xml_reader.py`（XML 读取）、`hashes.py`（哈希计算）、`wrapper_2x.py`（2.x 版本支持）、`file_utils.py`、`helpers.py` |
| `generators/` | 代码生成器 | `gen_defines.py`（switches）、`gen_header.py`（struct/access/define）、`para_server.py`（main_gen.c）、`nvm_par_access.py`（nvm_gen.c）、`calibration_hex.py`（hex 文件）、`resources_log.py`（日志）、`secure_flash.py`（PKI 安全） |
| 根目录 | `Program.cs` | `orchestrator.py`（主流程调度）、`cli.py`（命令行解析） |

### 为什么要做这两件事？

**技术原因**：

- C# + .NET Framework 4.0 是 2010 年的技术，Windows 专属，部署不便
- SHARCC 是闭源工具，依赖重，难以在 CI/CD（Docker/Linux）环境中运行
- Python 跨平台、易维护、团队更熟悉

**业务原因**：

- BAIC 项目的 CI/CD 流水线跑在 **Docker 容器（Linux）** 里（`compose.yaml`、`go_docker.sh`）
- 如果 ParGen 只能在 Windows 上跑，每次构建都需要 Windows 环境，这在现代 DevOps 中是瓶颈
- Python 版可以直接在 Linux Docker 容器里运行，无需 Mono 这样的中间层

### 工作全景对比

```
   当前状态（接手前）                        目标状态（成果）
   ──────────────────                      ──────────────────

   PDL + DCM + PMAVARDEF                   PDL + DCM + PMAVARDEF
          │                                       │
          ▼                                       ▼
   ┌─────────────┐                         ┌─────────────────┐
   │  SHARCC工具   │ ← 闭源，重依赖          │  Python DCM→XML  │ ← 新实现
   │  (Windows)   │                         │  (跨平台)        │
   └──────┬──────┘                         └───────┬─────────┘
          │                                        │
          ▼                                        ▼
   参数集 XML                              参数集 XML（格式完全一致）
          │                                        │
          ▼                                        ▼
   ┌─────────────┐                         ┌─────────────────┐
   │  ParGen C#   │ ← .NET 4.0, Windows     │  pargen_py       │ ← 新实现
   │  (pargen.exe)│                         │  (Python, 跨平台) │
   └──────┬──────┘                         └───────┬─────────┘
          │                                        │
          ▼                                        ▼
   .c .h .hex 文件                         .c .h .hex 文件（输出完全一致）
          │                                        │
          ▼                                        ▼
   Windows 编译环境                         Docker/Linux CI/CD 流水线
   （手动或 Jenkins）                       （全自动化）
```

### 完成的三件事

1. **理解**：深入学习了整个 7 阶段链路，从 PDL/DCM 到 ECU 运行
2. **移植验证**：用 Mono 在 Linux 上成功运行了原版 C# ParGen（包括修复路径 bug），证明了跨平台可行性
3. **重写**：用 Python 重写了 ParGen（`pargen_py/`）和 DCM→XML 转换（`WORKFLOW_DCM_TO_XML.md` 设计文档），目标是让整个工具链可以在 Linux/Docker 环境中原生运行

本质上是一个**工具链现代化**项目——把一个依赖 Windows + 闭源工具的遗留流程，改造成跨平台、开源、可容器化的现代流程。

---

## 模型转换详解：数据参数如何变成 C 代码

ParGen 的核心工作是一个**模型到模型再到代码**的转换过程。理解这个转换是理解整个工具链的关键。

### 转换全景

```
┌───────────────────────────────────────────────────────────────────────────┐
│                       模型转换全景（3 层）                                 │
│                                                                           │
│  数据模型层 (Data Model)                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │  XML 文本                                                          │   │
│  │  <ParameterSet>                                                    │   │
│  │    <Component>                                                     │   │
│  │      <ParameterCluster>                                            │   │
│  │        <ParameterDefinition>                                       │   │
│  │        <VariantImplementation>                                     │   │
│  └──────────────────────┬─────────────────────────────────────────────┘   │
│                         │ XmlSerializer.Deserialize()                      │
│                         ▼                                                  │
│  对象模型层 (Object Model)                                                │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │  C# 对象树                                                         │   │
│  │  cParameterSet                                                     │   │
│  │  ├── cParameterSetInfo (版本/时间戳)                                │   │
│  │  ├── List<string> MajorVariants (车型列表)                          │   │
│  │  ├── cDataTypes (枚举/自定义类型)                                   │   │
│  │  ├── List<cConstant> Constants (全局常量)                           │   │
│  │  └── List<cComponent> Components (组件列表)                         │   │
│  │       └── cComponent                                               │   │
│  │            ├── List<cDefine> Defines (宏定义)                       │   │
│  │            └── List<cParameterCluster> Clusters (参数簇)            │   │
│  │                 ├── string MemoryArea (ROM/NVM/RAM)                 │   │
│  │                 ├── List<cParameterDefinition> (参数元数据)          │   │
│  │                 └── List<cVariantImplementation> (各车型的值)        │   │
│  └──────────────────────┬─────────────────────────────────────────────┘   │
│                         │ 遍历 + 模板输出                                  │
│                         ▼                                                  │
│  代码模型层 (Code Model)                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │  .c / .h / .hex / .xml 文件                                        │   │
│  │  #define, typedef struct, const 初始化, 函数, 二进制                │   │
│  └────────────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────────────┘
```

### 第 1 步：XML → C# 对象（反序列化）

ParGen 使用 .NET 的 `XmlSerializer` 将 XML 直接映射到 C# 类。每个 XML 元素名对应一个 C# 类的属性名（通过 `[XmlElement]` / `[XmlArray]` 标注）：

```
XML 元素                          C# 类 / 属性
──────────                       ────────────
<ParameterSet>              →    cParameterSet
  <Info>                    →      cParameterSetInfo ParameterSetInfo
  <MajorVariantList>        →      List<string> MajorVariants
  <Constants>               →      List<cConstant> Constants
  <DataTypes>               →      cDataTypes DataTypes
  <ComponentList>           →      List<cComponent> Components
    <Component>             →        cComponent
      <Name>                →          string Name
      <Version>             →          string Version
      <DefineList>          →          List<cDefine> Defines
      <ParameterClusterList>→          List<cParameterCluster> Clusters
        <ParameterCluster>  →            cParameterCluster
          <Name>            →              string Name
          <MemoryArea>      →              string MemoryArea  (ROM/NVM_INIT/...)
          <VariantApplication> →           string VariantApplication
          <ParameterDefinitionList> →      List<cParameterDefinition>
            <ParameterDefinition> →          cParameterDefinition
              <Name>        →                  string Name
              <Type>        →                  string Type  (Float32/UInt16/...)
              <Number>      →                  string Number (数组大小)
              <Min>/<Max>   →                  string Min, Max
              <Offset>      →                  string Offset
              <Resolution>  →                  string Resolution
          <VariantImplementationList> →    List<cVariantImplementation>
            <VariantImplementation> →        cVariantImplementation
              <MajorVariant>→                  string MajorVariant
              <ParameterValueList>→            List<string> Values
```

**核心代码**（`Program.cs` 第 1104 行）：

```csharp
my_ps = XmlSerializerDeserialize<cParameterSet>(my_InputFileName);
```

一行代码就把整个 XML（几千行甚至几万行）变成了内存中的 C# 对象树。之后所有操作都是对这棵对象树的遍历和转换。

### 第 2 步：对象树 → C 代码（遍历 + 模板生成）

对象树加载完成后，ParGen 按顺序调用各个生成器（Generator），每个生成器遍历对象树的不同部分，输出不同的 C 文件：

```
cParameterSet 对象树
│
├──→ Gen_Defines.PrintGen_Switch_Defines()
│      遍历: Components → Defines
│      输出: rbp_par_switches_gen.h
│      规则: 每个 cDefine → #define rbp_{Name}_{类型后缀} (({Type})({Value}))
│
├──→ Gen_Header.PrintGenHeader()
│      遍历: Components → Clusters
│      输出: rbp_par_header_gen.h
│      规则: 外部声明 extern const ...
│
├──→ Gen_Header.PrintGenParStruct()
│      遍历: Components → Clusters → ParameterDefinitions
│      输出: rbp_par_struct_gen.h
│      规则: 每个 Cluster → typedef struct { 每个 ParameterDefinition → 字段 }
│
├──→ Gen_Header.PrintGenParDefine()
│      遍历: Components → Clusters (按顺序编号)
│      输出: rbp_par_define_gen.h
│      规则: 每个 Cluster → #define rbp_Idx_{ClusterName}_du16 ((UInt16)({索引}))
│
├──→ Gen_Header.PrintGenParAccess()
│      遍历: Components → Clusters → ParameterDefinitions
│      输出: rbp_par_access_gen.h
│      规则: 每个参数 → #define rbp_par_get_{Cluster}_{Name}_{后缀} (访问表达式)
│
├──→ ParaServer.PrintGenMain()
│      遍历: Components → Clusters → VariantImplementations → Values
│      输出: rbp_par_main_gen.c
│      规则: ROM cluster 的值 → const 结构体初始化列表 { val1, val2, ... }
│
├──→ CalibrationHex.PrintGenNEC_CAL_Section()
│      遍历: 同上，但输出二进制
│      输出: rbp_cal_section_gen.hex
│      规则: 值 → 按类型转字节 → 按地址排列 → Intel HEX 编码
│
├──→ NVM_PAR_Access.PrintGenNVMHeader() / PrintGenNVMC()
│      遍历: MemoryArea=NVM_INIT 的 Clusters
│      输出: rbp_par_nvm_header_gen.h, rbp_par_nvm_gen.c
│      规则: NVM cluster → 读写函数 + 默认值表
│
└──→ NVM_PAR_Access.PrintGenST_CHORUS_XML()
       遍历: NVM 参数值
       输出: gen_par_ST_CHORUS_*.xml
       规则: 值 → 字节序列 → XML 格式的 NVM 数据容器
```

### 第 3 步：转换规则详解——一个参数的完整变形

以一个具体参数 `scalarParameter_classB` (Float32, ROM, 值=2.0) 追踪它在每个生成器中的变形：

```
┌──────────────────────────────────────────────────────────────────────────┐
│ 输入：XML 中的一个 ParameterDefinition                                   │
│                                                                          │
│   <ParameterCluster>                                                     │
│     <Name>classB</Name>                                                  │
│     <MemoryArea>ROM</MemoryArea>                                         │
│     <ParameterDefinitionList>                                            │
│       <ParameterDefinition>                                              │
│         <Name>scalarParameter_classB</Name>                              │
│         <Type>Float32</Type>                                             │
│         <Number>1</Number>                                               │
│       </ParameterDefinition>                                             │
│     </ParameterDefinitionList>                                           │
│     <VariantImplementationList>                                          │
│       <VariantImplementation>                                            │
│         <MajorVariant>ASIC_EU</MajorVariant>                             │
│         <ParameterValueList>                                             │
│           <ParameterValue>2.0</ParameterValue>                           │
│         </ParameterValueList>                                            │
│       </VariantImplementation>                                           │
│     </VariantImplementationList>                                         │
│   </ParameterCluster>                                                    │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ rbp_par_struct_gen.h — 结构体定义                                        │
│                                                                          │
│   typedef struct {                                                       │
│     Float32 scalarParameter_classB_f32;  ← Name + 类型后缀               │
│     ...                                                                  │
│   } rbp_Type_classB_st;                  ← "rbp_Type_" + ClusterName     │
└──────────────────────────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────────────────────────┐
│ rbp_par_define_gen.h — 索引号                                            │
│                                                                          │
│   #define rbp_Idx_classB_du16 ((UInt16)(0))  ← Cluster 在组件中的序号     │
└──────────────────────────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────────────────────────┐
│ rbp_par_access_gen.h — 访问宏                                            │
│                                                                          │
│   #define rbp_par_get_classB_scalarParameter_classB_f32 \                │
│     rbp_par_ROMData_st.classB[rbp_Idx_classB_du16]                       │
│       .scalarParameter_classB_f32                                        │
│                                                                          │
│   ← "rbp_par_get_" + ClusterName + "_" + Name + "_" + 后缀              │
│   ← 访问路径: 全局ROM表.cluster数组[索引].字段名                          │
└──────────────────────────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────────────────────────┐
│ rbp_par_main_gen.c — 值初始化                                            │
│                                                                          │
│   const rbp_Type_par_ROMData_st rbp_par_ROMData_st = {                   │
│     {  // cluster: classB                                                │
│       {  // ASIC_EU                                                      │
│         2.0f,  // scalarParameter_classB_f32  ← Value + "f" 后缀         │
│         ...                                                              │
│       }                                                                  │
│     },                                                                   │
│   };                                                                     │
└──────────────────────────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────────────────────────┐
│ rbp_cal_section_gen.hex — 二进制值                                        │
│                                                                          │
│   2.0f → IEEE 754: 0x40000000 → 小端字节序: 00 00 00 40                  │
│   → Intel HEX: :04FC40000000004040                                       │
│     ← 地址由 segment_Chorus.xml 的 CALSection_StartAddress 决定           │
└──────────────────────────────────────────────────────────────────────────┘
```

### 类型后缀命名规则

ParGen 根据 `<Type>` 给 C 变量名加后缀，遵循 MISRA C 命名约定：

| XML Type | C 类型 | 变量后缀 | 值格式 |
|---|---|---|---|
| `Float32` | `Float32` | `_f32` | `2.0f` |
| `Float64` | `Float64` | `_f64` | `2.0` |
| `UInt8` | `UInt8` | `_u8` | `0` |
| `UInt16` | `UInt16` | `_u16` | `800` |
| `UInt32` | `UInt32` | `_u32` | `100UL` |
| `SInt8` | `SInt8` | `_s8` | `-1` |
| `SInt16` | `SInt16` | `_s16` | `-100` |
| `Boolean` | `Boolean` | `_b` | `true` / `false` |
| 枚举 | 枚举类型 | `_u8` | 枚举值(整数) |
| 数组 | 类型[] | `_p{后缀}` | `{ val1, val2 }` |

### 变体维度如何展开为 C 数据结构

XML 中的变体层级（MajorVariant × MinorVariant × SubVariant）在 C 中展开为**多维数组**：

```
XML 变体结构:
  MajorVariant = [ASIC_EU, ASIC_HC25]     ← 2 个车型
  classB:
    VariantApplication = SINGLE            ← 无 Minor/Sub 细分
    Values for ASIC_EU:  [2.0, true, ...]
    Values for ASIC_HC25: [3.0, false, ...]

                    ↓ ParGen 转换

C 结构体:
  typedef struct {
    Float32 scalarParameter_classB_f32;
    Boolean logicalParameter_classB_b;
    ...
  } rbp_Type_classB_st;

  // ROM 数据表: classB[MajorVariant 数量]
  {
    { 2.0f, true, ... },   // ASIC_EU 的值
    { 3.0f, false, ... },  // ASIC_HC25 的值
  }
```

对于有 Minor/Sub 变体的 cluster，数组维度更多：

```
XML 变体结构:
  classA:
    VariantApplication = MINOR_ALL
    MinorVariant = [Config1, Config2]
    SubVariant = [SubA, SubB]

                    ↓ ParGen 转换

C:
  // classA[Major数量][Minor数量][Sub数量]
  {
    {  // MajorVariant = ASIC_EU
      {  // MinorVariant = Config1
        { ... },  // SubVariant = SubA
        { ... },  // SubVariant = SubB
      },
      {  // MinorVariant = Config2
        { ... },
        { ... },
      },
    },
  }
```

### XSD 校验在转换中的角色

在反序列化之前，ParGen 用 XSD Schema 验证 XML 格式是否合法。这是一道"安全门"：

```
XML 输入
  │
  ▼
XSD 校验（doc/ParkPilotParGenParameterSet_*.xsd）
  │
  ├── 通过 → 继续反序列化
  │
  └── 失败 → 报错退出（"XML 格式不符合 schema"）
```

支持三个版本的 schema（1.0 / 2.0 / 2.1），按顺序尝试匹配。2.0/2.1 版本增加了 `TypeInModel`、`TypeInPlatform` 等新字段，`cWrapper_XML_2_x` 类负责将这些新字段转换为 1.0 兼容的内部表示。

### 转换过程中的关键决策点

| 决策 | 依据 | 影响 |
|---|---|---|
| 参数放在哪种存储器 | `<MemoryArea>` 标签值 | ROM → main_gen.c + hex; NVM → nvm_gen.c + NVM xml |
| 结构体字段顺序 | `<ParameterDefinition>` 在 XML 中的出现顺序 | 直接决定 struct 字段顺序和二进制布局 |
| 数组维度 | `<VariantApplication>` + 变体数量 | SINGLE/MINOR_ALL/SUB_ALL 决定数组嵌套层数 |
| 字节序 | `<Endianness>` 配置 | Big/Little 决定 hex 文件中的字节排列 |
| 结构体对齐 | `<StructureAlignment>` 配置 | 决定 padding 插入位置 |
| 内存段标记 | `segment_Chorus.xml` 的符号名 | 决定 `#pragma` 和内存放置 |

---

## V10 模块：Python 版 SHARCC 替代工具

### V10 是什么

V10 是 `asic_byd_ai_hc_appl` 项目中的一个 Python 命令行工具集，用于**替代 SHARCC 工具**完成阶段 3 的工作——将 PDL/DCM/pmavardef 原始文件转换为 ParGen 所需的参数集 XML。

在完整链路中的位置：

```
阶段1-2                 阶段3                          阶段4
PDL + DCM + PMAVARDEF → ┌──────────────────────────┐ → ParGen → C 代码
                         │ 原来: SHARCC (Java/GUI)   │
                         │ 现在: V10 (Python/CLI)    │
                         └──────────────────────────┘
```

### 为什么叫 V10

V10 是这个工具的第 10 个迭代版本。在 `asic_byd_ai_hc_appl` 项目中可以看到从 v1 到 v9 的历史输出目录（`v1_generated/`、`v2_test_output/` ... `v9_test_output/`），每个版本都在逐步完善 XML 生成的正确性和完整性。V10 是最终的整合版本，统一了所有功能。

### V10 的四大功能

```
v10/
├── export    — PDL/DCM/pmavardef → JSON 中间格式
├── generate  — JSON → 单车型 ParGen XML
├── combine   — JSON → 多车型合并 XML
└── add       — 变体管理（复制/新建/删除车型）
```

#### 1. export：原始文件 → JSON 中间格式

```
PDL 文件 ──→ model.json          (参数元数据：类型、量化、枚举)
DCM 文件 ──→ value/*.value.json  (参数值，每个 DCM 文件一个)
pmavardef ──→ variant.json       (变体层级定义)
```

这一步对应 SHARCC 的数据导入功能。引入 JSON 中间层的好处：
- JSON 比 PDL/DCM 格式更易解析，后续步骤不需要重新解析原始文件
- 可以增量更新——只改了一个 DCM 就只重新导出那个 value.json
- 调试友好——可以直接查看 JSON 内容

#### 2. generate：生成单车型 XML

```bash
python -m v10 generate --configmaster-dir ./configmaster_output --variant ASIC_MCHG
```

从 JSON 中间格式为指定车型生成 ParGen XML。核心类 `ParGenXmlGenerator` 的工作：

1. 从 `variant.json` 找到 ASIC_MCHG 引用了哪些 DCM 文件
2. 加载对应的 `value/*.value.json` 获取参数值
3. 从 `model.json` 获取参数元数据（类型、量化公式）
4. 构建 XML 结构：Component → Cluster → ParameterDefinition → VariantImplementation
5. 值转换管线：物理值 → 存储值（应用量化公式）

#### 3. combine：多车型合并

```bash
python -m v10 combine --configmaster-dir ./configmaster_output \
    --variants ASIC_MCHG ASIC_EU ASIC_HC25 --combine
```

| 模式 | 输出 | 用途 |
|---|---|---|
| `--combine`（默认）| 1 个 XML，包含所有车型 | 正式构建，一次生成所有车型的参数 |
| `--no-combine` | 每车型 1 个 XML | 调试，只看某个车型的参数 |

合并模式下，`_compare_vi_trees` 算法会自动去重：如果两个车型某个 Cluster 的所有参数值完全一致，它们共享同一个 `VariantImplementation` 节点，减少 XML 体积。

#### 4. add：变体管理

对应 SHARCC GUI 中的"Copy / New / Assign / Delete"操作，提供命令行和交互式向导两种方式：

```bash
# 复制一个已有车型
python -m v10 add copy variant.json ASIC_MCHG ASIC_MCHG_NEW --value-dir ./value

# 交互式向导（支持所有操作）
python -m v10 add new variant.json
```

### V10 的数据流

```
                原始文件                    JSON 中间格式                    输出
                ─────────                  ──────────────                  ─────

                                          ┌─ variant.json ──┐
*.pmavardef ──── export ──▶               │                 │
                                          │                 │
*.pdl ────────── export ──▶               ├─ model.json ────┼── generate ──▶ 单车型 XML
                                          │                 │
*.dcm ────────── export ──▶               ├─ value/*.json ──┼── combine ───▶ 合并 XML
                                          │                 │
                                          └─────────────────┼── add ───────▶ 更新后的
                                                                            variant.json
```

### V10 的技术实现

| 技术选择 | 说明 |
|---|---|
| 纯 Python 标准库 | 无第三方依赖，可在任何 Python 3.9+ 环境运行 |
| `dataclasses` | 数据模型定义，替代 SHARCC 的 Eclipse EMF |
| `xml.etree.ElementTree` | XML 构建，替代 Java 的 DOM API |
| `decimal.Decimal` + `ROUND_HALF_UP` | 量化计算，保证与 SHARCC Java 行为一致 |
| `argparse` | CLI 参数解析，替代 SHARCC 的 Eclipse GUI |

### V10 与 SHARCC 的对照

```
SHARCC (Java/GUI)                           V10 (Python/CLI)
──────────────────                         ─────────────────
Eclipse EMF 数据模型                        dataclasses 数据模型
GUI: 拖拽 DCM → SubVariant                 CLI: python -m v10 add new
GUI: 右键 → Copy MajorVariant              CLI: python -m v10 add copy
内部: BuildParGenXMLOperation              generator.py: ParGenXmlGenerator
内部: VariantImplementationCreator          generator.py: _add_variant_impl_pdl
内部: writeOptimizedVariantImplementation   generator.py: _compare_vi_trees
内部: QuantizationConverter                 quantization.py: QuantizationConverter
PDL/DCM 直接解析                            先导出 JSON，再从 JSON 生成
Windows 桌面应用                            跨平台命令行工具
需要安装 Eclipse + 插件                     只需 Python 3.9+
```

### V10 在 BAIC 项目中的实际使用

在 BAIC 项目中，V10 处理超过 60 个车型变体（ASIC_MCHG、ASIC_EU、ASIC_HC25 等），每个车型有几十个 DCM 文件。以 `ASIC_MCHG` 为例：

```bash
# 步骤 1：导出原始文件到 JSON
python -m v10 export

# 步骤 2：为单个车型生成 XML（用于调试）
python -m v10 generate --configmaster-dir ./configmaster_output --variant ASIC_MCHG

# 步骤 3：批量生成所有车型的合并 XML（正式构建）
python -m v10 combine --configmaster-dir ./configmaster_output \
    --variants ASIC_MCHG ASIC_EU ASIC_HC25 ... --combine --optimize

# 步骤 4：新增车型（当有新客户项目时）
python -m v10 add copy variant.json ASIC_MCHG ASIC_MCHG_NEW
```

V10 生成的 XML 与 SHARCC 生成的 XML 格式完全一致，可以直接交给 ParGen（阶段 4）处理。

### V10 在工具链现代化中的角色

```
   当前状态                                          目标状态
   ─────────                                        ─────────

   PDL + DCM + PMAVARDEF                            PDL + DCM + PMAVARDEF
          │                                                │
          ▼                                                ▼
   ┌─────────────┐                                  ┌─────────────┐
   │  SHARCC      │ ← Java GUI, Windows,             │  V10         │ ← Python CLI
   │  (闭源)      │   需要 Eclipse                    │  (开源)      │   跨平台
   └──────┬──────┘                                  └──────┬──────┘
          │                                                │
          ▼                                                ▼
   参数集 XML                                        参数集 XML（格式一致）
          │                                                │
          ▼                                                ▼
   ┌─────────────┐                                  ┌──────────────┐
   │  ParGen C#   │ ← .NET 4.0, Windows              │  pargen_py    │ ← Python
   └──────┬──────┘                                  └──────┬───────┘
          │                                                │
          ▼                                                ▼
   .c .h .hex                                        .c .h .hex（输出一致）
          │                                                │
          ▼                                                ▼
   Windows Jenkins                                   Docker/Linux CI/CD
```

V10（替代 SHARCC）+ pargen_py（替代 ParGen C#）= 完整的纯 Python 工具链，可以在 Docker 容器中原生运行，消除了对 Windows + Java Eclipse + .NET Framework 的依赖。

---

## 阶段 5 详解：编译——从 C 代码到固件二进制

### 编译的 4 个步骤

```
  .c 文件（ParGen 生成的 + 算法工程师手写的）
     │
     ▼
  ① 预处理器（Preprocessor）
     │  处理 #include、#define、#if
     │  把所有头文件展开、宏替换
     │  输出：纯 C 文件（没有任何 # 开头的东西了）
     ▼
  ② 编译器（Compiler）
     │  C 语法 → 汇编语言
     │  输出：.s 汇编文件
     ▼
  ③ 汇编器（Assembler）
     │  汇编语言 → 机器码（二进制）
     │  输出：.o 目标文件（地址还没确定）
     ▼
  ④ 链接器（Linker）
     │  把所有 .o 文件拼在一起
     │  根据链接器脚本分配物理地址
     │  输出：firmware.elf（完整的固件）
```

### ECU 芯片的存储器——三种"格子纸"

ECU 芯片的存储器就是一张很长的格子纸，每格存 1 字节，但不同区域的"纸质"不同：

| 存储器 | 地址范围（示例） | 特点 | 类比 |
|---|---|---|---|
| **Code Flash** | `0x00000000 - 0x001FFFFF` (2MB) | 掉电保持，只能整块擦除重写，寿命~100次 | 光盘：刻好了就不动，要改就重刻 |
| **CAL Flash** | `0x00FC4000 - 0x00FCBFFF` (32KB) | 同上，但单独分区，可独立刷写 | 活页：可以单独抽出来换 |
| **Data Flash / EEPROM** | `0x02000000 - 0x0200FFFF` (64KB) | 掉电保持，可按小块读写，寿命~10万次 | U盘：随便存删改 |
| **RAM** | `0xFEDE0000 - 0xFEDFFFFF` (128KB) | 掉电丢失，读写最快，无寿命限制 | 白板：随便写随便擦，关灯就没了 |

### 链接器脚本——给每个数据分配位置

链接器脚本定义"哪些数据放到哪片存储区域"：

```
MEMORY {
    CODE_FLASH : ORIGIN = 0x00000000, LENGTH = 2M
    CAL_FLASH  : ORIGIN = 0x00FC4000, LENGTH = 32K
    DATA_FLASH : ORIGIN = 0x02000000, LENGTH = 64K
    RAM        : ORIGIN = 0xFEDE0000, LENGTH = 128K
}

SECTIONS {
    .text       > CODE_FLASH    /* 函数代码 → Code Flash */
    .rodata_cal > CAL_FLASH     /* 标定常量 → CAL Flash */
    .bss        > RAM           /* 全局变量 → RAM */
}
```

### segment_Chorus.xml 和链接器的配合

`segment_Chorus.xml` 定义了段标记名，ParGen 在生成的 C 代码中使用这些标记，编译器和链接器根据标记把数据放到正确的地址：

```
segment_Chorus.xml:
    <CALDataBegin>
        <Line>RBP_MEM_ASW_RODATA_CAL_BEGIN</Line>
    </CALDataBegin>
        ↓
ParGen 生成的 C 代码:
    RBP_MEM_ASW_RODATA_CAL_BEGIN    ← 展开为 #pragma ghs section rodata=".rodata_cal"
    const rbp_Type_par_ROMData_st rbp_par_ROMData_st = { 2.0f, true, ... };
    RBP_MEM_ASW_RODATA_CAL_END
        ↓
链接器:
    .rodata_cal 段 → 放到 CAL_FLASH → 地址 0x00FC4000
```

### 三种参数存放位置

ParGen 根据 XML 中的 `<MemoryArea>` 标签决定参数放在哪种存储器：

```
MemoryArea=ROM       → 值写进 rbp_par_main_gen.c（const 常量表）
                     → 值写进 rbp_cal_section_gen.hex（二进制）
                     → 编译/刷写后放在 CAL Flash 里
                     → 程序运行时只读，改值需要重新刷 hex

MemoryArea=NVM_INIT  → 生成读写函数到 rbp_par_nvm_gen.c
                     → 值写进 gen_par_ST_CHORUS_*.xml
                     → 放在 EEPROM 里
                     → 程序运行时可读可写，诊断工具可在线修改

MemoryArea=RAM       → 只生成结构体定义
                     → 放在 RAM 里
                     → 掉电丢失，运行时随意读写
```

### 为什么同一个 ROM 参数既有 .c 文件又有 .hex 文件

```
同一个值 "800.0f":

  .c 文件（const 常量表）── 编译进 firmware.hex ──→ 完整构建时用
  .hex 文件（单独的二进制）── 直接刷 CAL Flash ──→ 只改参数时用（不用重编译固件）
```

代码通过宏读取 CAL Flash 地址上的值，不关心具体值是多少。改参数只需要重刷 32KB 的 hex 文件，不用重编译整个 2MB 固件。

---

## 阶段 6 详解：刷写——把二进制写进芯片

### 刷写方式

| 方式 | 场景 | 连接方式 |
|---|---|---|
| **JTAG 调试器** | 开发阶段 | USB → 调试器小盒子 → JTAG 线 → 芯片 |
| **UDS 诊断刷写** | 产线/售后 | 诊断仪 → CAN 总线 → ECU 的 Bootloader |

### JTAG 刷写（开发时）

调试器（如 Lauterbach）通过 JTAG 接口直接控制芯片的 Flash 控制器，擦除 → 写入 → 校验。类似用数据线给手机刷机。

### UDS 诊断刷写（产线/售后）

通过车上已有的 CAN 总线通信，不需要拆 ECU：

```
诊断仪                                  ECU
  │  "切换到编程模式"                      │
  │  ──── CAN 消息 ──────────────────→   │  ECU 进入 Bootloader
  │  "擦除 CAL Flash"                    │
  │  ──── CAN 消息 ──────────────────→   │  擦除 0x00FC4000 区域
  │  "写入数据..."                        │
  │  ──── CAN 消息（分包发送）────────→   │  写入 Flash
  │  "校验 CRC"                           │
  │  ──── CAN 消息 ──────────────────→   │  校验通过
  │  "重启"                               │
  │  ──── CAN 消息 ──────────────────→   │  从新固件启动
```

类似手机 OTA 升级，只不过用 CAN 总线而不是 WiFi。

### 刷写内容对应关系

| 刷什么 | 刷到哪里 | 来源 |
|---|---|---|
| firmware.hex（固件代码） | Code Flash | 编译器生成 |
| rbp_cal_section_gen.hex（标定参数） | CAL Flash | ParGen 生成 |
| gen_par_ST_CHORUS_*.xml（NVM 数据） | EEPROM | ParGen 生成，通过 MCM 工具写入 |

---

## 阶段 7 详解：ECU 上电启动

```
芯片上电 → CPU 复位
  │
  ▼
startup 启动代码（汇编，在 Flash 里）
  │  1. 设置栈指针（告诉 CPU "栈在 RAM 的哪里"）
  │  2. 初始化硬件（时钟、内存控制器）
  │  3. 把全局变量初始值从 Flash 复制到 RAM
  │  4. 未初始化变量在 RAM 中清零
  │  5. const 常量不动，留在 Flash（CPU 直接读 Flash）
  ▼
main()
  │  BSW 初始化：CAN 驱动、NVM Manager（从 EEPROM 加载数据到 RAM）
  ▼
主循环（永远不退出）
  │  每 5ms：
  │  ├── 读传感器
  │  ├── 读 ROM 参数（直接读 CAL Flash 地址）
  │  ├── 读 NVM 参数（读 RAM 中的 EEPROM 镜像）
  │  ├── 算法计算
  │  └── 发送结果（CAN 总线 → 仪表盘）
```

### 完整时间线

| 时间 | 发生了什么 | 涉及的存储器 |
|---|---|---|
| **工厂刷写** | firmware.hex → Code Flash | Code Flash |
| | cal.hex → CAL Flash | CAL Flash |
| | NVM 默认值 → EEPROM | EEPROM |
| **上电启动** | startup 复制变量初始值 | Flash → RAM |
| | NVM Manager 加载 EEPROM 数据 | EEPROM → RAM |
| **运行中** | 读 ROM 参数 | CAL Flash（读） |
| | 读/写 NVM 参数 | RAM ↔ EEPROM |
| | 算法计算 | RAM（读写） |
| **售后更新** | 只刷新 cal.hex | CAL Flash（擦除+重写） |
| | 诊断工具改 NVM | EEPROM（写） |



# V10 项目架构文档

## 1. 项目概述

V10 是一个 Python 命令行工具集，用于将汽车 ECU 参数标定数据（PDL/DCM/pmavardef）转换为 ParGen XML 格式。功能上对标 SHARCC（基于 Eclipse EMF 的 Java 桌面应用），将其核心逻辑用 Python 重新实现。

核心能力：
- **export** — 原始文件解析，生成中间 JSON 格式
- **generate** — 单车型 ParGen XML 生成
- **combine** — 多车型合并/拆分 XML 生成
- **add** — 车型变体的复制新增

## 2. 项目结构

```
v10/
├── __init__.py
├── __main__.py                  # python -m v10 入口
├── main.py                      # CLI 定义与子命令分发
│
├── export/                      # 阶段一：原始文件 → JSON
│   ├── configmaster_export.py       # PDL/DCM/pmavardef → JSON 转换
│   └── verify_configmaster.py       # 导出结果验证
│
├── generator/                   # 核心引擎
│   ├── models.py                    # 全局数据模型（dataclass）
│   ├── pdl_parser.py                # PDL 文件解析器
│   ├── dcm_parser.py                # DCM 文件解析器
│   ├── configmaster_loader.py       # JSON → 数据模型 加载器
│   ├── component.py                 # PDL Component/Cluster 索引构建
│   ├── quantization.py              # 物理值 ↔ 存储值 量化转换
│   ├── variant.py                   # 变体文件系统解析
│   ├── pmatarget.py                 # PMA Target / PID 配置
│   └── generator.py                 # ParGen XML 生成器（核心）
│
├── combine/                     # 多车型合并
│   └── combine_builder.py           # Combined / Non-combined 构建器
│
└── add/                         # 车型新增
    └── add_variant.py               # 复制车型 + 重命名引用
```

## 3. 软件架构

### 3.1 分层架构

```
┌─────────────────────────────────────────────────┐
│                   CLI 层                         │
│                  main.py                         │
│         argparse 子命令分发                       │
├─────────────────────────────────────────────────┤
│               业务逻辑层                          │
│  ┌───────────┐ ┌──────────────┐ ┌─────────────┐ │
│  │ generator  │ │combine_builder│ │ add_variant │ │
│  │  .py       │ │  .py          │ │  .py        │ │
│  └─────┬─────┘ └──────┬───────┘ └─────────────┘ │
│        │               │                          │
│        │    ┌──────────┘                          │
│        ▼    ▼                                     │
│  ┌─────────────────────────────────────┐          │
│  │         ParGenXmlGenerator          │          │
│  │  XML 结构构建 + 值转换 + 去重合并    │          │
│  └─────────────┬───────────────────────┘          │
├────────────────┼────────────────────────────────┤
│            基础设施层                              │
│  ┌─────────┐ ┌┴────────────┐ ┌───────────────┐  │
│  │component │ │quantization │ │variant        │  │
│  │索引构建   │ │量化计算      │ │变体解析       │  │
│  └─────────┘ └─────────────┘ └───────────────┘  │
│  ┌─────────────────────────────────────────────┐ │
│  │              models.py                       │ │
│  │         全局 dataclass 数据模型               │ │
│  └─────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────┤
│              数据加载层                            │
│  ┌───────────┐ ┌───────────┐ ┌────────────────┐ │
│  │pdl_parser  │ │dcm_parser │ │configmaster    │ │
│  │PDL 解析    │ │DCM 解析    │ │_loader JSON加载│ │
│  └───────────┘ └───────────┘ └────────────────┘ │
└─────────────────────────────────────────────────┘
```

### 3.2 数据流

```
原始文件                    中间格式 (JSON)                    输出
─────────                  ──────────────                    ─────

                         ┌─ variant.json ──┐
*.pmavardef ──export──▶  │                 │
                         │                 │
*.pdl ────── export──▶   ├─ model.json ────┼── generate ──▶ 单车型.xml
                         │                 │
*.dcm ────── export──▶   ├─ value/*.json ──┼── combine ───▶ 合并.xml / 拆分.xml
                         │                 │
                         └─────────────────┼── add ───────▶ 更新后的 variant.json
                                                            + 复制的 value.json
```

## 4. 数据模型

### 4.1 变体层级（pmavardef / variant.json）

```
VariantDefinition
├── all_major_variant_dcms: List[str]     # 所有车型共享的 DCM
└── major_variants: List[MajorVariant]
    └── MajorVariant                       # 车型（如 ASIC_MCHG）
        ├── common_dcm_files: List[str]    # 该车型的公共 DCM
        └── modules: List[VariantModule]
            └── VariantModule              # 模块
                └── minor_variants: List[MinorVariant]
                    └── MinorVariant       # 小变体
                        └── sub_variants: List[SubVariant]
                            └── SubVariant # 子变体
                                └── dcm_files: List[str]
```

### 4.2 参数定义（PDL）

```
PdlPackage                              # 一个 .pdl 包
├── classes: List[PdlClass]             # 参数类（= Cluster）
│   └── PdlClass
│       └── parameters: List[PdlParameter]
│           └── PdlParameter
│               ├── type_name / type_kind
│               ├── init_expression      # 默认值表达式
│               ├── quantization_ref     # 量化引用
│               └── is_virtual / is_constant / is_protected
├── enumerations: List[PdlEnumeration]
├── quantizations: List[PdlQuantization]
│   └── factor_text / factor_value       # 量化因子
├── units: List[PdlUnit]
├── custom_types: List[PdlCustomType]    # 自定义类型（范围、基类型）
├── memory_defs: List[PdlMemoryDef]      # ROM / NVM 分配
├── exported_parameters / exported_constants
└── source_dir
```

### 4.3 参数值（DCM / value.json）

```
DcmParameter
├── name: str         # 全限定名（如 rbp.vmc.apg.APG_Abort.VeloAbort）
├── kind: str         # FESTWERT | FESTWERTEBLOCK | KENNLINIE | KENNFELD
├── values: List[str] # 标定值
├── x_axis / y_axis   # 曲线/特性图的轴
└── size_x / size_y   # 维度
```

### 4.4 数据在内存中的索引结构

`PdlComponentBuilder` 从 `PdlPackage` 列表构建以下索引：

| 索引 | 键 | 值 | 用途 |
|------|----|----|------|
| `cluster_to_component` | Cluster 名 | Component 名 | Cluster → 所属组件 |
| `cluster_to_pdlclass` | Cluster 名 | PdlClass | Cluster → 参数定义 |
| `cluster_to_package` | Cluster 名 | Package 名 | Cluster → 包名（用于 FQN 匹配） |
| `cluster_to_memory` | Cluster 名 | ROM/NVM | 内存类型 |
| `all_quantizations` | 名称 | PdlQuantization | 全局量化表 |
| `all_enumerations` | 名称 | PdlEnumeration | 全局枚举表 |
| `all_custom_types` | 名称 | PdlCustomType | 全局自定义类型表 |
| `param_name_to_alias` | 参数名 | 别名 | 参数别名映射 |

## 5. 核心模块技术细节

### 5.1 ParGenXmlGenerator（generator.py）

XML 生成的核心类，负责构建完整的 ParGen ParameterSet XML。

**生成的 XML 结构：**

```xml
<ParameterSet>
  <Info>                          <!-- 版本号 + 时间戳 -->
  <Constants>                     <!-- PDL 导出常量 -->
  <DataTypes>                     <!-- 枚举、量化、自定义类型定义 -->
  <MajorVariantList>              <!-- 车型列表 -->
  <ComponentList>                 <!-- 组件 → Cluster → 参数 -->
    <Component>
      <ParameterCluster>
        <Define>                  <!-- 参数元数据 -->
        <VariantImplementationList>
          <VariantImplementation> <!-- 每个车型的参数值 -->
            <MajorVariant>
            <MinorVariant>
              <SubVariant>
                <Parameter value="..."/>
```

**关键算法：**

**值转换管线** (`_convert_values`):
```
DCM 原始字符串
    │
    ├── 文本类型 → 直接输出
    ├── 逻辑类型 → true/false → 1/0
    ├── 枚举类型 → 枚举名 → 整数值
    ├── 有量化公式 → storage = round_half_up(physical × factor)
    ├── 全局量化公式 → 同上
    └── 无量化 → float32 格式化 或 取整
```

**表达式求值** (`_eval_expression`):
- 支持 PDL 三元表达式 `(a > b) ? x : y`
- 运算符优先级解析
- 跨 Cluster 上下文引用（同一 minor/sub 下其他 Cluster 的值）

**VariantImplementation 去重** (`_compare_vi_trees`):
- 对同一 Cluster，比较多个车型的 minor/sub/value 树
- 结构和值完全一致的 VariantImplementation 合并为一个，MajorVariantList 包含多个车型名
- 对应 SHARCC 的 writeOptimizedVariantImplementation 逻辑

**DCM 索引与包匹配**:
- `dcm_by_cluster`: DCM 参数按 Cluster 分组
- `dcm_by_short_name`: 按短名索引（用于表达式引用）
- FQN 包前缀必须匹配 PDL `cluster_to_package`，避免跨包同名冲突

### 5.2 QuantizationConverter（quantization.py）

物理值到存储值的转换：

```
storage = round_half_up(Decimal(physical_str) × factor_value)
```

- 使用 Python `Decimal` 类型，避免浮点精度问题
- `ROUND_HALF_UP` 舍入策略（四舍五入），与 SHARCC Java 行为一致
- `get_resolution()` 返回 `1 / factor`，用于 XML DataTypes 定义

### 5.3 PdlComponentBuilder（component.py）

从 `List[PdlPackage]` 构建全局索引：

- 包名到组件名的映射遵循 Java 命名规则（`rbp.vmc.apg` → `APG`）
- 合并所有包中的量化、枚举、自定义类型、单位到全局表
- 构建 Cluster → Component 映射（一个 Component 包含多个 Cluster）
- 提取导出参数和常量

### 5.4 ConfigMasterLoader（configmaster_loader.py）

从 JSON 文件重建内存数据模型：

| JSON 文件 | 重建对象 | 说明 |
|-----------|---------|------|
| `variant.json` | `VariantDefinition` | 解析 `modules.vehicle.options` 中的层级结构 |
| `value/*.value.json` | `Dict[str, List[DcmParameter]]` | JSON 值 → DcmParameter，自动推断 kind |
| `model.json` | `List[PdlPackage]` + 全局类型表 | 重建 PDL 元数据，供 PdlComponentBuilder 使用 |

**value.json 值推断规则：**
- 标量数字 → `FESTWERT`
- 数字列表 → `FESTWERTEBLOCK`
- 含 `values` + `x_axis` 的对象 → `KENNLINIE` 或 `KENNFELD`

### 5.5 CombineBuilder（combine_builder.py）

多车型 XML 构建器，两种模式：

| 模式 | 输入 | 输出 | MajorVariantList |
|------|------|------|----------------|
| Combined | N 个车型 | 1 个 XML | 包含所有 N 个车型名 |
| Non-combined | N 个车型 | N 个 XML | 每个文件只含 1 个车型名 |

**Combined 模式实现：**
1. 第一个车型作为 primary，其余作为 `related_variants`
2. 每个车型独立解析 DCM 数据
3. 传入 `ParGenXmlGenerator`，由其内部的 `_compare_vi_trees` 处理去重合并

**ConfigMasterVariantResolver：**
- `variant.json` 的内存解析器
- `resolve_dcm_data(name)` — 收集该车型引用的所有 value.json 数据
- `collect_dcm_names_for_variant(mv)` — 收集该车型所有 DCM stem 名

### 5.6 Export（configmaster_export.py）

四阶段导出流程：

| 阶段 | 输入 | 输出 | 核心类/函数 |
|------|------|------|-----------|
| 1. parameter.json | PDL 文件 | `parameter/*.parameter.json` | `PdlComponentBuilder` + `generate_parameter_json` |
| 2. value.json | DCM 文件 | `value/*.value.json` | `DcmParser` + `generate_value_json` |
| 3. variant.json | pmavardef | `variant.json` | `PmaVardefParser` + `generate_variant_json` |
| 4. model.json | PDL 元数据 | `model.json` | `generate_model_json` |

### 5.7 AddVariant（add_variant.py）

车型复制流程：

1. 解析 `variant.json` 中 `modules.vehicle.options[SOURCE]`
2. 深拷贝整个 JSON 子树
3. 用正则 `(?<=_)OLD_SHORT(?=_|\b)` 重命名所有 value.json 文件引用
4. 将新条目写入 `modules.vehicle.options[NEW_NAME]`
5. （可选）复制对应的 value.json 物理文件并重命名

## 6. 模块依赖关系

```
main.py
 ├──▶ generator/configmaster_loader.py ──▶ generator/models.py
 │                                     ──▶ generator/component.py
 ├──▶ generator/component.py ──▶ generator/models.py
 ├──▶ generator/quantization.py ──▶ generator/models.py
 ├──▶ generator/generator.py ──▶ generator/models.py
 │                            ──▶ generator/component.py
 │                            ──▶ generator/quantization.py
 │                            ──▶ generator/variant.py
 ├──▶ combine/combine_builder.py ──▶ generator/generator.py
 │                                ──▶ generator/configmaster_loader.py
 │                                ──▶ generator/component.py
 │                                ──▶ generator/quantization.py
 │                                ──▶ generator/models.py
 ├──▶ export/configmaster_export.py ──▶ v7/（外部依赖）
 └──▶ add/add_variant.py（无内部依赖，仅 stdlib）
```

## 7. 与 SHARCC 的对应关系

| SHARCC (Java) | V10 (Python) | 说明 |
|---------------|-------------|------|
| `IVariantsFactory` / EMF | `models.py` dataclass | 数据模型 |
| `VariantsSwitch` (Visitor) | `VariantResolver` / `ConfigMasterVariantResolver` | 变体遍历 |
| `PdlComponentBuilder` | `component.py` `PdlComponentBuilder` | 同名，逻辑一致 |
| `QuantizationConverter` | `quantization.py` `QuantizationConverter` | Decimal HALF_UP |
| `ParGenXmlGenerator` | `generator.py` `ParGenXmlGenerator` | 同名，核心生成 |
| `BuildParGenXMLOperation` | `combine_builder.py` `CombineBuilder` | 合并/拆分逻辑 |
| `VariantImplementationCreator` | `generator.py` `_add_variant_impl_pdl` | VI 构建 + 去重 |
| `writeOptimizedVariantImplementation` | `generator.py` `_compare_vi_trees` | 相同 VI 合并 |
| pmavardef XML 解析 | `export/configmaster_export.py` → `variant.json` | 转为 JSON 中间格式 |

## 8. 技术栈

| 技术 | 用途 |
|------|------|
| Python 3.9+ | 运行时 |
| `dataclasses` | 数据模型定义 |
| `xml.etree.ElementTree` | XML 构建 |
| `xml.dom.minidom` | XML 格式化输出 |
| `decimal.Decimal` | 高精度量化计算 |
| `argparse` | CLI 参数解析 |
| `json` | JSON 读写 |
| `re` | 正则匹配（文件引用重命名、表达式解析） |
| `pathlib` | 文件路径处理 |

无第三方依赖，仅使用 Python 标准库。
