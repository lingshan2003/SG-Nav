# SG-Nav 深度代码研究报告（scenegraph.py 数据流）

## 研究范围与约束

- 仓库：`lingshan2003/SG-Nav`
- 重点：`/home/runner/work/SG-Nav/SG-Nav/scenegraph.py` 的完整数据流
- 约束：**仅使用预先拍摄的 RGB 图片（无深度、无实时模拟器）**
- 说明：以下结论均基于代码事实；若仓库内未发现对应概念，将明确标注“未发现”。

---

## 1) 分层图结构定义与构建

### 1.1 节点定义位置、字段、关系约束

定义文件：`/home/runner/work/SG-Nav/SG-Nav/scenegraph.py`

#### RoomNode（第 35–41 行附近）
- 字段：
  - `caption`
  - `exploration_level`
  - `nodes`（`set`，存放 `ObjectNode`）
  - `group_nodes`（`list`，存放 `GroupNode`）

#### GroupNode（第 43–71 行附近）
- 字段：
  - `caption`
  - `exploration_level`
  - `corr_score`
  - `center`
  - `center_node`
  - `nodes`（`ObjectNode` 列表）
  - `edges`（`Edge` 集合）
- 关键方法：
  - `get_graph()`：计算组中心、中心节点、聚合边，并生成文本描述
  - `graph_to_text()`：将节点/边转成字符串

#### ObjectNode（第 73–113 行附近）
- 字段：
  - `is_new_node`
  - `is_goal_node`
  - `caption`
  - `object`
  - `reason`
  - `center`
  - `room_node`
  - `exploration_level`
  - `distance`
  - `score`
  - `edges`
- 关键方法：
  - `set_caption()`：改名时清空旧边并重置状态
  - `set_object()`：双向绑定 object 字典（`object['node']=self`）

#### Edge（第 115–132 行附近）
- 字段：
  - `node1`, `node2`
  - `relation`
- 约束：
  - 构造时自动注册到两个节点的 `edges`
  - `delete()` 会从两端节点删除该边

### 1.2 分层关系如何建立（room->group->object）

- `room -> object`：`update_node()`（`scenegraph.py` 第 609–645 行附近）
  - 先由 object 的 3D 点云中心投影得到 map 坐标 `(x,y)`
  - 再通过 `room_map[0,:,y,x]` 的非零通道索引确定 `room_label`
  - 把 `ObjectNode` 加入对应 `RoomNode.nodes`
- `room -> group` + `group -> object`：`update_group()`（第 706–721 行附近）
  - 对同一房间中的 `ObjectNode.center` 做 `DBSCAN(eps=10,min_samples=1)`
  - 每个聚类生成一个 `GroupNode`，包含该簇对象节点
  - `group_node.get_graph()` 聚合边并生成组描述文本

### 1.3 是否有规则/阈值/先验映射/模板

- 有：
  - DBSCAN 聚类参数：`eps=10, min_samples=1`（`scenegraph.py` 第 713 行）
  - 目标最少检测阈值：`threshold_list`（`scenegraph.py` 第 172 行）
  - 对象过滤阈值：`filter_objects()`（`utils/utils_scenegraph/utils.py` 第 12–19 行）
  - 房间类别固定列表（9 类）：`rooms`（`utils/utils_glip.py` 第 39 行）
- 未发现：
  - 显式的“厨房用品”中间本体层（RoomNode 与 ObjectNode 之间的语义模板层）
  - 预定义“厨房->厨房用品->家具”硬编码模板

### 1.4 “厨房/厨房用品/具体家具”语义层级实现路径

- 代码中实际是：
  - 房间层：`RoomNode`（如 `kitchen`）
  - 对象层：`ObjectNode.caption`（如 `sink`, `table`，来自 GroundingDINO+SAM）
  - 组层：`GroupNode`（几何聚类产物，不是显式“用品”语义类）
- 即：`kitchen` 与具体对象存在；“厨房用品”作为独立语义节点 **未发现**。

---

## 2) scenegraph.py 主流程拆解（从单张 RGB 出发）

主入口：`update_scenegraph()`（`scenegraph.py` 第 750–758 行）

1. `segment2d()`
2. `mapping3d()`
3. `get_caption()`
4. `update_node()`
5. `update_edge()`

### 2.1 单张 RGB 起点调用链与中间结构

- `set_observations()`（第 252–257 行）写入：
  - `self.image_rgb = observations['rgb']`
  - `self.image_depth = observations['depth']`
  - `self.pose_matrix = get_pose_matrix()`（依赖 gps/compass）

#### A. `segment2d()`（第 521–551 行）
- 输入：`image_rgb`
- 调用：`get_sam_segmentation_dense()`（第 321–399 行）
  - GroundingDINO 产生 `boxes_filt + caption`
  - SAM 产生 `mask + conf`
- 输出：`segment2d_results.append({...})`，关键字段：
  - `xyxy` `(N,4)`
  - `confidence` `(N,)`
  - `mask` `(N,H,W)`
  - `caption` `list[str]`
  - `image_rgb`

#### B. `mapping3d()`（第 553–597 行）
- 输入：`image_depth`, `camera_matrix`, `pose_matrix`, `segment2d_results[-1]`
- 调用：`gobs_to_detection_list()`（`utils/utils_scenegraph/utils.py` 第 200–296 行）
  - 内部 `create_object_pcd()` 由深度 + mask 反投影点云
  - 可选 `trans_pose` 把点云变换到全局
- 然后：
  - `compute_spatial_similarities()`（`utils/utils_scenegraph/mapping.py` 第 7–34 行）
  - `merge_detections_to_objects()`（同文件第 37–57 行）
  - `filter_objects()`（`utils/utils_scenegraph/utils.py` 第 12–19 行）
- 输出：`self.objects_post`（MapObjectList）

#### C. `get_caption()`（第 599–607 行）
- 从 object 历史 `image_idx/mask_idx` 回读 `segment2d_results[*]['caption']`
- 多帧 caption 取众数写入 `object['captions']`

#### D. `update_node()`（第 609–645 行）
- 新建/更新 `ObjectNode`
- 从 `object['pcd'].points` 求中心，映射到 map 坐标
- 用 `room_map` 赋房间归属（RoomNode）

#### E. `update_edge()`（第 647–704 行）
- 对新旧节点两两建边
- 优先 VLM 直接回答空间关系（`get_vlm_response()`）
- 否则 LLM 提关系提案，再 `discriminate_relation()` 过滤

### 2.2 哪些步骤必须依赖外部条件

- 依赖模拟器状态（强）：
  - `set_observations()` 需要 `gps/compass/depth/rgb`
  - `get_pose_matrix()` 需要 gps/compass
  - `mapping3d()` 需要 depth + pose + camera_matrix
  - `update_node()` 的 room 分配依赖 `room_map`
  - `discriminate_relation()` 几何分支依赖 `fbe_free_map`
- 依赖检测器输出（强）：
  - `segment2d()` 依赖 GroundingDINO + SAM
- 依赖历史轨迹（中）：
  - `get_caption()` 聚合多帧 caption
  - `get_joint_image()` 依赖 object 的历史 `image_idx`

### 2.3 离线最小化可运行 vs 直接不可执行

#### 可离线复用（仅 RGB）
- `segment2d()`
- `get_sam_segmentation_dense()`（groundedsam 分支）
- `get_joint_image()`（如果有历史帧缓存）
- `get_llm_response()/get_vlm_response()`
- `graph_to_text()`（输入节点边即可）

#### 缺失输入后不可直接执行
- `mapping3d()`（无深度即无法建点云）
- `update_node()`（依赖 `pcd` 和 `room_map`）
- `update_group()` 当前实现依赖 `ObjectNode.center`（来自 3D）
- `discriminate_relation()` 的 free-map 几何校验路径

---

## 3) 依赖模块与数据源

### 3.1 上游模块与接口

1. Grounded-SAM
- 文件：`utils/utils_scenegraph/grounded_sam_demo.py`
- 接口：
  - `load_model(config, checkpoint, ..., device)`
  - `get_grounding_output(model, image, caption, box_threshold, text_threshold, ...)`
- I/O：
  - 输入图像 tensor `(3,h,w)`
  - 输出 `boxes_filt` `(N,4)`、`pred_phrases` `list[str]`

2. SceneGraph 2D 分割缓存
- 文件：`scenegraph.py` `segment2d()`
- `segment2d_results` 每帧结构：
  - `xyxy`: `(N,4)`
  - `confidence`: `(N,)`
  - `class_id`: `(N,)`
  - `mask`: `(N,H,W)`
  - `caption`: `list[str]`
  - `image_rgb`: `(H,W,3)`

3. 3D 映射与对象融合
- 文件：
  - `utils/utils_scenegraph/utils.py`：`gobs_to_detection_list()`, `create_object_pcd()`
  - `utils/utils_scenegraph/mapping.py`：`compute_spatial_similarities()`, `merge_detections_to_objects()`
- 关键输出对象字段（字典）：
  - `image_idx`, `mask_idx`, `class_name`, `class_id`
  - `mask`, `xyxy`, `conf`
  - `pcd`（open3d point cloud）
  - `bbox`（open3d bounding box）
  - `num_detections`

4. 地图与坐标
- 文件：
  - `utils/utils_fmm/mapping.py`（`Semantic_Mapping`）
  - `utils/utils_fmm/depth_utils.py`（`get_camera_matrix`）
- 依赖：
  - 深度图 + 位姿（pose）+ 相机内参
- 输出：
  - `full_map`, `fbe_free_map`, `room_map` 等栅格图

5. 房间/类别词表与先验
- 文件：`utils/utils_glip.py`
- 关键常量：
  - `rooms`（9 类房间）
  - `rooms_captions`
  - `categories_21` / `object_captions`
  - `projection`（任务类别映射）

### 3.2 “仅 RGB 离线输入”模块可复用清单

#### 可复用
- GroundingDINO + SAM 推理链（`segment2d`）
- 文字层推理（LLM/VLM 调用）
- 图文本化（`graph_to_text`）

#### 不可直接复用（缺深度/位姿/地图）
- `mapping3d`
- `update_node`（当前版本）
- `update_group`（当前依赖 3D 中心）
- 依赖 `room_map/fbe_free_map/full_map` 的逻辑

---

## 4) 针对用户约束的客观方案

### 4.1 RoomNode 先验（FloorPlan->Room）可替代环节

用户已知场景房间（如 FloorPlan1~30=厨房）时，可替代：

1. 原流程中 `update_node()` 的 room_map 判别逻辑（`scenegraph.py` 第 632–643 行）
   - 可直接将对象绑定到先验 `RoomNode("kitchen")`
2. 原流程 `insert_goal()` 里基于候选房间列表的 room 预测提示
   - 若房间先验确定，可跳过 room 推断

### 4.2 不引入实时模拟器前提下的可执行最小流程

#### 最小流程（客观边界版）
1. RGB 输入 -> `segment2d()` 得到 object 候选（box/mask/caption）
2. 用检测结果直接构建轻量 `ObjectNode`（中心用 2D bbox center，非 3D）
3. 将所有 ObjectNode 直接挂到先验 RoomNode（如 kitchen）
4. 以 2D 中心进行聚类生成 GroupNode（替代现有 3D center DBSCAN）
5. 用 VLM/LLM 生成候选关系（可不走 free-map 几何过滤）
6. 输出 room->group->object 层级结构

### 4.3 各环节“已有能力可直接调用” vs “需轻量改造”

#### 可直接调用
- `segment2d()` / Grounded-SAM
- `get_llm_response()` / `get_vlm_response()`
- `GroupNode.graph_to_text()`（前提是已构建节点与边）

#### 需轻量改造/替换
- `update_node()`：当前强依赖 3D `pcd` + `room_map`
- `update_group()`：当前中心来自 3D
- `mapping3d()`：在纯 RGB 下需整体绕过
- `discriminate_relation()`：free_map 分支需绕过或替代

### 4.4 受限点与可行边界（不强行补深度）

- 受限点：
  - 无法恢复真实 3D 坐标与跨视角精确融合
  - 无法使用 map-based room attribution 与 free-space 几何验证
- 可行边界：
  - 可获得稳定的 2D object 候选与语义标签
  - 可构造“弱几何”的层级图（基于 2D 近邻 + 语言关系）
  - 可在有 RoomNode 先验时完成 room->group->object 组织，但语义/几何一致性弱于原始 RGBD 在线流程

---

## 5) 交付给 AI Agent 的执行计划（分阶段+验收标准）

> 以下为面向后续编码 agent 的实施计划，聚焦“离线 RGB + Room 先验”。

### Phase 0: 代码定位与最小调用验证

- 要读文件：
  - `/home/runner/work/SG-Nav/SG-Nav/scenegraph.py`
  - `/home/runner/work/SG-Nav/SG-Nav/utils/utils_scenegraph/grounded_sam_demo.py`
  - `/home/runner/work/SG-Nav/SG-Nav/utils/utils_glip.py`
- 关注函数：
  - `segment2d`, `get_sam_segmentation_dense`, `update_scenegraph`
- 输入/输出契约：
  - 输入：单张 `np.ndarray(H,W,3)` RGB
  - 输出：`segment2d_results[-1]` 含 `xyxy/mask/caption`
- 可观测日志：
  - 检测数量 N、caption 列表
- 通过条件：
  - 不依赖 simulator loop 成功得到非空候选

### Phase 1: 离线 RGB 输入适配（不依赖 simulator loop）

- 要改文件：
  - `/home/runner/work/SG-Nav/SG-Nav/scenegraph.py`
- 目标函数：
  - 新增或改造离线入口（例如 `update_scenegraph_offline(rgb)`）
  - 避免进入 `mapping3d` 强依赖链
- 输入/输出契约：
  - 输入：RGB 图像（单张/批量）
  - 输出：每帧候选对象（`xyxy/mask/caption/conf`）
- 可观测日志：
  - 每帧候选计数、平均置信度
- 通过条件：
  - 全流程无 depth/gps/compass 字段也可运行

### Phase 2: RoomNode 先验注入（FloorPlan->Room 映射）

- 要读/改文件：
  - `/home/runner/work/SG-Nav/SG-Nav/scenegraph.py`
  - （可选）新增先验配置读取位置
- 目标函数：
  - `init_room_nodes`, `update_node`（或离线替代节点构建函数）
- 输入/输出契约：
  - 输入：`scene_id -> room_name`（例如 `FloorPlan1 -> kitchen`）
  - 输出：每个 ObjectNode 挂接到先验 RoomNode
- 可观测日志：
  - 节点归属房间统计
- 通过条件：
  - 节点房间归属与先验映射一致率 100%

### Phase 3: Group/Object 生成与层级图导出

- 要改文件：
  - `/home/runner/work/SG-Nav/SG-Nav/scenegraph.py`
  - （可选）新增导出工具模块
- 目标函数：
  - `update_group`（支持 2D 中心）
  - 新增图导出函数（JSON/Markdown）
- 输入/输出契约：
  - 输入：ObjectNode 列表（2D center + caption）
  - 输出：`RoomNode -> GroupNode -> ObjectNode` 分层结构
- 可观测日志：
  - 每层节点数量、每组对象列表
- 通过条件：
  - 成功导出结构化结果，字段完整（room/group/object/relation）

### Phase 4: 质量评估与失败案例归因

- 要读文件：
  - `/home/runner/work/SG-Nav/SG-Nav/scenegraph.py`
  - `/home/runner/work/SG-Nav/SG-Nav/utils/utils_scenegraph/utils.py`
- 评估内容：
  - 候选召回（检测数量）
  - 分组稳定性（同类对象是否聚到同组）
  - 关系可用率（edge 有效占比）
- 可观测日志：
  - 每图统计报表 + 失败样例索引
- 通过条件：
  - 输出可复现实验日志与失败归因清单
  - 明确标注失败由“无深度/无位姿/无地图”导致的占比

---

## 关键代码引用清单（便于复核）

- `scenegraph.py`
  - `RoomNode/GroupNode/ObjectNode/Edge`: 35–132
  - `SceneGraph.__init__`: 136–207
  - `set_observations`: 252–257
  - `get_pose_matrix`: 509–519
  - `segment2d`: 521–551
  - `mapping3d`: 553–597
  - `get_caption`: 599–607
  - `update_node`: 609–645
  - `update_edge`: 647–704
  - `update_group`: 706–721
  - `insert_goal`: 723–748
  - `update_scenegraph`: 750–758
  - `discriminate_relation`: 847–873
- `utils/utils_scenegraph/utils.py`
  - `filter_objects`: 12–19
  - `create_object_pcd`: 139–197
  - `gobs_to_detection_list`: 200–296
- `utils/utils_scenegraph/mapping.py`
  - `compute_spatial_similarities`: 7–34
  - `merge_detections_to_objects`: 37–57
- `utils/utils_glip.py`
  - `categories_21`: 27–38
  - `rooms / rooms_captions`: 39–40
- `SG_Nav.py`
  - `act()` 中 scenegraph 调用：370–379
  - `update_room_map`: 624–634
- `utils/utils_fmm/mapping.py`
  - `Semantic_Mapping.forward`: 64–187

