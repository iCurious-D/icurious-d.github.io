+++
date = '2026-08-30T12:00:00+08:00'
draft = false
title = 'RAG 分块可视化工具'
tags = ["AI", "RAG", "工具"]
description = "输入一段文本，可视化展示不同分块策略（固定长度 / 语义分块）的效果，直观理解 chunk_size 和 overlap 参数的影响。"
+++

输入一段文本，调整参数，实时查看不同分块策略的效果。

<div style="margin: 2rem 0;">
<label style="font-weight:bold; display:block; margin-bottom:0.3rem;">输入文本</label>
<textarea id="chunkInput" rows="8" placeholder="粘贴一段较长的文本（建议 300 字以上），观察不同分块策略的效果..." style="width:100%; padding:0.8rem; font-size:0.95rem; border:1px solid #ccc; border-radius:6px; resize:vertical;"></textarea>
</div>
<div style="display:grid; grid-template-columns: 1fr 1fr 1fr; gap:1rem; margin-bottom:1.5rem;">
<div>
<label style="font-weight:bold; display:block; margin-bottom:0.3rem;">Chunk Size（字符数）</label>
<input id="chunkSize" type="range" min="50" max="500" value="150" oninput="updateChunks()" style="width:100%;">
<span id="chunkSizeVal" style="font-size:0.9rem; color:#666;">150</span>
</div>
<div>
<label style="font-weight:bold; display:block; margin-bottom:0.3rem;">Overlap（重叠字符数）</label>
<input id="overlap" type="range" min="0" max="100" value="30" oninput="updateChunks()" style="width:100%;">
<span id="overlapVal" style="font-size:0.9rem; color:#666;">30</span>
</div>
<div>
<label style="font-weight:bold; display:block; margin-bottom:0.3rem;">分块策略</label>
<select id="strategy" onchange="updateChunks()" style="width:100%; padding:0.5rem; border:1px solid #ccc; border-radius:4px;">
<option value="fixed">固定长度分块</option>
<option value="sentence">按句子分块</option>
<option value="paragraph">按段落分块</option>
</select>
</div>
</div>

<div id="statsBar" style="display:flex; gap:1rem; flex-wrap:wrap; margin-bottom:1rem; padding:0.8rem; background:#f8f8f8; border-radius:6px;">
<span>分块数：<strong id="chunkCount">0</strong></span>
<span>平均块大小：<strong id="avgSize">0</strong> 字符</span>
<span>最大块：<strong id="maxChunk">0</strong> 字符</span>
<span>最小块：<strong id="minChunk">0</strong> 字符</span>
</div>

<div id="chunksDisplay" style="display:grid; gap:0.5rem;"></div>

<script>
const COLORS = ['#4a9eff', '#ff6b6b', '#2ecc71', '#f39c12', '#9b59b6', '#1abc9c', '#e74c3c', '#3498db'];

function fixedChunk(text, size, overlap) {
  const chunks = [];
  let start = 0;
  while (start < text.length) {
    chunks.push(text.substring(start, start + size));
    start += size - overlap;
  }
  return chunks.filter(c => c.trim().length > 0);
}

function sentenceChunk(text, size, overlap) {
  const sentences = text.match(/[^。！？\n]+[。！？]?/g) || [text];
  const chunks = [];
  let current = '';
  for (const s of sentences) {
    if (current.length + s.length > size && current.length > 0) {
      chunks.push(current);
      // overlap: 保留上一块的末尾部分
      if (overlap > 0 && current.length > overlap) {
        current = current.slice(-overlap) + s;
      } else {
        current = s;
      }
    } else {
      current += s;
    }
  }
  if (current.trim()) chunks.push(current);
  return chunks;
}

function paragraphChunk(text, size, overlap) {
  const paragraphs = text.split(/\n{2,}|\n/).filter(p => p.trim().length > 0);
  const chunks = [];
  let current = '';
  for (const p of paragraphs) {
    if (current.length + p.length > size && current.length > 0) {
      chunks.push(current);
      if (overlap > 0 && current.length > overlap) {
        current = current.slice(-overlap) + '\n' + p;
      } else {
        current = p;
      }
    } else {
      current += (current ? '\n' : '') + p;
    }
  }
  if (current.trim()) chunks.push(current);
  return chunks;
}

function updateChunks() {
  const text = document.getElementById('chunkInput').value.trim();
  const size = parseInt(document.getElementById('chunkSize').value);
  const overlap = parseInt(document.getElementById('overlap').value);
  const strategy = document.getElementById('strategy').value;

  document.getElementById('chunkSizeVal').textContent = size;
  document.getElementById('overlapVal').textContent = overlap;

  if (!text) {
    document.getElementById('chunksDisplay').innerHTML = '<p style="color:#888; text-align:center;">请在上方输入文本</p>';
    return;
  }

  let chunks;
  switch (strategy) {
    case 'sentence': chunks = sentenceChunk(text, size, overlap); break;
    case 'paragraph': chunks = paragraphChunk(text, size, overlap); break;
    default: chunks = fixedChunk(text, size, overlap);
  }

  const sizes = chunks.map(c => c.length);
  const maxLen = Math.max(...sizes);
  document.getElementById('chunkCount').textContent = chunks.length;
  document.getElementById('avgSize').textContent = Math.round(sizes.reduce((a, b) => a + b, 0) / sizes.length);
  document.getElementById('maxChunk').textContent = Math.max(...sizes);
  document.getElementById('minChunk').textContent = Math.min(...sizes);

  document.getElementById('chunksDisplay').innerHTML = chunks.map(function(chunk, i) {
    var color = COLORS[i % COLORS.length];
    var widthPct = (chunk.length / maxLen) * 100;
    var preview = chunk.length > 80 ? chunk.substring(0, 80) + '...' : chunk;
    return '<div style="border:1px solid ' + color + '30; border-radius:6px; padding:0.8rem; border-left:4px solid ' + color + ';">' +
      '<div style="display:flex; justify-content:space-between; margin-bottom:0.3rem;">' +
      '<span style="font-weight:bold; color:' + color + ';">Chunk ' + (i + 1) + '</span>' +
      '<span style="font-size:0.85rem; color:#888;">' + chunk.length + ' 字符</span>' +
      '</div>' +
      '<div style="background:#f0f0f0; border-radius:3px; height:6px; margin-bottom:0.4rem;">' +
      '<div style="width:' + widthPct + '%; background:' + color + '; height:100%; border-radius:3px;"></div>' +
      '</div>' +
      '<div style="font-size:0.85rem; color:#555; line-height:1.5; word-break:break-all;">' + preview + '</div>' +
      '</div>';
  }).join('');
}

document.getElementById('chunkInput').addEventListener('input', updateChunks);
</script>

---

### 关于这个工具

RAG 系统中，文档分块（Chunking）是影响检索效果的关键环节。这个工具帮助你直观理解：

1. **Chunk Size**：每个分块的大小。太小会丢失上下文，太大会引入噪音
2. **Overlap**：相邻块之间的重叠字符数。适当的 overlap 可以避免信息在切分边界处丢失
3. **分块策略**：
   - **固定长度**：按字符数硬切，简单但可能截断语义
   - **按句子**：在句子边界处切分，保持语义完整性
   - **按段落**：在段落边界处切分，适合结构清晰的文档

拖动滑块调整参数，观察分块数量和大小分布的变化，找到最适合你数据的参数组合。
