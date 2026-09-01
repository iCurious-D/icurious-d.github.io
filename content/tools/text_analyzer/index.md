+++
date = '2026-08-30T10:00:00+08:00'
draft = false
title = 'AI 文本分析器'
tags = ["AI", "NLP", "工具"]
description = "输入一段文本，自动提取关键词、统计词频、分析情感倾向。纯前端实现，无需 API。"
+++

输入一段中文文本，工具会自动分析关键词、词频和情感倾向。

<div style="margin: 2rem 0;">
<textarea id="inputText" rows="6" placeholder="在此粘贴或输入一段中文文本..." style="width:100%; padding:0.8rem; font-size:1rem; border:1px solid #ccc; border-radius:6px; resize:vertical;"></textarea>
<div style="margin-top:0.8rem; display:flex; gap:0.5rem; flex-wrap:wrap;">
<button onclick="analyzeText()" style="padding:0.5rem 1.5rem; cursor:pointer; background:#4a9eff; color:#fff; border:none; border-radius:4px;">分析</button>
<button onclick="clearAll()" style="padding:0.5rem 1rem; cursor:pointer; background:#eee; border:1px solid #ccc; border-radius:4px;">清空</button>
</div>
</div>
<div id="results" style="display:none;">

### 基本信息

<div id="basicStats" style="display:flex; gap:1.5rem; flex-wrap:wrap; margin:1rem 0;"></div>

### 关键词提取 (TF-IDF)

<div id="keywords" style="display:flex; flex-wrap:wrap; gap:0.5rem; margin:1rem 0;"></div>

### 词频统计 Top 10

<div id="freqChart" style="margin:1rem 0;"></div>

### 情感分析

<div id="sentiment" style="margin:1rem 0; padding:1rem; border-radius:6px;"></div>

</div>

<script>
// 中文停用词表（精简版）
const STOPWORDS = new Set(['的','了','在','是','我','有','和','就','不','人','都','一','一个','上','也','很',
  '到','说','要','去','你','会','着','没有','看','好','自己','这','他','她','它','们','那','里','为',
  '什么','吗','呢','吧','啊','哦','嗯','呀','哈','把','被','让','给','从','向','对','但','而',
  '或','如果','因为','所以','虽然','但是','而且','并且','可以','可能','应该','已经','还是','或者',
  '之','与','及','等','中','以','及','其','将','于','则','此','该','这些','那些','这个','那个']);

// 简易中文分词（基于字符n-gram + 规则）
function segment(text) {
  // 去除标点，提取连续的汉字/字母序列
  const words = [];
  const patterns = text.match(/[\u4e00-\u9fa5]+|[a-zA-Z]+/g);
  if (!patterns) return words;
  for (const seg of patterns) {
    // 对中文做 2-gram 和 3-gram 切分作为简易分词
    if (/[\u4e00-\u9fa5]/.test(seg)) {
      for (let i = 0; i < seg.length - 1; i++) {
        const bigram = seg.substring(i, i + 2);
        if (!STOPWORDS.has(bigram)) words.push(bigram);
      }
      for (let i = 0; i < seg.length - 2; i++) {
        const trigram = seg.substring(i, i + 3);
        if (!STOPWORDS.has(trigram)) words.push(trigram);
      }
      // 单字也保留
      for (const ch of seg) {
        if (!STOPWORDS.has(ch) && seg.length <= 4) words.push(ch);
      }
    } else {
      words.push(seg.toLowerCase());
    }
  }
  return words;
}

// 词频统计
function countFreq(words) {
  const freq = {};
  for (const w of words) {
    freq[w] = (freq[w] || 0) + 1;
  }
  return freq;
}

// 简易 TF-IDF（将文本按句号分段作为"文档"）
function tfIdf(text) {
  const sentences = text.split(/[。！？\n]+/).filter(s => s.trim().length > 0);
  if (sentences.length === 0) return [];

  const docs = sentences.map(s => segment(s));
  const N = docs.length;

  // 计算 DF（文档频率）
  const df = {};
  for (const doc of docs) {
    const unique = new Set(doc);
    for (const w of unique) {
      df[w] = (df[w] || 0) + 1;
    }
  }

  // 计算每段的 TF-IDF 并汇总
  const scores = {};
  for (const doc of docs) {
    const tf = countFreq(doc);
    const total = doc.length;
    for (const [w, c] of Object.entries(tf)) {
      if (w.length < 2) continue;
      const tfVal = c / total;
      const idfVal = Math.log(N / (1 + (df[w] || 0)));
      scores[w] = (scores[w] || 0) + tfVal * idfVal;
    }
  }

  return Object.entries(scores)
    .sort((a, b) => b[1] - a[1])
    .slice(0, 10);
}

// 情感词典
const POS_WORDS = new Set(['好','棒','优秀','出色','喜欢','满意','开心','快乐','精彩','完美','强大',
  '方便','简单','清晰','高效','稳定','流畅','漂亮','赞','推荐','成功','突破','进步','提升']);
const NEG_WORDS = new Set(['差','烂','糟','坏','失败','失望','难过','痛苦','无聊','困难','问题',
  '错误','崩溃','卡顿','缓慢','复杂','混乱','模糊','缺陷','bug','漏洞','风险','威胁','损失']);

function analyzeSentiment(text) {
  const chars = [...text];
  let pos = 0, neg = 0;
  const posHits = [], negHits = [];
  for (const w of POS_WORDS) {
    if (text.includes(w)) { pos++; posHits.push(w); }
  }
  for (const w of NEG_WORDS) {
    if (text.includes(w)) { neg++; negHits.push(w); }
  }
  const total = pos + neg;
  let label, color;
  if (total === 0) { label = '中性'; color = '#888'; }
  else if (pos > neg) { label = '正面'; color = '#2ecc71'; }
  else if (neg > pos) { label = '负面'; color = '#e74c3c'; }
  else { label = '中性（正负持平）'; color = '#f39c12'; }
  return { label, color, pos, neg, posHits, negHits };
}

function analyzeText() {
  const text = document.getElementById('inputText').value.trim();
  if (!text) { alert('请先输入文本'); return; }

  document.getElementById('results').style.display = 'block';

  // 基本统计
  const charCount = text.replace(/\s/g, '').length;
  const sentenceCount = text.split(/[。！？\n]+/).filter(s => s.trim()).length;
  const words = segment(text);
  const uniqueWords = new Set(words);

  document.getElementById('basicStats').innerHTML = [
    '📝 字符数：<strong>' + charCount + '</strong>',
    '📋 句数：<strong>' + sentenceCount + '</strong>',
    '🔤 词数（分词后）：<strong>' + words.length + '</strong>',
    '📊 去重词数：<strong>' + uniqueWords.size + '</strong>'
  ].map(function(s) { return '<span style="background:#f0f0f0; padding:0.4rem 0.8rem; border-radius:4px;">' + s + '</span>'; }).join('');

  // 关键词（TF-IDF）
  const keywords = tfIdf(text);
  const maxScore = keywords.length > 0 ? keywords[0][1] : 1;
  document.getElementById('keywords').innerHTML = keywords.map(([w, s]) => {
    const size = 0.8 + (s / maxScore) * 0.8;
    return '<span style="background:#4a9eff; color:#fff; padding:0.3rem 0.7rem; border-radius:12px; font-size:' + size + 'rem;">' + w + ' <small>(' + s.toFixed(3) + ')</small></span>';
  }).join('');

  // 词频统计
  const freq = countFreq(words);
  const top10 = Object.entries(freq).filter(([w]) => w.length >= 2).sort((a, b) => b[1] - a[1]).slice(0, 10);
  const maxFreq = top10.length > 0 ? top10[0][1] : 1;
  document.getElementById('freqChart').innerHTML = top10.map(function(item) {
    var w = item[0], c = item[1];
    return '<div style="display:flex; align-items:center; margin:0.3rem 0;">' +
      '<span style="width:60px; text-align:right; margin-right:0.5rem; font-size:0.9rem;">' + w + '</span>' +
      '<div style="flex:1; background:#eee; border-radius:4px; height:22px;">' +
      '<div style="width:' + (c/maxFreq)*100 + '%; background:#4a9eff; height:100%; border-radius:4px; display:flex; align-items:center; justify-content:flex-end; padding-right:6px;">' +
      '<span style="color:#fff; font-size:0.75rem;">' + c + '</span>' +
      '</div></div></div>';
  }).join('');

  // 情感分析
  const sent = analyzeSentiment(text);
  document.getElementById('sentiment').style.background = sent.color + '15';
  document.getElementById('sentiment').style.borderLeft = '4px solid ' + sent.color;
  document.getElementById('sentiment').innerHTML =
    '<div style="font-size:1.2rem; font-weight:bold; color:' + sent.color + ';">情感倾向：' + sent.label + '</div>' +
    '<div style="margin-top:0.5rem; font-size:0.9rem;">' +
    '正面词：' + (sent.posHits.length > 0 ? sent.posHits.join('、') : '无') + '（' + sent.pos + ' 个）<br>' +
    '负面词：' + (sent.negHits.length > 0 ? sent.negHits.join('、') : '无') + '（' + sent.neg + ' 个）' +
    '</div>' +
    '<div style="margin-top:0.5rem; font-size:0.8rem; color:#888;">基于简易情感词典分析，仅供演示。生产环境建议使用专业 NLP 模型。</div>';
}

function clearAll() {
  document.getElementById('inputText').value = '';
  document.getElementById('results').style.display = 'none';
}
</script>

---

### 关于这个工具

纯前端实现的文本分析工具，包含三个功能：

1. **关键词提取**：基于 TF-IDF 算法，将文本按句子分段作为"文档集"，计算每个词的 TF-IDF 值
2. **词频统计**：对分词结果进行频率统计，展示 Top 10 高频词
3. **情感分析**：基于预定义的正负面情感词典，判断文本的整体情感倾向

分词采用中文 bigram + trigram 的简易方案，适用于演示和学习。如需生产级精度，建议使用 jieba 分词或专业 NLP 模型。
