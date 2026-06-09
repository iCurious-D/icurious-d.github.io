+++
date = '2026-06-08T14:04:38+08:00'
draft = false
title = '每日随机一句话'

+++

点击按钮，获取一句随机名言。

<div style="text-align:center; margin: 2rem 0;">
  <p id="quote" style="font-size: 1.2rem; font-style: italic; min-height: 3rem;">✨ 等待你的点击 ✨</p>
  <button id="btn" onclick="newQuote()" style="padding: 0.5rem 1.5rem; cursor: pointer;">换一句</button>
</div>

<script>
const quotes = [
  "别想，去行动。",
  "种一棵树最好的时间是十年前，其次是现在。",
  "完美是优秀的敌人。",
  "简单比复杂更难，你必须努力让你的思维变得清晰。",
  "你的时间有限，不要浪费在重复别人的生活上。"
];
function newQuote() {
  const random = Math.floor(Math.random() * quotes.length);
  document.getElementById('quote').innerText = quotes[random];
}
// 页面加载时随机显示一句
window.onload = newQuote;
</script>

---

### 关于这个小工具

用原生 JavaScript 写的一个简单随机语句生成器，未来可以升级成调用 API 的版本。