AI 答题助手（通用框架）
一个油猴 / 脚本猫用户脚本：把网页上的题目抓下来，交给任意 OpenAI 兼容的大模型作答，再自动填回页面。
平台差异交给 adapter 层，内置头歌（Educoder）类实训关卡、超星 / 学习通类试卷，其余页面走通用启发式识别。没有 API Key 也能用——导出题目粘给任意对话式 AI，再把答案粘回来。
⚠️ 仅供个人复习、自测与前端自动化学习使用。请遵守你所在平台的用户协议，成绩与后果由使用者自负。
✨ 功能
功能
说明
一键作答
抓题 → 请求 AI → 自动勾选 / 填写
导出题目
把整份卷子复制到剪贴板，可直接粘给任意对话式 AI
导入答案
支持 1: B 逐行格式，也支持 JSON，自动回填
重新检测
页面结构变了、或识别错了，点一下重新扫描
测试连接
填完 Base URL 和 Key 先验证，不用开卷子试错
多平台
通过 adapter 适配，不写死某一个网站
零后端
纯前端，Key 只存在浏览器本地
📦 安装
方式一：脚本猫（推荐）
1. 安装 ScriptCat 脚本猫 或 Tampermonkey 扩展。
2. 打开扩展的「管理面板」→「新建脚本」。
3. 删掉模板内容，把 ai-answer-helper.user.js 全文粘进去 → 保存（Ctrl+S）。
4. 打开任意网页，右下角会出现一个圆形按钮。
方式二：直接拖文件
把 .user.js 文件拖进浏览器窗口，扩展会弹出安装页，点「安装」。
两个脚本别同时装。如果之前装过旧版，先禁用或删除，否则会出现两个悬浮球。
🚀 快速开始
第一步：配置 AI
点右下角圆形按钮展开面板 → 设置，按下表填写：
字段
填什么
服务商预设
选一家；换别家选「自定义」
Base URL
见下方对照表，结尾不要带 /chat/completions
模型名
照抄服务商控制台里的模型 ID
API Key
你的 Key；本地 Ollama 可留空
每次打包题数
默认 8，题目长 / 计算多时调到 5~6 更稳
通用模式最少识别题数
默认 2，防止把点赞按钮当题目
填完点 测试连接，显示「✅ 连通成功」再保存。
常见服务的 Base URL
服务
Base URL
模型名示例
DeepSeek 官方
https://api.deepseek.com/v1
deepseek-chat
阿里云百炼（通义 / DeepSeek）
https://dashscope.aliyuncs.com/compatible-mode/v1
qwen-plus、deepseek-v4-pro
月之暗面 Kimi
https://api.moonshot.cn/v1
moonshot-v1-8k
智谱 GLM
https://open.bigmodel.cn/api/paas/v4
glm-4-flash
OpenAI
https://api.openai.com/v1
gpt-4o-mini
本地 Ollama
http://localhost:11434/v1
qwen2.5:7b
百炼的模型 ID 每上线一版会变，以控制台「模型广场」里复制的为准，写错会报 model not found。
第二步：答题
打开作业页面，面板会显示识别到的平台名和题目数量：
- 点 一键作答 → 等 2~10 秒 → 显示「已填 N/N 题」。
- 核对一眼页面，然后自己点平台的提交 / 交卷按钮。
脚本只负责填，不负责提交。这是刻意设计：署名和成绩归属由你决定。
没有 API Key 也能用
1. 点 导出题目 —— 整份卷子进剪贴板。
2. 粘到任意对话式 AI（或直接粘给我），让它按 题号: 答案 返回。
3. 点 导入答案，把那段文字粘进去 → 填入。
日志会提示「已回填 N 题，缺：x,y」，缺的那几题自己补。
🧩 支持的页面
面板右上角的标签显示当前命中哪个 adapter：
adapter
识别方式
适用
头歌类实训关卡
存在 ul.choose-container > li
头歌 / Educoder 实训
超星类 · 试卷接口
路径含 /exercise/{id}，且 start.json 真能返回试卷
超星 / 学习通
通用页面（启发式）
页面上有 ≥2 组单选 / 多选，或有填空框
其他所有网站
显示 未识别 时：
1. 先等页面加载完，点 重新检测。
2. 仍不行就把设置里的「通用模式最少识别题数」改成 1 再试。
3. 还不行说明该站结构特殊，看下一节自己写 adapter。
🛠 自己写 adapter
核心（AI 调用、UI、配置、导入导出）与平台无关，接入新站点只要实现 4 个方法：
{
  id: 'my-site',
  name: '我的平台',

  // 1. 是否由我接管这个页面（可返回 Promise，比如探测接口）
  detect() {
    return !!document.querySelector('.question-item');
  },

  // 2. 抓题：返回数组，题号 n 从 1 开始
  async collect() {
    return [...document.querySelectorAll('.question-item')].map((el, i) => ({
      n: i + 1,
      kind: 'choice',          // 'choice' 或 'fill'
      type: 'single_choice',   // single_choice | multiple_choice | judgement | fill_blank
      title: el.querySelector('.stem').innerText.trim(),
      choices: [...el.querySelectorAll('label')].map((lab) => ({
        t: lab.innerText.trim(),   // 选项文字（用来给 AI 看）
        el: lab                    // 用来点：填好后调用 el.click()
      }))
    }));
  },

  // 3. 填答案：map 是 { 题号: 'A' }，返回 { ok, miss }
  async apply(qs, map) {
    let ok = 0; const miss = [];
    for (const q of qs) {
      const a = map[q.n];
      if (a == null || a === '') { miss.push(q.n); continue; }
      if (q.kind === 'fill') {
        // 填空题的输入框要在 collect 里自己存下来，比如 _el: el.querySelector('textarea')
        const ta = q._el;
        const set = Object.getOwnPropertyDescriptor(HTMLTextAreaElement.prototype, 'value').set;
        set.call(ta, a);
        ta.dispatchEvent(new Event('input', { bubbles: true }));
        ta.dispatchEvent(new Event('change', { bubbles: true }));
      } else {
        const idx = 'ABCDEFGHIJ'.indexOf(String(a).toUpperCase().trim());
        if (idx >= 0 && q.choices[idx]) q.choices[idx].el.click();
        else { miss.push(q.n); continue; }
      }
      ok++;
    }
    return { ok, miss };
  },

  // 4. 可选：页面上的提交按钮（不实现就不显示「提交」按钮）
  submit() { /* document.querySelector('#submit').click(); */ }
}
把它 ADAPTERS.push(...) 进脚本，顺序即优先级。
三个必须注意的坑
1. 不要按位置选答案。很多平台每次刷新都会打乱选项顺序，必须按选项文字内容定位。
2. React / Vue 的受控输入框，直接 el.value = x 不生效，必须用原型上的 value setter + 派发 input / change 事件。
3. 填空题的判分常是精确匹配，别自作多情地加句号、加空格、改大小写。
❓ 常见问题
点一键作答没反应，日志提示「还没填 API Key」
去设置里填 Key。用本地 Ollama 时，Base URL 里含 localhost / 127.0.0.1 可免 Key。
报 401
Key 错了或没生效（复制时带了空格、用了别的平台的 Key）。
报 404 / model not found
Base URL 多了或少了路径；模型名要照抄控制台，别照抄文档里的旧版本号。
报 400 且提到 response_format
部分服务不支持 JSON 模式。脚本会自动去掉 response_format 重试一次；两次都失败说明该接口不兼容，改用「导出题目 → 粘给 AI → 导入答案」这条路径。
填的位置不对 / 串题
多半是题干识别偏了。看「导出题目」的结果：如果题号和页面对不上，点「重新检测」；某些网站需要自己写 adapter。
答案是空 / 只填了一部分
日志会列出缺失题号。通常是该题题干里缺少空格、或者 AI 觉得信息不足。把题数拆分更小（每次打包题数调到 5）再试。
页面刷新后填的全没了
正常。多数平台只用前端状态保存草稿，脚本没有绕过服务端。填完尽快提交。
能不能帮我自动交卷
不能，脚本里没有这个动作。提交前请自己核对一遍。
🔒 隐私
- 脚本不含任何 API Key，apiKey 默认空字符串。
- Key 存在扩展的本地存储里（GM_setValue），只在请求你填的那个 AI 域名时作为 Authorization 头发出去。
- 题目内容会发给你配置的 AI 服务商——这一点和使用网页版 ChatGPT 粘贴题目是一样的，介意就用本地 Ollama。
- 不收集、不上报任何使用数据，没有第三方统计。
⚖️ 使用边界
这个项目的定位是前端自动化与 LLM 应用的练习。请只在你自己的账号、用于复习自测等正当场景；用于考试作弊、代刷他人作业等行为与作者无关，也可能违反你所在平台的规定甚至校规。
📄 License
MIT
