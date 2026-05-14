# 添加新 AI 平台支持

## 步骤

### 1. 用浏览器开发者工具检测元素选择器

| 值 | 找法 |
|---|------|
| **editor** | 在输入框元素上右键 → Copy → Copy selector |
| **sendBtn** | 在发送按钮上右键 → Copy → Copy selector |
| **responseSelector** | 在 AI 回复内容容器上找稳定的选择器 |

### 2. 测试 fillMethod

在浏览器控制台手动测试哪种方式能正确填入内容：

```javascript
// value - 用于 <textarea>
document.querySelector('selector').value = 'test';

// execCommand - 用于 contenteditable div
document.execCommand('insertText', false, 'test');

// paste - 模拟粘贴事件
const dt = new DataTransfer();
dt.setData('text/plain', 'test');
document.querySelector('selector').dispatchEvent(new ClipboardEvent('paste', {clipboardData: dt}));

// prosemirror - 用于富文本编辑器
el.innerHTML = 'test';
```

### 3. 决定 useObserver

- `useObserver: true` - 用 MutationObserver 监听 AI 回复中的 `<tool>` 标签
- `useObserver: false` - 需要注入脚本（`injected.js`），目前未实现

### 4. 添加配置

在 `extension/src/content/index.ts` 的 `getSiteConfig()` 函数中添加：

```typescript
if (h.includes('新平台域名.com'))
  return {
    editor: 'selector',
    sendBtn: 'selector',
    stopBtn: null,
    fillMethod: 'value', // paste | execCommand | value | prosemirror
    useObserver: true,
    responseSelector: 'selector',
  };
```

### 5. 更新 manifest.json

在 `extension/public/manifest.json` 的以下位置添加域名：
- `content_scripts.matches`
- `web_accessible_resources.matches`