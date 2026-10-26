# 📋 ywsj 项目开发规范

## 一、项目结构标准

```
project-name/
├── app.py                 # Flask 主应用
├── (核心模块).py           # 如 asr_engine.py / translator.py
├── templates/
│   ├── index.html         # 主页面
│   ├── setup.html         # 首次初始化设置密码
│   └── login.html         # 登录页
├── assets/
│   └── donation.jpg       # 打赏码
├── data/                  # 持久化数据（.gitignore）
├── Dockerfile
├── docker-compose.yml
├── entrypoint.sh          # gosu 降权 + 修复/data权限
├── requirements.txt
├── .github/workflows/
│   └── docker-publish.yml # CI/CD
└── README.md
```

---

## 二、代码规范

### 后端 (Python/Flask)
1. 密码用 **werkzeug scrypt** 加盐哈希，禁止明文/MD5
2. 数据存储用 **SQLite**，不用 JSON 文件
3. 首次部署 `/setup` 路由初始化密码，`is_auth_enabled()` 判断是否已初始化
4. 防暴力破解：同 IP 失败 5 次锁定 15 分钟
5. session 用 `session.permanent = True`
6. Google 翻译用 **clients5 接口** 避免 429
7. 日志用 `logging` 模块，关键错误记录
8. 第三方依赖尽量少，能不依赖就不依赖
9. **绝不依赖不可控的第三方服务做核心功能**（教训：audio-to-subtitle 曾依赖 bkfeng.top 签名服务导致全挂）
10. 多引擎/多后端时，默认用**依赖最少**的那个，其他作为备用

### 前端 (HTML/CSS/JS)
1. **UI 必须一步到位**，不做简陋版再迭代（用户原话："界面也太烂了"是质量信号）
2. 暗色/亮色主题切换（CSS 变量 `--bg --card --border --text --muted --accent`）
3. 卡片式布局 + 阴影 + 圆角 + 过渡动画 + 渐变色按钮，hover 有反馈
4. 响应式布局，移动端可用
5. 所有交互有 loading 状态和进度反馈
6. GitHub 图标放右上角（每个页面都要有）
7. 模态框 `max-height: 90vh` + flex布局 + overflow滚动

### Dockerfile
- 基础镜像 `python:3.x-slim`
- 创建非root用户 `appuser`
- gosu 降权运行
- entrypoint.sh 自动修复 `/data` 权限

### entrypoint.sh
```bash
#!/bin/sh
if [ "$(id -u)" = '0' ]; then
    chown -R appuser:appgroup /data
    exec gosu appuser "$@"
fi
exec "$@"
```

---

## 三、CI/CD 标准

### 触发条件
- push 到 main/master，**paths 过滤核心文件**变更才触发
- CI 内部自动生成 tag（不需要手动 push tag）

### 版本号格式
```
vYYYYMMDD-N
```
- UTC+8（北京时间）
- N 是当天第几次（从1开始）
- **永远不用语义化版本**（不用 v1.0.0），用户明确要求用日期版本

### CI 流程（标准10步）
1. Checkout（fetch-depth: 0）
2. Generate version tag（`vYYYYMMDD-HHMM`）
3. Set up QEMU
4. Set up Docker Buildx
5. Login to Docker Hub（secrets.DOCKERHUB_USERNAME + TOKEN）
6. Build and push（`linux/amd64,linux/arm64`，`build-args: APP_VERSION`，tags: latest + 版本号）
7. Create Git tag（自动打tag）
8. Create GitHub Release（`generate_release_notes: true`）
9. Sync Docker Hub description（同步README到Docker Hub）
10. 权限：`contents: write`（Release必须）

### 多架构
- amd64 + arm64
- 优先用 GitHub Actions 原生 ARM runner（`ubuntu-24.04-arm`），不用 QEMU
- 见 docker-packaging skill 的 split-arch-build 模式

---

## 四、README 标准

### 必须包含的10个部分
1. **标题 + emoji**（如 `# 🎙️ ywsj-audio-to-subtitle`）
2. **一句话简介**
3. **功能特性**（✨ 带emoji列表）
4. **快速部署**（Docker run + Docker Compose 两版本）
5. **使用方法**（步骤说明）
6. **技术栈**
7. **项目结构**
8. **致谢**（参考的开源项目）
9. **打赏码**（图片用 `raw.githubusercontent.com` 绝对路径）
10. **License**（MIT）

### Docker Hub 页面
- CI 自动同步 README
- 图片**必须用** `https://raw.githubusercontent.com/{org}/{repo}/{branch}/assets/x.jpg` 绝对路径

---

## 五、部署规范

### docker-compose.yml 模板
```yaml
services:
  app-name:
    image: ywsj/app-name:latest
    container_name: app-name
    ports:
      - "端口:5200"
    volumes:
      - ./data:/data
    restart: always
```

### 热修复流程
```bash
# 1. 修改本地代码
# 2. docker cp 到容器（注意权限！）
docker cp file.py container:/app/file.py
docker exec -u root container chown appuser:appgroup /app/file.py
docker exec -u root container chmod 644 /app/file.py
docker restart container
# 3. 验证 curl localhost:端口/health
# 4. git push 触发CI构建新镜像
```

### ⚠️ 重建容器注意
`docker compose up -d` / `docker compose up -d --force-recreate` 会使用 Docker Hub 镜像，**会覆盖热修复的代码**。热修复后如果重建容器，必须重新 `docker cp`。

### ⚠️ docker cp 权限陷阱
`docker cp` 默认 root 所有权，非root容器会崩溃。必须：
```bash
docker exec -u root container chown appuser:appgroup /app/file.py
docker exec -u root container chmod 644 /app/file.py
```

---

## 六、测试规范

### 上线前必测
1. **用真实数据测试**，不能只测连通性
   - audio-to-subtitle：用 gTTS 生成真实人声音频验证识别结果
   - translator：用真实字幕文件验证翻译质量
2. 单元测 API 调用 → 集成测完整流程 → Web 界面测
3. 引擎/服务类：测正弦波（API连通性）+ 测真实人声（识别准确率）

### 测试工具
- gTTS 生成人声测试音频（edge-tts 经常403）
- ffmpeg 生成正弦波音频 `ffmpeg -f lavfi -i "sine=frequency=440:duration=3"`
- curl 测 Web 接口（先 setup → 登录拿cookie → 测功能）

### 自验证（用户明确要求）
用户会反复问"你自己测试下" / "好了吗" / "不行"。必须在报告完成前自己 curl 验证：
1. `curl /health` 确认服务正常
2. `curl /api/engines` 确认引擎列表
3. `curl -c cookies -X POST /login` 登录
4. 实际调用核心功能验证结果
5. 只有全部通过才能说"搞定了"

---

## 七、安全规范

1. **首次 `/setup` 初始化**，不硬编码密码
2. 密码 scrypt 哈希，**不记录任何明文密码**
3. 防暴力破解（5次失败锁15分钟）
4. 数据库 host 限制（不用 `%` 全放开）
5. fail2ban 保护 SSH
6. Web 终端不开放
7. 跨机 SSH 运维主动说明认证方式（密钥/密码）

---

## 八、更新通知格式

发布新版时的通知模板：
```
**🔊 项目名 vX.Y.Z 更新公告**

🎯 **重点更新：**
简述最重要的变化

📋 **本次更新内容：**
1. ✅ 变更1
2. ✅ 变更2

🌐 **项目地址：**
GitHub: link
Docker: link

📦 **更新方式：**
docker pull ywsj/xxx:latest
docker compose down && docker compose up -d
```

---

## 九、常见坑（Pitfalls）

### 架构层面
1. **绝不移除可用的引擎/后端做"清理"** — audio-to-subtitle 曾移除BCut只留剪映，剪映签名服务429后整个工具全挂。多引擎时默认用依赖最少的。
2. **第三方免费API可能随时失效**，必有备用引擎
3. **Javet V8 需 glibc**，不能用 Alpine（sonovel）
4. **windres 仅 amd64**，arm64 跑不了（sonovel）

### 部署层面
5. **rm -rf 项目目录 + git clone 会摧毁运行中容器 bind-mount**，clone 前先移走 data/（见 docker-packaging pitfall #73）
6. **docker compose up 重建容器用旧镜像**，热修复代码会被覆盖，需重新 cp
7. **docker cp 默认 root 所有权**，非root容器会 PermissionError 崩溃，必须 chown+chmod
8. **宝塔 PHP-FPM/Redis 自启需补 LSB header**
9. **apt 系统 Redis 冲突 → 禁用系统版**
10. **edge-tts 被微软封 token → 升最新版修复**（7.2.8→7.3.1）

### CI 层面
11. **GitHub token 过期**：`unset GH_TOKEN` + `git credential fill` 提取有效token重认证
12. **force-push tag 后要重新确认 Docker Hub 镜像时间是最新的那次构建**
13. **`generate_release_notes: true` 对非PR项目生成空body**，改用 `git log` 提取 commit msg
14. **CI 没跑完时先 docker cp 热修复**，不要等 CI 跑完

### 前端层面
15. **版本号模板用 `{{ APP_VERSION }}` 不用 `v{{ }}`**，否则显示 `vv20261026-1`
16. **CSS var() 不要带 hardcoded fallback**，会破坏 light theme
17. **`::after` 伪元素不能 querySelector**，会静默抛异常

---

## 十、参考资源

- **docker-packaging skill** — Docker 打包的完整细节（CI模板、多架构、权限修复、Docker Hub API等），详见该 skill
- **用户偏好**（记忆）：
  - 遇到客户端兼容性问题优先改服务端配置/代码，不让用户改客户端
  - 代码 push 后自动更新 Release+Docker Hub+拉新镜像重启，不要每次问
  - 安全意识强：Web终端已移除，fail2ban封IP，不记录root密码
  - UI 质量要求高：功能能 work 不够，视觉必须精致
