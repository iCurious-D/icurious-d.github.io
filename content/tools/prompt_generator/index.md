+++
date = '2026-08-30T11:00:00+08:00'
draft = false
title = 'Prompt 生成器'
tags = ["AI", "Prompt", "工具"]
description = "通过表单填写角色、任务、格式、约束等要素，一键生成结构化的高质量 Prompt。"
+++

通过填写表单要素，自动生成结构化的 Prompt，可直接复制使用。

<div style="margin: 2rem 0; display:grid; gap:1rem;">
<div>
<label style="font-weight:bold; display:block; margin-bottom:0.3rem;">角色（Role）</label>
<input id="role" type="text" placeholder="例如：资深 Python 工程师、数据分析师" style="width:100%; padding:0.6rem; border:1px solid #ccc; border-radius:4px; font-size:0.95rem;">
</div>
<div>
<label style="font-weight:bold; display:block; margin-bottom:0.3rem;">任务（Task）</label>
<textarea id="task" rows="2" placeholder="例如：审查以下代码并指出潜在的性能问题" style="width:100%; padding:0.6rem; border:1px solid #ccc; border-radius:4px; font-size:0.95rem;"></textarea>
</div>
<div>
<label style="font-weight:bold; display:block; margin-bottom:0.3rem;">上下文（Context）</label>
<textarea id="context" rows="3" placeholder="例如：这是一段处理用户日志的 Python 脚本，日志文件约 10GB..." style="width:100%; padding:0.6rem; border:1px solid #ccc; border-radius:4px; font-size:0.95rem;"></textarea>
</div>
<div>
<label style="font-weight:bold; display:block; margin-bottom:0.3rem;">输出格式（Format）</label>
<select id="format" style="width:100%; padding:0.6rem; border:1px solid #ccc; border-radius:4px; font-size:0.95rem;">
<option value="">-- 选择输出格式 --</option>
<option value="Markdown 文本">Markdown 文本</option>
<option value="JSON 格式">JSON 格式</option>
<option value="Markdown 表格，包含问题分析和建议">Markdown 表格</option>
<option value="分步骤的列表，每步包含说明和代码示例">分步骤列表</option>
<option value="简洁的要点列表">要点列表</option>
<option value="自定义">自定义格式</option>
</select>
<input id="formatCustom" type="text" placeholder="输入自定义格式描述..." style="width:100%; padding:0.6rem; border:1px solid #ccc; border-radius:4px; font-size:0.95rem; margin-top:0.3rem; display:none;">
</div>
<div>
<label style="font-weight:bold; display:block; margin-bottom:0.3rem;">约束条件（Constraint）</label>
<textarea id="constraint" rows="2" placeholder="例如：只关注性能问题，不要修改代码风格；回答用中文；不超过 500 字" style="width:100%; padding:0.6rem; border:1px solid #ccc; border-radius:4px; font-size:0.95rem;"></textarea>
</div>
<div>
<label style="font-weight:bold; display:block; margin-bottom:0.3rem;">示例（Few-shot，可选）</label>
<textarea id="examples" rows="3" placeholder="输入 → 输出的示例，每行一组，用 --- 分隔多组示例" style="width:100%; padding:0.6rem; border:1px solid #ccc; border-radius:4px; font-size:0.95rem;"></textarea>
</div>
<div style="display:flex; gap:0.5rem; flex-wrap:wrap;">
<button onclick="generatePrompt()" style="padding:0.5rem 1.5rem; cursor:pointer; background:#4a9eff; color:#fff; border:none; border-radius:4px;">生成 Prompt</button>
<button onclick="copyPrompt()" id="copyBtn" style="padding:0.5rem 1rem; cursor:pointer; background:#2ecc71; color:#fff; border:none; border-radius:4px; display:none;">复制</button>
<button onclick="clearForm()" style="padding:0.5rem 1rem; cursor:pointer; background:#eee; border:1px solid #ccc; border-radius:4px;">清空</button>
</div>
</div>
<div id="outputArea" style="display:none; margin-top:1rem;">
<h3>生成的 Prompt</h3>
<pre id="promptOutput" style="background:#1e1e1e; color:#d4d4d4; padding:1rem; border-radius:6px; white-space:pre-wrap; word-wrap:break-word; font-size:0.9rem; line-height:1.6; max-height:500px; overflow-y:auto;"></pre>
</div>

<script>
document.getElementById('format').addEventListener('change', function() {
  document.getElementById('formatCustom').style.display = this.value === '自定义' ? 'block' : 'none';
});

function generatePrompt() {
  const role = document.getElementById('role').value.trim();
  const task = document.getElementById('task').value.trim();
  const context = document.getElementById('context').value.trim();
  const formatSel = document.getElementById('format').value;
  const formatCustom = document.getElementById('formatCustom').value.trim();
  const format = formatSel === '自定义' ? formatCustom : formatSel;
  const constraint = document.getElementById('constraint').value.trim();
  const examples = document.getElementById('examples').value.trim();

  if (!task) { alert('请至少填写"任务"字段'); return; }

  let prompt = '';

  if (role) {
    prompt += '# 角色\n你是一位' + role + '。\n\n';
  }

  prompt += '# 任务\n' + task + '\n\n';

  if (context) {
    prompt += '# 上下文\n' + context + '\n\n';
  }

  if (format) {
    prompt += '# 输出要求\n- 格式：' + format + '\n- 语言：中文\n\n';
  }

  if (constraint) {
    const lines = constraint.split(/[;；\n]/).map(s => s.trim()).filter(Boolean);
    if (format) {
      // 追加到已有的输出要求
      prompt = prompt.replace(/\n\n$/, '\n');
      for (var i = 0; i < lines.length; i++) {
        prompt += '- ' + lines[i] + '\n';
      }
      prompt += '\n';
    } else {
      prompt += '# 约束条件\n';
      for (var i = 0; i < lines.length; i++) {
        prompt += '- ' + lines[i] + '\n';
      }
      prompt += '\n';
    }
  }

  if (examples) {
    prompt += '# 示例\n';
    const exampleList = examples.split(/---|\n{2,}/).map(s => s.trim()).filter(Boolean);
    exampleList.forEach(function(ex, i) {
      prompt += '示例 ' + (i + 1) + '：\n' + ex + '\n\n';
    });
  }

  prompt += '# 用户输入\n{在此输入具体内容}';

  document.getElementById('promptOutput').textContent = prompt;
  document.getElementById('outputArea').style.display = 'block';
  document.getElementById('copyBtn').style.display = 'inline-block';
}

function copyPrompt() {
  const text = document.getElementById('promptOutput').textContent;
  navigator.clipboard.writeText(text).then(() => {
    const btn = document.getElementById('copyBtn');
    btn.textContent = '已复制!';
    setTimeout(() => { btn.textContent = '复制'; }, 2000);
  });
}

function clearForm() {
  ['role','task','context','constraint','examples','formatCustom'].forEach(id => {
    document.getElementById(id).value = '';
  });
  document.getElementById('format').selectedIndex = 0;
  document.getElementById('outputArea').style.display = 'none';
  document.getElementById('copyBtn').style.display = 'none';
}
</script>

---

### 关于这个工具

基于 Prompt Engineering 的四要素框架（角色 / 任务 / 格式 / 约束）设计的结构化 Prompt 生成器。

使用方式：
1. 填写各表单字段（只有"任务"是必填的）
2. 点击"生成 Prompt"，自动生成结构化的 Prompt
3. 点击"复制"即可粘贴到 ChatGPT、Claude 等对话界面使用

这个工具本身也体现了 Prompt Engineering 的核心理念——**用结构化的方式组织指令，让 LLM 更容易理解你的意图**。
