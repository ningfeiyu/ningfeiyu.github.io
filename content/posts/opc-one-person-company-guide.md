---
title: "OPC 一人公司实操指南：从 0 到 1 搭起你的自动化生意"
date: 2026-06-11
draft: false
---

![One Person Company](https://images.unsplash.com/photo-1551836022-d5d88e9218df?w=1200&h=600&fit=crop)

**OPC**（One Person Company，一人公司）指个人借助 AI、自动化工具与云服务，独立完成从产品设计、开发、运营到客户服务的全链路业务。本文给出一份**可立即上手**的搭建手册，覆盖技术栈选型、自动化流水线、收款合规与持续增长。

---

## 一、技术栈总览

![Tech Stack](https://images.unsplash.com/photo-1518770660439-4636190af475?w=1200&h=600&fit=crop)

### 1.1 推荐组合

```yaml
前端: Astro / Next.js 14 (静态站 + Edge Functions)
后端: Cloudflare Workers / Vercel Functions
数据库: Supabase (Postgres + Auth + Storage)
支付: Stripe (订阅) + Creem (国内)
AI: Hermes Agent + OpenAI / Anthropic / DeepSeek
部署: GitHub Actions + Vercel / Cloudflare Pages
监控: Sentry + Plausible (隐私友好统计)
沟通: Telegram Bot + Discord Webhook
```

### 1.2 为什么选这套？

| 模块 | 选择 | 核心理由 |
|------|------|---------|
| **Astro** | 静态站首选 | 零 JS 默认，SEO 满分，构建极快 |
| **Cloudflare** | 边缘部署 | 免费额度大，国内可达性强 |
| **Supabase** | 开源 BaaS | Postgres 真数据库，比 Firebase 更可控 |
| **Stripe** | 全球支付 | Developer 体验最好的支付平台 |
| **Hermes Agent** | AI 编排 | 跨会话记忆、技能系统、24h 自动化 |

---

## 二、域名 + 邮箱

![Domain Setup](https://images.unsplash.com/photo-1568952433726-3896e3881c65?w=1200&h=600&fit=crop)

### 2.1 域名采购
- **国外**：Cloudflare Registrar（成本价，无溢价）
- **国内**：阿里云 / 腾讯云（需备案）

### 2.2 一人公司邮箱

推荐 **Cloudflare Email Routing** 或 **Resend**：

```bash
# Cloudflare Email Routing（免费）
# 1. 添加域名 → Email → Email Routing → Enable
# 2. 添加 catch-all 路由到你的 Gmail / Outlook
# 3. 自动得到 hello@yourdomain.com、support@yourdomain.com
```

---

## 三、产品构建 30 天路线

![Roadmap](https://images.unsplash.com/photo-1454165804606-c3d57bc86b40?w=1200&h=600&fit=crop)

### Week 1：选赛道 + MVP

**高利润低门槛赛道参考：**

```yaml
SaaS 工具: 
  - SEO 内容生成器 ($9-49/月)
  - 数据抓取 / API 包装站 ($0.01/调用)
  - 小红书 / Twitter 自动化工具

内容产品:
  - 行业报告订阅 ($49-299/月)
  - 模板 / Prompt 市场 (一次性 $5-99)
  - 付费 Newsletter (Substack 模式)

数字资产:
  - Notion 模板 ($19-49)
  - Figma 插件
  - Chrome 扩展
```

**MVP 原则：**
1. **3 天内做出能付款的版本**——再丑也能用
2. **一个人能维护**——不加任何 6 个月后自己看不懂的功能
3. **Stripe 测试模式先跑通**——别等"完美"再上线

### Week 2-3：搭建 + 自动化

```bash
# 示例：5 分钟搭起 Stripe 订阅
npm create astro@latest -- --template minimal
cd my-saas

# 安装 Stripe
npm install stripe @stripe/stripe-js

# Astro 配置
cat >> astro.config.mjs << 'EOF'
import { defineConfig } from 'astro/config';
import node from '@astrojs/node';

export default defineConfig({
  output: 'server',
  adapter: node({ mode: 'standalone' }),
});
EOF
```

**升级套餐推荐：**
- [Shipfa.st](https://shipfa.st) - Next.js + Supabase + Stripe 全栈模板，$199 一次性
- [Indie Starter](https://indieapp.dev) - Astro 全栈 SaaS 模板，$99

### Week 4：上线 + 首批 10 个客户

发到这三个地方：
1. **Product Hunt**（海外用户）
2. **V2EX / 即刻 / 少数派**（中文用户）
3. **你的 Twitter / X 时间线**（个人品牌）

---

## 四、自动化流水线（核心）

![Automation](https://images.unsplash.com/photo-1518432031352-d6fc5c10da5a?w=1200&h=600&fit=crop)

### 4.1 客服自动化

```python
# Telegram 机器人自动回复（5 行代码）
import telebot, openai

bot = telebot.TeleBot("YOUR_TOKEN")

@bot.message_handler(func=lambda m: True)
def reply(msg):
    r = openai.ChatCompletion.create(
        model="gpt-4o-mini",
        messages=[{"role":"user","content":msg.text}]
    )
    bot.reply_to(msg, r.choices[0].message.content)

bot.polling()
```

### 4.2 营销自动化

**用 Hermes Agent 的 cronjob 每天自动产出内容：**

```bash
# 早 8 点 - 生成今日推文
0 8 * * * /usr/bin/hermes run "\
基于昨日用户痛点，写一条 280 字内的 Twitter/X 推文，\
风格参考 @levelsio，发到 Telegram channel"

# 中午 12 点 - 周报自动化
0 12 * * 1 /usr/bin/hermes run "\
抓取 Stripe 本周新增订阅，生成 3 段中文摘要,\
推送 Telegram: /sales-weekly"
```

### 4.3 部署自动化（GitHub Actions）

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pages: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - uses: withastro/action@v3
      - uses: actions/configure-pages@v4
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./dist
      - id: deployment
        uses: actions/deploy-pages@v4
```

---

## 五、收款与税务

![Payments](https://images.unsplash.com/photo-1554224155-6726b3ff858f?w=1200&h=600&fit=crop)

### 5.1 全球收款

| 渠道 | 适用场景 | 费率 |
|------|---------|------|
| **Stripe** | 海外订阅 / SaaS | 2.9% + $0.3 |
| **Paddle** | 含税务代征 | 5% + $0.5 |
| **Lemon Squeezy** | 数字产品 | 5% + $0.5 |
| **Creem** | 国内出海 | 2.9% + $0.3 |
| **支付宝 / 微信** | 国内个人收款 | 1% |

### 5.2 国内合规

**个人独资企业 / 个体工商户**是 OPC 最合适的主体：

```yaml
路径:
  1. 工商注册 (1-3 天, 在线办理)
     - 各省政务网 / 随申办 / 粤省事
     - 经营范围写: 技术服务、软件开发、信息咨询
  
  2. 税务登记
     - 月收入 10 万以下: 小规模纳税人免增值税
     - 季度 30 万以下: 全免
     - 个税: 5%-35% 累进
  
  3. 对公账户 (可选)
     - 微信/支付宝商户号即可起步
     - 后期: 工行 / 招行 数字账户免费开

费用估算: 0 注册金 + 200-500/年 代理记账
```

### 5.3 季度报税模板

```
季度收入: _____ 元
季度成本: _____ 元 (服务器 + 服务采购 + 差旅)
应税所得: _____ 元
应缴个税: _____ 元 (5%-35%)
申报截止: 次月 15 日
```

---

## 六、运营增长矩阵

![Growth](https://images.unsplash.com/photo-1460925895917-afdab827c52f?w=1200&h=600&fit=crop)

### 6.1 获客渠道 ROI 排序

```yaml
1. SEO 长期内容 (6-12 月见效, LTV 高)
   - Hugo / Astro 静态站 + 长尾关键词布局
   - 每月 4-8 篇深度文章

2. Twitter/X 个人品牌 (即时反馈, 跨境友好)
   - 每天 3-5 条
   - build in public 路线

3. Product Hunt + Hacker News (爆发式)
   - 1 周 1 个新功能发布
   - PH 发布是单点战斗, 团队节奏

4. 小红书 / 抖音 / VideoStan (国内流量)
   - 1 周 3 短视频 + 1 直播
   - 工具型产品天然友好

5. 联盟营销 (被动增长)
   - 30% 分成是行业标准
   - Refersion / FirstPromoter 一键接入
```

### 6.2 留存指标公式

```python
# 健康的一人公司指标
LTV = ARPU × 平均订阅月数 (目标 > CAC × 3)
CAC = 单客获取成本 (广告 + 内容投入)
NRR = (期初 + 升级) / 期初 (目标 > 110% 月)
Churn = 月流失率 (目标 < 5% / 月)

# 例: ARPU $29, 平均 8 月
LTV = $232; 如 CAC < $77 则健康
```

---

## 七、避坑清单

![Pitfalls](https://images.unsplash.com/photo-1515886657613-9f3515b0c78f?w=1200&h=600&fit=crop)

**4 个致命坑：**

1. **完美主义**——等"完美"再上线，等一年还在改 UI
2. **多产品并行**——3 个 70% 完成的产品 < 1 个 100% 完美
3. **不做营销**——"产品会说话"在 2026 年已不成立
4. **拒绝自动化**——一个人的边际收益来自**解放自己的时间**

**3 个隐性优势：**

1. **决策成本 = 0**：不需对齐，开会即写代码
2. **金钱成本极低**：Vue / Astro + Supabase Free + Workers Free = 月成本 < $5
3. **失败安全网**：个人独资企业债务隔离，破产不影响个人

---

## 八、收入里程碑参考

```yaml
阶段 1 - 验证 ($0-1k MRR):
  - 3-6 个月: 产品上线 + 首批 20 个付费用户
  
阶段 2 - 增长 ($1k-5k MRR):
  - 6-12 个月: SEO + 内容 + 联盟
  
阶段 3 - 规模 ($5k-20k MRR):
  - 12-18 个月: 付费广告 + 团队 (1-2 人兼职)
  
阶段 4 - 出海 / 收购 ($20k+ MRR):
  - 18-36 个月: 
    - 年收入 $240k+: 申请 O-1 / EB-1 签证 (美国)
    - 年收入 $500k+: 考虑出售 (Acquire.com / Quiet)
```

---

## 九、立即开始的 7 步

![Action](https://images.unsplash.com/photo-1484480974693-6ca0a78fb36b?w=1200&h=600&fit=crop)

1. **今天**：注册域名，搭起 Astro 静态站
2. **明天**：选 1 个赛道，画 5 页功能清单
3. **第 3 天**：用 Shipfa.st / Indie Starter 套模板改
4. **第 7 天**：Stripe 测试模式接入，定价表上线
5. **第 14 天**：Product Hunt 文案、Twitter 账号、产品视频准备
6. **第 21 天**：正式发布，做 100 个用户的 launch
7. **第 30 天**：复盘数据，进化到 v2

---

## 十、推荐资源

**模板工具：**
- [Shipfa.st](https://shipfa.st) - Next.js SaaS 全栈
- [Indie Starter](https://indieapp.dev) - Astro SaaS
- [Tailscan](https://tailscan.com) - Tailwind UI 套件
- [Ship Your SaaS](https://shipyoursaas.fastlane.dev) - Ruby on Rails 全套

**学习路径：**
- [Indie Hackers](https://indiehackers.com) - 一人公司社群
- [MicroConf](https://microconf.com) - 实战会议
- [Tyler Tringas 的 Storemy](https://storemy.com) - 出售教程
- [Arvid Kahl](https://thebootstrappedfounder.com) - 播客 + 书《The Bootstrapper's Manifesto》

**财务工具：**
- [Mercury](https://mercury.com) - 美国对公账户（远程开户）
- [Wise](https://wise.com) - 多币种账户
- [Plaid + Stripe Atlas](https://stripe.com/atlas) - 美国公司注册 + 银行 + 税务

---

## 写在最后

**OPC 不是"打工人 2.0"，是一种"杠杆生活方式"。**

AI 让 1 人的边际生产力 ×10，Cloud 让 1 人的基础设施成本 ÷10，社交媒体让 1 人的营销触达 ×1000。

**2026 年是一个人公司的"黄金 10 年"**——现在入场，你仍在最早期。

> "The best time to start was yesterday. The second best time is now."

---

**下一步**：把这份清单复制到你的 Notion / Obsidian，建一个 `OPC` 项目，选一条赛道，今天就写下第一行产品文档。

![Final](https://images.unsplash.com/photo-1499951360447-b19be8fe80f5?w=1200&h=600&fit=crop)

*愿你不只是用户，而是建造者——一人即公司。*