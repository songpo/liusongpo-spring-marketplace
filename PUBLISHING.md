# 发布到 GitHub 的步骤

## 1. 在 GitHub 上创建仓库

访问 https://github.com/new 创建一个新仓库：
- Repository name: `liusongpo-springboot-marketplace`
- Description: `Spring Boot Skills Marketplace - 专注于 Spring Boot 开发的技能市场`
- Public（公开仓库，这样其他人才能使用）
- 不要初始化 README、.gitignore 或 LICENSE（我们已经创建了）

## 2. 推送到 GitHub

```bash
cd D:\Workspace\IdeaProjects\liusongpo-springboot-marketplace

# 添加远程仓库
git remote add origin https://github.com/songpo/liusongpo-springboot-marketplace.git

# 推送到 GitHub
git push -u origin master
```

## 3. 安装使用

其他用户可以通过以下方式安装：

```bash
# 添加你的 marketplace
claude plugin marketplace add https://github.com/songpo/liusongpo-springboot-marketplace

# 安装所有技能
claude plugin install liusongpo-springboot-skills@liusongpo-springboot-marketplace

# 或单独安装
claude plugin install springboot-ddd-cn@liusongpo-springboot-marketplace
claude plugin install springboot-testing-cn@liusongpo-springboot-marketplace
```

## 4. 添加新技能

当你想添加新的 Spring Boot 技能时：

1. 在 marketplace 根目录创建新的技能目录，例如 `springboot-security-cn/`
2. 创建 `.claude-plugin/plugin.json` 配置文件
3. 创建 `SKILL.md` 技能内容文件
4. 更新根目录的 `.claude-plugin/marketplace.json`，在 `plugins` 数组中添加新技能
5. 提交并推送到 GitHub

## 5. 版本管理

当你更新技能内容时：

1. 修改技能的 `SKILL.md` 文件
2. 更新 `.claude-plugin/plugin.json` 中的版本号
3. 提交并推送
4. 用户可以通过 `claude plugin update` 命令更新

## 目录结构

```
liusongpo-springboot-marketplace/
├── .claude-plugin/
│   ├── marketplace.json      # Marketplace 配置
│   └── plugin.json           # 插件元数据
├── springboot-ddd-cn/
│   ├── .claude-plugin/
│   │   └── plugin.json       # DDD 技能配置
│   └── SKILL.md              # DDD 技能内容
├── springboot-testing-cn/
│   ├── .claude-plugin/
│   │   └── plugin.json       # 测试技能配置
│   └── SKILL.md              # 测试技能内容
├── .gitignore
├── LICENSE
└── README.md
```

## 注意事项

- 每个技能都需要自己的 `.claude-plugin/plugin.json` 文件
- `SKILL.md` 文件开头必须包含 frontmatter（`---` 包围的元数据）
- 版本号遵循语义化版本规范（Semantic Versioning）
- 技能名称使用小写字母和连字符
