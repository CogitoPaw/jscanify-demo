# AGENTS 指南（识别逻辑改动前必读）

本项目的文档边框识别调参规则见：`DETECTION_TUNING_GUIDE.md`。

## 1) 适用范围

- 当你（AI 或人）要修改以下任一内容时，必须先阅读调参手册：
  - `detectCornersSmart`
  - `detectByCvFallback`
  - `scoreQuadBreakdown` / `isReasonableQuad`
  - `camTick` / `captureCam` / `autoDetect`
  - 任何 `cameraStrict` / `preferInner` / 稳定器相关参数

## 2) 强制约束

- 默认不破坏上传通道；摄像头专项问题优先在 `cameraStrict` 分支修复。
- 保留手动拖点微调机制，不得移除。
- 改阈值前先用 `?debug=1` 观察候选与分数分项。
- 每次改动后至少完成一轮“上传 + 摄像头”回归。

## 3) 修改后必须更新文档

- 至少同步更新 `DETECTION_TUNING_GUIDE.md` 中以下任一项：
  - 核心流程
  - 摄像头专项策略
  - 已验证问题与对应修复
  - 回归测试清单
- 建议使用手册里的“参数变更日志模板”记录本次调参。

## 4) 命名说明

- `DETECTION_TUNING_GUIDE.md`：以“人可读”为主，也可供 AI 读取参考。
- `AGENTS.md`：以“AI 执行约束”为主，告诉后续代理如何安全修改识别逻辑。
