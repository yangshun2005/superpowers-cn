# GitHub Issue

`gh` 已安装并完成认证时优先使用它：它能处理认证、速率限制和 JSON。否则退回用 curl 调用公开 API，再不行就给你搭档一个 URL 让他自己打开。

## 搜索

```bash
gh search issues --repo obra/superpowers --limit 10 "<terms>" \
  --json number,state,title --jq '.[] | "\(.number)\t\(.state)\t\(.title)"'
```

没有 `gh` 时（未认证，每分钟 10 次请求）：

```bash
curl -s -H "Accept: application/vnd.github+json" \
  "https://api.github.com/search/issues?q=repo:obra/superpowers+is:issue+<url-encoded terms>&per_page=10" \
  | jq -r '.items[] | "\(.number)\t\(.state)\t\(.title)"'
```

没有 curl 时，直接给出 `https://github.com/obra/superpowers/issues?q=<terms>`。

## 提交

把填好的 `templates/issue.md` 写入工作区，并展示完整文本。确认后执行：

```bash
gh issue create --repo obra/superpowers --title "<title>" --body-file <path> \
  --label bug --label automated-issue-report
```

报告者没有 push 权限时，GitHub 会静默丢弃标签，因此标签只对协作者生效；模板页脚仍会标明该 Issue 由技能提交。`gh` 无法附加文件：Issue 创建后，把打包文件路径交给搭档，让他在浏览器中附加。

没有 `gh` 时，给出基于 `bug_report.md` 模板的预填链接，任何报告者使用该模板都会自动打上 `bug` 标签：

```
https://github.com/obra/superpowers/issues/new?template=bug_report.md&title=<url-encoded title>&body=<url-encoded body>
```

GitHub 会拒绝超过约 8,000 字符的 URL；超出时只带标题发送链接，并让搭档从文件中粘贴正文。