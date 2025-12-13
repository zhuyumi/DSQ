蓝图生成业务流程（面向非开发者的可读版）

业务目标与约束
1. 输出：生成可粘贴回游戏的蓝图字符串，包含布局、建筑、物流、配方与增产参数。  
2. 约束：占地尽量小、建筑不碰撞、满足产能需求（来自输入表）、尽量少用额外建筑（塔/喷涂/物流只在需要时生成）。  
3. 碰撞判定：代码以“占用矩形”而非物理网格计算。各生产类别的占地由 calculateBuildingArea() 给出（如熔炉/组装机 7×6，炼油 7×11，粒子对撞机 11×7/6，实验室 7×6；可堆叠实验室高度=buildingMap.lab.height=3）。传送带/分拣器占 1×1。  
4. 依赖：数据与生成逻辑全在前端脚本中，无后端；生成依赖 pako 压缩和全局映射表（itemMap/buildingMap/recipeMap）。  
5. 兼容：序列化格式遵循参考仓库 dspbp/dspbptk（相同 header + gzip + base64 + 校验和）。

输入与配置（来自 Scripts/data.js 的 UI）
1. 产线需求：用户在表格中勾选/填写每个子配方的输入、输出、建筑类型与数量（subRecipes），以及增产剂。  
2. 布局参数：x_y_ratio（长宽比，x/y）、magic_layer_cnt（层数）、magic_layer_height（层间高度）、magic_vertical_conveyor_belt_enabled（允许纵向带）、belt 堆叠层数、是否只用 MK3 带/分拣器、紧凑布局开关、特斯拉塔间隔。  
3. 蓝图元数据：标题、描述、最多 5 个图标。  
4. 校验：mapRecipeID() 将输入配方字符串映射到 recipeMap；无法识别会报错（cocoMessage）。

核心数据结构（Scripts/blueprint.js）
1. recipe：{ proliferator, subRecipes[] }，每个 subRecipe 含 input/output[{name, rate}]、building{name,num}、recipeID、acceleratorMode。  
2. config：上述布局/物流参数。  
3. 占用跟踪：blueprintSize{x,y}，occupiedArea[层][区域] 保存已占矩形；conveyorTempProductOffset 记录临时产物流的带路起点。  
4. buildingArray：按行记录已放置建筑及其 sorter 列表，用于后续布线。  
5. sorters：按物品名收集产出/消耗端的分拣器信息（owner、槽位、速率）。  
6. blueprintTemplate：最终导出的数据体（header、areas、buildings）。

生成流程（编号步骤）
1. 解析输入  
   1.1 读取表格，映射本地化名称→内部 id（itemNameList）。  
   1.2 组装 subRecipes、蓝图元数据、config。  
   1.3 校验 recipeMap（mapRecipeID + sortRecipeId），提示未知配方。  
2. 面积规划  
   2.1 计算总占地：遍历 subRecipes，按类别取 calculateBuildingArea(subRecipe).area × ceil(building.num)。  
   2.2 分层均摊：averageLayerArea=ceil(totalArea / magic_layer_cnt)。  
   2.3 推导每层长宽：y=ceil(sqrt(avg/x_y_ratio))，x=ceil(x_y_ratio*y)；occupiedArea 初始化为 [-1,-1 → x,y] 边界。  
3. 建筑放置（newProductionBuilding）  
   3.1 遍历 subRecipes；按层循环寻找可放位置（occupiedArea 间距 ≥ 占地的一半）。  
   3.2 计算建筑的中心/朝向（calculateBuildingArea）；写入 building.localOffset、yaw、itemId/modelIndex。  
   3.3 为每个输入/输出槽生成 sorter（slotIndex 递减，实验室特殊堆叠）。  
   3.4 按塔间隔（teslaTowerInterval）插入特斯拉塔，距离依据生产类别的默认偏移（calculateTeslaTowerOffset）。  
   3.5 记录 buildingArray（行分组），更新 occupiedArea 的 x2/y2 以避免碰撞。  
4. 物流布线  
   4.1 汇总 itemSummary：统计每物品的产出/消耗总速率（含增产剂加成）。  
   4.2 选择传送带：默认 MK1，如 onlyConveyorBeltMk3 或需求速率超过带速则提升到 MK3（transportSpeed 28/30）。  
   4.3 newConveyor：从产出端到消耗端排布 1×1 带节点，处理“临时产品”时可启用纵向带（magic_vertical_conveyor_belt_enabled）并用 conveyorTempProductOffset 跳层。  
   4.4 分拣器速率：按实际需求拆分到多条 sorter，超出 sorterMk3.sortingSpeed 时分段。  
   4.5 喷涂：若需要增产剂，记录 sprayCoaterOffsetList；generateConveyorBeltsForSprayCoater() 追加自喷回路，优先放置在 y 最小/x 最小的喷涂器附近。  
5. 序列化（toStr）  
   5.1 组装 header（layout=图标数量×10 或 51、时间戳、版本、描述）。  
   5.2 BufferWriter 按协议写 meta → areas → buildings（包括 parameters 经 parameterParsers 编码）。  
   5.3 pako gzip → base64 → 计算校验和（digest+hex）→ 拼接 BLUEPRINT: 字符串。

示例拆解（需求：长宽比 1:2，2 层，无带流蓝图，产出分拣器 2 个/分钟 + 传送带 1 个/分钟）
前提：本仓库当前逻辑仍会自动布带与分拣；“无带流”需求需要新增规则以跳过 4. 物流布线。以下按现有逻辑描述：  
1) 输入配置：x_y_ratio=0.5（x:y=1:2），magic_layer_cnt=2，magic_vertical_conveyor_belt_enabled 设为所需（若禁用则所有流平铺在同层），belt 堆叠层=0（无堆叠），onlySorterMk3/onlyConveyorBeltMk3 可根据速率开启。  
2) 产线定义：在表格中添加两个 subRecipe：  
   - 分拣器：选择对应配方（参见 recipeMap 中 sorter 配方），填目标产出速率 2/min，选择生产建筑（默认组装机）并填写数量（依配方时间和 productionSpeed 预估）。  
   - 传送带：选择带配方，目标产出 1/min，同理填写建筑数量。  
3) 计算占地：calculateBuildingArea 以“组装机”类别占地 7×6 估算；总面积 ≈ (7×6×分拣器台数 + 7×6×传送带台数)，再按 2 层均摊求每层 x=ceil(0.5*y)、y=ceil(sqrt(avg/0.5))。occupiedArea 初始化为 [-1,-1 → x,y]。  
4) 放置建筑：逐 subRecipe 在层内自左向右补齐占用矩形，若一行剩余 x 不足则换行（y2 递增），层满则尝试下一层；按 teslaTowerInterval 插入塔。  
5) 布线：为每个配方的输入物品生成分拣器和带路；如果“无带流”是强需求，应在 newConveyor 前短路掉物流生成，只保留建筑与配方，这目前未实现。  
6) 导出：toStr 输出蓝图字符串；layout=20（两图标）或据实际图标数确定。

参考仓库要点
1. dspbp、dspbptk：提供相同蓝图二进制协议的解码/编辑，佐证当前序列化格式（meta+area+buildings gzip+base64+校验）。  
2. dsp_blueprint_editor、edit-dspblue-print：UI 侧的配方编辑与可视化，可作为后续重构时的数据分层与解码参考，但不涉及本仓库的自动布线逻辑。

现存不清晰与待补充
1. 理解“无带流”：当前仓库的布线仍然用传送带和分拣器连接产出/消耗，只是有一套“临时产物”与纵向带的处理（newConveyor_isTempProduct + magic_vertical_conveyor_belt_enabled），在 
  需要跨层或暂存时用短带/竖带来减少主干带长度，但并没有完全取消传送带。 
2. buildingMap 部分建筑未声明 size；布局实际依赖 calculateBuildingArea 的硬编码占地，若要严谨碰撞校验需补全尺寸表。  
3. 配方产率/配方本身依赖 data.js/recipeMap，需在文档中补一份可读表以方便人工估算建筑数量。  
4. 缺少权威回归样例（蓝图串 + 解析结果）来验证重构后的行为一致性。
