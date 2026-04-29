// ポートフォリオ追記
if (url.pathname === '/add-portfolio') {
  const body = await request.json();

  // AIで整形
  const aiRes = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': ANTHROPIC_KEY,
      'anthropic-version': '2023-06-01'
    },
    body: JSON.stringify({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1500,
      messages: [{ role: 'user', content: `以下の情報をもとに、ポートフォリオ用の作品紹介文をJSONのみで返してください（\`\`\`不要）。
作品名：${body.name}
URL：${body.url}
背景・課題：${body.background}
解決策：${body.solution}
使用技術：${body.tech}
工夫したポイント：${body.points}
成果：${body.result}

{"summary":"作品の概要1文","background_text":"背景・課題を整理した文章（箇条書き・「・」始まり）","solution_text":"解決策の説明（2〜3文）","tech_list":"使用技術（箇条書き・「・」始まり）","points_text":"工夫したポイント（箇条書き・「・」始まり）","result_text":"成果（箇条書き・「・」始まり）"}` }]
    })
  });
  const aiData = await aiRes.json();
  const rawText = aiData.content?.find(b => b.type === 'text')?.text || '{}';
  let formatted;
  try { formatted = JSON.parse(rawText.replace(/```json|```/g, '').trim()); }
  catch(e) { formatted = { summary: body.solution, background_text: body.background, solution_text: body.solution, tech_list: body.tech, points_text: body.points, result_text: body.result }; }

  // Notionに追記
  const content = `---

### ${body.name}

**🔗 URL**
${body.url || '（なし）'}

**📌 概要**
${formatted.summary}

**📋 作った背景・課題**
${formatted.background_text}

**💡 解決策**
${formatted.solution_text}

**🔧 使用技術**
${formatted.tech_list}

**✨ 工夫したポイント**
${formatted.points_text}

**📊 成果**
${formatted.result_text}

`;

  const notionRes = await fetch(
    `https://api.notion.com/v1/blocks/${PORTFOLIO_PAGE_ID}/children`,
    {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${NOTION_TOKEN}`,
        'Notion-Version': '2022-06-28',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        children: [{
          object: 'block',
          type: 'paragraph',
          paragraph: {
            rich_text: [{ type: 'text', text: { content: content } }]
          }
        }]
      })
    }
  );
  const notionData = await notionRes.json();
  return new Response(JSON.stringify({ success: true, notion: notionData }), {
    headers: { 'Content-Type': 'application/json', ...CORS }
  });
}
