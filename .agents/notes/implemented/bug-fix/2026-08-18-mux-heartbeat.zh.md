# Agent Note: 心跳清理由僵尸浏览器下行连接，让 pending 提问得以恢复

Status: implemented

[English](2026-08-18-mux-heartbeat.md) | 中文

## Problem（问题）

当没有浏览器 UI 实际在接收帧时，pending 的 `ask_user_question` 会让会话永久挂起。宿主在提问时把 `question/requested` 推给每一个在线的 mux 队列，并在每次 mux-open 时重放仍未决的提问，但两条路径都要求有已连接的客户端。静默死亡的浏览器 WebSocket——机器睡眠、网络切换、后台标签节流——任何一侧都无法发现：服务端没有心跳，浏览器只能在系统级 TCP 超时（可能长达几十分钟）后才察觉。在该窗口内 UI 仍显示"已连接"，没有新的 mux 连接代次打开，重放不会运行，提问卡片永远不渲染，turn 无限期停滞。

## Decision（决策）

`dsh-client-connection` 在其两条下行 WebSocket（`/api/events.mux`、`/api/events.host`）上运行服务端心跳：每个 `heartbeatIntervalMs`（新增 `ConnectionConfig` 配置项，默认 30 秒）服务端 ping 每个已接受的 socket，连续两次未收到 pong 的连接将被终止。浏览器在协议层自动回 pong（RFC 6455），因此客户端代码无需改动。`terminate()` 会在浏览器触发 `close`，既有 `ConnectionController` 重连循环按退避策略重新打开两条流，既有 mux-open 重放把仍未决的提问与审批重新送达——提问卡片出现，会话继续。只漏掉一次 ping 的 socket（睡眠唤醒、瞬时抖动）不会被误杀：两拍宽容是算法常量，节奏则由部署配置决定。

## Alternatives considered（备选方案）

**客户端活性看门狗。** 浏览器无法观察协议层 pong，客户端需要应用层 ping 帧（新增 MuxFrame 变体）加静默计时器。它能修复同一类死连接，但新增了服务端心跳可以避免的线上表面积；既有重连循环已把服务端的 terminate 转化为带重放的重连。

**在无 mux 连接时让 `ask_user_question` 直接失败。** 这只覆盖干净关闭的 socket；僵尸 socket 仍占据其队列，提问照样挂起。它可以在未来配合超时使用，但无法把提问恢复给在场的用户。
