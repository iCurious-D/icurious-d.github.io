+++
date = '2026-08-30T15:00:00+08:00'
draft = false
title = '汉字组词大挑战'
tags = ["教育", "语文"]
description = "给一个汉字，尽可能多地组出词语，锻炼词汇量和联想能力。"
+++

给你一个汉字，你能组出多少个词语？试试看！

<div style="text-align:center; margin:2rem 0;">
<div style="margin-bottom:1rem;">
<label style="font-weight:bold; margin-right:0.5rem;">难度：</label>
<select id="wordDifficulty" style="padding:0.4rem; border-radius:4px; border:1px solid #ccc;">
<option value="easy">简单（常见字）</option>
<option value="medium">中等</option>
<option value="hard">困难（生僻字）</option>
</select>
</div>
<div id="gameArea">
<div style="font-size:1rem; color:#888; margin-bottom:0.5rem;">请用这个字组词：</div>
<div id="targetChar" style="font-size:4rem; font-weight:bold; color:#4a9eff; margin:1rem 0; min-height:5rem;"></div>
<div style="max-width:400px; margin:0 auto;">
<input id="wordInput" type="text" placeholder="输入一个词语（必须包含上面的字）" style="width:100%; padding:0.6rem; font-size:1.1rem; border:2px solid #4a9eff; border-radius:6px; text-align:center;" onkeydown="if(event.key==='Enter')submitWord()">
<div style="margin-top:0.5rem; display:flex; gap:0.5rem; justify-content:center;">
<button onclick="submitWord()" style="padding:0.5rem 1.5rem; cursor:pointer; background:#4a9eff; color:#fff; border:none; border-radius:4px;">提交</button>
<button onclick="newRound()" style="padding:0.5rem 1rem; cursor:pointer; background:#2ecc71; color:#fff; border:none; border-radius:4px;">换一个字</button>
</div>
</div>
<div id="wordFeedback" style="font-size:1.1rem; margin-top:1rem; min-height:1.5rem;"></div>
<div style="margin-top:1.5rem;">
<div style="font-weight:bold; margin-bottom:0.5rem;">你已经组出的词语：</div>
<div id="wordList" style="display:flex; flex-wrap:wrap; gap:0.5rem; justify-content:center; min-height:2rem;"></div>
</div>
<div style="margin-top:1rem; font-size:0.95rem; color:#666;">
已组出：<strong id="wordCount">0</strong> 个词 | 参考答案还有：<strong id="remainingCount">0</strong> 个
</div>
</div>
</div>

<script>
const WORD_BANK = {
  easy: [
    { char: '花', words: ['花朵','花园','鲜花','花瓣','花盆','开花','花纹','火花','烟花','棉花','花生','花篮'] },
    { char: '水', words: ['水果','水滴','水分','水平','水流','河水','口水','水果','水花','水杯','雨水','汗水'] },
    { char: '天', words: ['天空','天气','天上','今天','明天','昨天','白天','天使','天然','天才','天地','秋天'] },
    { char: '风', words: ['风车','风景','大风','风沙','台风','微风','风筝','风暴','风格','风俗','风光','刮风'] },
    { char: '月', words: ['月亮','月光','日月','月饼','月份','一月','月牙','明月','半月','月牙','月光','岁月'] },
    { char: '山', words: ['山水','山上','火山','山峰','山谷','爬山','山林','山区','山路','高山','青山','山河'] },
  ],
  medium: [
    { char: '意', words: ['意思','意义','意见','意外','注意','满意','愿意','创意','随意','故意','生意','意识'] },
    { char: '明', words: ['明天','明亮','明白','光明','聪明','明显','发明','文明','说明','明确','证明','透明'] },
    { char: '思', words: ['思考','思想','意思','思念','思路','心思','沉思','反思','相思','深思','构思','思维'] },
    { char: '行', words: ['行动','行为','行走','进行','银行','旅行','步行','行业','执行','品行','行李','通行'] },
  ],
  hard: [
    { char: '逸', words: ['安逸','逃逸','飘逸','逸事','劳逸','逸闻','逸出','闲逸','清逸','逸趣','逸群','超逸'] },
    { char: '蕴', words: ['蕴含','底蕴','蕴藏','蕴育','蕴涵','蕴蓄','蕴藉','蕴酿','蕴结','内蕴','蕴聚','精蕴'] },
  ]
};

let currentChar = '', validWords = [], foundWords = [];

function newRound() {
  const diff = document.getElementById('wordDifficulty').value;
  const bank = WORD_BANK[diff];
  const item = bank[Math.floor(Math.random() * bank.length)];
  currentChar = item.char;
  validWords = item.words.map(w => w.toLowerCase());
  foundWords = [];
  document.getElementById('targetChar').textContent = currentChar;
  document.getElementById('wordInput').value = '';
  document.getElementById('wordFeedback').textContent = '';
  document.getElementById('wordList').innerHTML = '';
  document.getElementById('wordCount').textContent = 0;
  document.getElementById('remainingCount').textContent = validWords.length;
  document.getElementById('wordInput').focus();
}

function submitWord() {
  const input = document.getElementById('wordInput').value.trim();
  if (!input) return;
  if (input.length < 2) {
    document.getElementById('wordFeedback').innerHTML = '<span style="color:#f39c12;">词语至少需要两个字哦</span>';
    return;
  }
  if (!input.includes(currentChar)) {
    document.getElementById('wordFeedback').innerHTML = '<span style="color:#e74c3c;">词语中必须包含"' + currentChar + '"字</span>';
    return;
  }
  if (foundWords.includes(input)) {
    document.getElementById('wordFeedback').innerHTML = '<span style="color:#f39c12;">这个词语已经组过了</span>';
    return;
  }
  foundWords.push(input);
  const isValid = validWords.includes(input);
  const color = isValid ? '#2ecc71' : '#9b59b6';
  const label = isValid ? '✓ 参考答案中有这个词！' : '✓ 新词语，参考答案里没有，但也不错！';
  document.getElementById('wordFeedback').innerHTML = '<span style="color:' + color + ';">' + label + '</span>';

  const tag = document.createElement('span');
  tag.style.cssText = 'background:' + color + '; color:#fff; padding:0.3rem 0.7rem; border-radius:12px; font-size:0.95rem;';
  tag.textContent = input;
  document.getElementById('wordList').appendChild(tag);

  document.getElementById('wordCount').textContent = foundWords.length;
  const matched = foundWords.filter(w => validWords.includes(w)).length;
  document.getElementById('remainingCount').textContent = Math.max(0, validWords.length - matched);
  document.getElementById('wordInput').value = '';
  document.getElementById('wordInput').focus();
}

newRound();
</script>

---

### 关于这个工具

汉字组词游戏，锻炼小朋友的词汇量和联想能力。

玩法：
1. 看上方的目标汉字
2. 在输入框中输入包含这个字的词语（至少两个字）
3. 尽可能多地组出不同的词语
4. 点击"换一个字"可以开始新一轮

三个难度：简单用常见字（花、水、天），困难用相对生僻的字（逸、蕴）。
