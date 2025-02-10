# Camera Resolution Scanner

一个基于 Web 的摄像头分辨率和帧率扫描工具，帮助开发者和用户快速检测摄像头支持的分辨率和帧率组合。

## 功能特点

- 🎥 自动检测系统中的所有摄像头设备
- 📊 支持扫描多种预设分辨率（从 QQVGA 到 4K）
- ⚡ 支持扫描多种帧率（从 5fps 到 60fps）
- ➕ 支持添加自定义分辨率和帧率
- 🔍 实时预览指定分辨率和帧率的画面
- 📝 显示实际输出的分辨率和帧率参数
- 🌐 纯前端实现，无需后端服务
- 🎯 支持主流浏览器

## 技术栈

- Vue 3
- Ant Design Vue
- WebRTC API
- MediaDevices API

## 安装和使用

### 环境要求

- Node.js >= 16
- npm >= 7 或 yarn >= 1.22

### 安装依赖

```bash
# 使用 npm
npm install

# 或使用 yarn
yarn install
```

### 开发环境运行

```bash
# 使用 npm
npm run dev

# 或使用 yarn
yarn dev
```

### 生产环境构建

```bash
# 使用 npm
npm run build

# 或使用 yarn
yarn build
```

## 使用说明

1. 启动应用后，选择要测试的摄像头设备
2. 默认已预设常用分辨率和帧率组合
3. 可以通过界面添加自定义的分辨率和帧率
4. 点击"开始扫描"按钮开始测试
5. 扫描完成后，可以预览每个支持的配置

## 浏览器兼容性

- Chrome >= 88
- Firefox >= 85
- Edge >= 88
- Safari >= 14

## 注意事项

- 使用前请确保浏览器已获得摄像头访问权限
- 部分摄像头可能不支持某些分辨率和帧率的组合
- 实际输出的分辨率和帧率可能与请求的参数有所差异
- 扫描过程可能需要一些时间，请耐心等待

## 贡献指南

欢迎提交 Issue 和 Pull Request 来帮助改进这个项目。

## 许可证

[MIT License](LICENSE) 