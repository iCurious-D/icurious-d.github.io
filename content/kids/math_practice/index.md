+++
date = '2026-08-30T14:30:00+08:00'
draft = false
title = '数学口算大闯关'
tags = ["教育", "数学"]
description = "趣味数学口算练习，支持加减乘除四则运算，可选择难度，自动计分。"
+++

选择难度，挑战口算题，看看你能得多少分！

<div style="text-align:center; margin:2rem 0;">
<div style="margin-bottom:1rem;">
<label style="font-weight:bold; margin-right:0.5rem;">难度：</label>
<select id="difficulty" style="padding:0.4rem; border-radius:4px; border:1px solid #ccc;">
<option value="easy">简单（1-10 加减法）</option>
<option value="medium">中等（1-50 加减法）</option>
<option value="hard">困难（含乘除法）</option>
</select>
<label style="font-weight:bold; margin-left:1rem; margin-right:0.5rem;">题数：</label>
<select id="totalQ" style="padding:0.4rem; border-radius:4px; border:1px solid #ccc;">
<option value="10">10 题</option>
<option value="20" selected>20 题</option>
<option value="50">50 题</option>
</select>
</div>
<div id="quizArea">
<div id="question" style="font-size:2.5rem; font-weight:bold; margin:1.5rem 0; min-height:4rem; color:#333;"></div>
<div style="margin-bottom:1rem;">
<input id="answer" type="number" placeholder="输入答案" style="font-size:1.5rem; text-align:center; padding:0.5rem; width:150px; border:2px solid #4a9eff; border-radius:8px;" onkeydown="if(event.key==='Enter')checkAnswer()">
</div>
<div style="display:flex; gap:0.5rem; justify-content:center;">
<button onclick="checkAnswer()" style="padding:0.5rem 1.5rem; cursor:pointer; background:#4a9eff; color:#fff; border:none; border-radius:4px; font-size:1rem;">确定</button>
<button onclick="skipQuestion()" style="padding:0.5rem 1rem; cursor:pointer; background:#eee; border:1px solid #ccc; border-radius:4px;">跳过</button>
</div>
<div id="feedback" style="font-size:1.3rem; margin-top:1rem; min-height:2rem;"></div>
<div style="margin-top:1.5rem; font-size:1rem; color:#666;">
第 <strong id="currentQ">0</strong> / <strong id="totalQDisplay">20</strong> 题 | 正确：<strong id="correctCount" style="color:#2ecc71;">0</strong> | 用时：<strong id="timer">0:00</strong>
</div>
</div>
<button id="startBtn" onclick="startQuiz()" style="padding:0.8rem 2rem; cursor:pointer; background:#2ecc71; color:#fff; border:none; border-radius:8px; font-size:1.2rem; margin-top:1rem;">开始挑战</button>
<div id="resultArea" style="display:none; margin-top:2rem; padding:1.5rem; background:#f8f8f8; border-radius:8px;">
<div style="font-size:1.8rem; font-weight:bold; margin-bottom:1rem;" id="resultTitle"></div>
<div id="resultDetails" style="font-size:1.1rem; line-height:2;"></div>
<button onclick="startQuiz()" style="padding:0.5rem 1.5rem; cursor:pointer; background:#4a9eff; color:#fff; border:none; border-radius:4px; margin-top:1rem;">再来一轮</button>
</div>
</div>

<script>
let questions = [], current = 0, correct = 0, total = 20, timerInterval, startTime;

function genQuestion(diff) {
  let a, b, op, answer;
  if (diff === 'easy') {
    a = Math.floor(Math.random() * 10) + 1;
    b = Math.floor(Math.random() * 10) + 1;
    op = Math.random() < 0.5 ? '+' : '-';
    if (op === '-' && a < b) [a, b] = [b, a];
    answer = op === '+' ? a + b : a - b;
  } else if (diff === 'medium') {
    a = Math.floor(Math.random() * 50) + 1;
    b = Math.floor(Math.random() * 50) + 1;
    op = Math.random() < 0.5 ? '+' : '-';
    if (op === '-' && a < b) [a, b] = [b, a];
    answer = op === '+' ? a + b : a - b;
  } else {
    const ops = ['+', '-', '×', '÷'];
    op = ops[Math.floor(Math.random() * ops.length)];
    if (op === '×') {
      a = Math.floor(Math.random() * 9) + 2;
      b = Math.floor(Math.random() * 9) + 2;
      answer = a * b;
    } else if (op === '÷') {
      b = Math.floor(Math.random() * 9) + 2;
      answer = Math.floor(Math.random() * 9) + 2;
      a = b * answer;
    } else {
      a = Math.floor(Math.random() * 50) + 1;
      b = Math.floor(Math.random() * 50) + 1;
      if (op === '-' && a < b) [a, b] = [b, a];
      answer = op === '+' ? a + b : a - b;
    }
  }
  return { text: a + ' ' + op + ' ' + b + ' = ?', answer: answer };
}

function startQuiz() {
  const diff = document.getElementById('difficulty').value;
  total = parseInt(document.getElementById('totalQ').value);
  questions = Array.from({ length: total }, () => genQuestion(diff));
  current = 0; correct = 0;
  document.getElementById('totalQDisplay').textContent = total;
  document.getElementById('startBtn').style.display = 'none';
  document.getElementById('resultArea').style.display = 'none';
  document.getElementById('quizArea').style.display = 'block';
  startTime = Date.now();
  timerInterval = setInterval(() => {
    const s = Math.floor((Date.now() - startTime) / 1000);
    document.getElementById('timer').textContent = Math.floor(s / 60) + ':' + (s % 60).toString().padStart(2, '0');
  }, 1000);
  showQuestion();
}

function showQuestion() {
  if (current >= total) { endQuiz(); return; }
  document.getElementById('currentQ').textContent = current + 1;
  document.getElementById('question').textContent = questions[current].text;
  document.getElementById('answer').value = '';
  document.getElementById('feedback').textContent = '';
  document.getElementById('answer').focus();
}

function checkAnswer() {
  const input = document.getElementById('answer').value.trim();
  if (input === '') return;
  const userAns = parseInt(input);
  if (userAns === questions[current].answer) {
    correct++;
    document.getElementById('feedback').innerHTML = '<span style="color:#2ecc71;">✓ 正确！太棒了！</span>';
  } else {
    document.getElementById('feedback').innerHTML = '<span style="color:#e74c3c;">✗ 答案是 ' + questions[current].answer + '</span>';
  }
  document.getElementById('correctCount').textContent = correct;
  current++;
  setTimeout(showQuestion, 800);
}

function skipQuestion() {
  document.getElementById('feedback').innerHTML = '<span style="color:#f39c12;">跳过，答案是 ' + questions[current].answer + '</span>';
  current++;
  setTimeout(showQuestion, 800);
}

function endQuiz() {
  clearInterval(timerInterval);
  const elapsed = Math.floor((Date.now() - startTime) / 1000);
  const min = Math.floor(elapsed / 60);
  const sec = elapsed % 60;
  const pct = Math.round((correct / total) * 100);
  let emoji = pct >= 90 ? '🏆' : pct >= 70 ? '⭐' : pct >= 50 ? '💪' : '📚';
  document.getElementById('quizArea').style.display = 'none';
  document.getElementById('resultArea').style.display = 'block';
  document.getElementById('startBtn').style.display = 'inline-block';
  document.getElementById('resultTitle').textContent = emoji + ' ' + (pct >= 90 ? '太厉害了！' : pct >= 70 ? '很棒！' : pct >= 50 ? '继续加油！' : '多多练习哦！');
  document.getElementById('resultDetails').innerHTML =
    '正确率：<strong>' + correct + '/' + total + '</strong>（' + pct + '%）<br>' +
    '用时：<strong>' + min + '分' + sec + '秒</strong><br>' +
    '平均每题：<strong>' + (elapsed / total).toFixed(1) + '秒</strong>';
}
</script>

---

### 关于这个工具

趣味数学口算练习，适合小学低年级的小朋友。

特点：
- 三个难度等级：简单（10以内加减）、中等（50以内加减）、困难（含乘除法）
- 即时反馈，答对答错立刻显示结果
- 自动计时和计分，挑战自己的最好成绩
