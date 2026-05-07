# 学生听课状态监测系统 (Student Engagement Monitor)

这是一个基于 Vue 3 + Vite + Face-API.js 的前端项目，用于通过摄像头实时监测学生的听课状态（表情）和坐姿。

## 🛠 技术栈
- **Frontend**: Vue 3, Vite, Ant Design Vue, Face-API.js
- **Container**: Docker, Nginx

## ✨ 功能特性
- **实时预览**: 显示摄像头画面及人脸检测结果。
- **状态分析**: 识别表情（开心、困惑、专注等）及简单的坐姿判断。
- **时间轴**: 记录并展示随时间变化的学生状态。
- **配置**: 可调整检测记录的时间间隔。
- **响应式设计**: 兼容 PC 和移动端访问。

## 🚀 启动指南 (How to Run)

### NPM 启动（开发模式）

如果您想在本地开发或调试代码，可以使用 npm 启动：

1. 安装依赖：
   ```bash
   npm install
   ```
2. 启动开发服务器：
   ```bash
   npm run dev
   ```
3. 访问终端显示的本地地址（通常为 http://localhost:3000）

## ⚠️ 注意事项
- 请允许浏览器访问摄像头权限。
- 如遇到模型加载失败，请检查网络连接。
