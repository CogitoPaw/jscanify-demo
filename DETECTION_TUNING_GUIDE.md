# 文档边框识别调参手册（jscanify-demo）

本文件用于沉淀当前会话的识别策略与调参依据，避免后续改动时误伤已修复场景。

## 1. 当前目标与约束

- 统一上传图片与摄像头拍照的识别内核，减少分叉维护成本。
- 上传图片体验：上传后自动识别，可手动拖拽微调。
- 摄像头体验：预览稳定、拍照后自动识别、对“贴边大框误检”更严格。
- 保持手动调整机制作为最终兜底。

## 2. 核心流程（当前实现）

### 2.1 入口统一

- 统一入口函数：`enterEditorWithSource(source, opts)`
- 上传：`loadFile()` -> `enterEditorWithSource(..., sourceKind: 'upload')`
- 拍照：`captureCam()` -> `enterEditorWithSource(..., sourceKind: 'camera-capture')`

### 2.2 自动识别主链

- `autoDetect(opts)`
  - 调用 `detectCornersSmart(source, opts)`
  - 摄像头来源会启用 `cameraStrict`
  - 若摄像头结果可疑，会触发二次重检（`preferInner`）
  - 若仍不理想，会和拍照前稳定框先验对比（`camera-hint` 兜底）

### 2.3 检测内核

- `detectCornersSmart(source, opts)`
  - 双路候选：`detectByJscanify` + `detectByCvFallback`
  - 候选通过 `isReasonableQuad` 后，用 `scoreQuad` 竞争
  - 支持 `cameraStrict / preferInner / debug`

- `detectByCvFallback(source, opts)`
  - 多通道：多组 Canny + adaptive threshold
  - 每个通道双搜索：
    - `suppressFrameBorder=true`（抑制画面边框干扰）
    - `suppressFrameBorder=false`（兜底）
  - 支持摄像头严格模式下的候选早筛

## 3. 摄像头专项策略（关键）

### 3.1 实时预览稳态

- `camTick()` 每 `CAM_DETECT_INTERVAL` 帧检测一次。
- 稳定器参数：
  - `CAM_STABLE_DRIFT`
  - `CAM_REPLACE_SCORE_DELTA`
  - `CAM_CLEAR_MISS_COUNT`
- 逻辑：新框漂移过大时，除非分数明显更高，否则不替换上一稳定框。

### 3.2 拍照后专项兜底

- 拍照时保存预览稳定框：`pendingCameraHintQuad = camLastQuad`
- 进入编辑区可先显示该先验框（`initialCorners`）
- 若拍照自动识别结果可疑：
  - 再跑一次 `preferInner=true` 的重检
  - 与先验框比较后，必要时使用 `camera-hint` 结果

### 3.3 可疑结果定义

- 函数：`isSuspiciousCameraQuad(c, w, h)`
- 核心信号：
  - 边界命中数高（`borderHits`）
  - 四边形面积占比/包围盒占比过大（常见为误选桌面或画面边缘）

## 4. 评分与过滤（关键函数）

- `scoreQuadBreakdown(...)`
  - 主要分项：`areaScore`, `areaSweetSpot`, `angle`, `pairBalance`, `centerScore`
  - 惩罚项：`borderPenalty`, `cameraCoveragePenalty`（摄像头专属加重）
  - 输出 `total` 供候选竞争

- `isReasonableQuad(...)`
  - 基础几何约束：面积、最短边、角度、对边一致性
  - 摄像头严格模式额外约束：
    - 贴边数量
    - 面积上限
    - 包围盒占比上限
    - `preferInner` 下进一步收紧

## 5. 调试与观测

- 打开调试：URL 加 `?debug=1`
- 调试输出位置：`#debug-info`
- 输出内容：
  - 最终来源与总分
  - Top3 候选分数分解（面积、角度、中心、边界惩罚、摄像头惩罚等）

建议：每次改阈值前先开 `debug=1`，确认是“找不到候选”还是“候选排序错误”。

## 6. 已验证问题与对应修复

- 上传后需手动点按钮：已改为自动识别。
- 上传与拍照逻辑分叉：已统一到同一入口与同一识别链。
- 摄像头误选贴边大框：已加 cameraStrict、preferInner、候选早筛、稳定器、拍照先验兜底。
- 手动可调性：保留。

## 7. 后续调参建议（不动主干时）

优先按以下顺序微调，避免破坏已修复路径：

1. `cameraCoveragePenalty`（先调摄像头误检）
2. `isSuspiciousCameraQuad` 判定阈值（控制二次重检触发率）
3. `CAM_*` 稳定参数（体验层）
4. `isReasonableQuad` cameraStrict 分支（最后再动）

## 8. 回归测试清单（每次改动必跑）

- 上传图片：高对比、低对比、强透视、贴边整页、小文档。
- 摄像头拍照：桌面纹理强、右侧背景干扰（手/衣物/屏幕边）、轻微抖动、倾斜角度大。
- 观察项：
  - 初始框是否贴合文档主体
  - 是否出现大范围背景误框
  - 是否出现明显跳框
  - 手动微调后预览是否稳定

## 9. 参数变更日志模板（建议每次调参都记录）

复制下面模板，追加到本文件末尾（或单独建 `CHANGELOG_DETECTION.md`）：

```markdown
### [YYYY-MM-DD HH:mm] 调参记录

- **目标问题**：
  - 例如：摄像头拍照后右侧贴边大框误检

- **改动范围**：
  - 文件：`index.html`
  - 函数：`detectCornersSmart` / `scoreQuadBreakdown` / `isReasonableQuad` / `camTick`

- **参数/逻辑变更**：
  - `CAM_DETECT_INTERVAL`: 3 -> 2
  - `cameraCoveragePenalty`: +0.45（条件：borderHits >= 1 && areaPct > 0.55）
  - 新增/删除规则：...

- **样本结果（至少 5 张）**：
  - case A（上传-高对比）：✅ / ❌，说明
  - case B（上传-低对比）：✅ / ❌，说明
  - case C（摄像头-桌面纹理强）：✅ / ❌，说明
  - case D（摄像头-大倾角）：✅ / ❌，说明
  - case E（摄像头-轻微抖动）：✅ / ❌，说明

- **副作用检查**：
  - 是否出现新误检：有/无
  - 是否出现跳框加重：有/无
  - 性能影响（主观）：低/中/高

- **结论**：
  - 保留 / 回滚 / 继续微调（下一步参数）
```

---

若后续引入新策略，请同步更新本文件中的：

- “核心流程”
- “摄像头专项策略”
- “已验证问题与对应修复”
- “回归测试清单”

