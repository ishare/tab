# Tab.archive

> 一个私人的吉他六弦谱曲库,带中文 / 拼音搜索的现代化阅读体验。

## 特性

- 🎸 收录大量曲目的吉他六弦谱(图片格式)
- 🔍 支持中文 / 拼音首字母 / 全拼搜索
- 🌓 深色 / 浅色双主题,默认复古录音棚配色
- 📱 响应式布局,手机 / 桌面 / 平板自适应
- ⛶ 双击曲谱区或点击 FULLSCREEN 进入沉浸阅读模式
- ⚡ 基于 `virtua` 的虚拟列表,上千首曲目流畅滚动

## 技术栈

- [Vue 3](https://vuejs.org/) + `<script setup>` SFC
- [Vite](https://vite.dev/)
- [virtua](https://github.com/inokawa/virtua) — 虚拟滚动
- [tiny-pinyin](https://github.com/idealoint/tiny-pinyin) — 拼音索引
- 纯 CSS(无 UI 框架),Google Fonts: Fraunces / Inter Tight / JetBrains Mono / Noto Serif SC

## 本地运行

```bash
bun install      # 或 npm install / pnpm install
bun run dev      # 启动开发服务器
bun run build    # 生产构建
bun run preview  # 预览构建产物
```

## 曲谱数据

曲谱图片位于 `public/data/pics/{曲目 ID}/0..n-1.webp`,索引文件为 `public/data/dict.json`,
字段:`i` 曲目 ID,`t` 标题,`c` 图片张数。

所有曲谱素材均来自互联网整理分享,仅供学习交流使用。
