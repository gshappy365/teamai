---
title: "ChatCut 竖版解说短视频制作流水线（pstack 系列）"
author: gshappy365
date: 2026-10-06
tags: [workflow, tool-usage, best-practice, video]
---

## Context

把一门课程的 HTML 讲义（pstack 系列）逐课制作成竖版解说短视频（1080×1440，小红书风格），并且每一条视频必须与上一条保持**完全一致的生成过程和模板**。同一套流程已连续跑通 5 条视频（含沉淀漏斗、Skill 调用、开工四件套、设计与决策等课节）。

## Solution

在 ChatCut Desktop（本机 MCP server，通过 `chatcut-mcp-client.js` 调用）上执行的固定流水线：

1. **新建项目**：1080×1440 @ 30fps
2. **应用模板**：`manage_design_style` apply_preset（Blue Orange Marker 预设：蓝 #79AFD4 / 橙 #FF8A42 / 米底 #FFF1D7 + 网格线 + Chill Huo Gothic 字体）
3. **旁白**：`submit_voice`（provider=doubao、voiceId=dayi、speedRatio=1.05），每课 5 段，**实测每段真实帧数**后铺位
4. **MG 动画**：`create_motion_graphic_from_code` 直接写可编辑代码（单顶层组件、纯内联样式、无 import），5 段画面各自呼应旁白内容
5. **轨道与铺位**：建 视觉(video)/旁白(audio anchor)/音乐(audio follower) 三条轨；`edit_item` 先 validateOnly 校验再正式提交
6. **BGM**：`submit_music` instrumental，volume=0.22，音乐轨 ducking -14dB（follower）
7. **字幕**：`edit_captions` enable deyi-card 预设；隐藏音乐轨被误转写的乱码字幕卡
8. **导出**：local_export 1080p / h264 / 30fps
9. **验证**：ffprobe 规格 + 5 点抽帧（`-ss <秒>` 方式）Read 检查画面与字幕 + volumedetect 检查音轨非静音

## Lessons

- **铺位帧数必须比实测少 1**：`durationFrames` 等于旁白实测帧数会报 `source range exceeds asset`，各段减 1 帧即通过（Audio 资产实际帧数比 inspect 结果少 1 帧）
- **MG 代码硬性约束**：不允许 import、只能一个顶层组件函数（形如 `function SceneX({ item })`）、顶层禁止变量（常量/数组须放函数内）、根元素必须 `<div style={{...}}>`（不能用 AbsoluteFill）、动画时间用秒做单位
- **JSX 陷阱**：文本里写 `<` 会被解析成标签（如 `P99 < 200ms` 报 `Identifier directly after number`），需写成 `{'<'}` 或 `&lt;`
- **BGM 生成时长不可控**：同一条 prompt 生成的时长在 147s–213s 之间波动，强调"更长"不保证有效；取最长的版本铺入，尾部短暂无音乐可接受，不值得反复重试烧 credits
- **音乐轨会被字幕系统误转写**成乱码卡（如"釣戓煉釣戓煉"），必须用 `edit_captions` action=track + trackId + visible=false 隐藏
- **Windows 抽帧坑**：`ffmpeg select=eq(n,...)` 会重复输出同一帧，改用 `-ss <秒> -i <mp4> -frames:v 1` 逐点抽帧
- **能力因项目而异**：部分项目 `edit_captions` 支持 read/list，本项目返回 `Unsupported Memory caption action`；用 read_project 确认字幕存在 + 直接隐藏音乐轨字幕（幂等）替代
- **铺位格式**：motion-graphic 带 `fit:"cover"`，audio 不带；trackId 可用 10 位前缀

## Related Skills

- chatcut-video-editor
- doubao-creative-video
