# Sage Stocks

*Last updated: November 23, 2025*

**AI that never guesses, never hallucinates, and never lets emotion cloud judgment.**

Genius-level financial intelligence that simultaneously analyzes the fundamentals, technicals, market regime, and sector dynamics of individual stocks in the context of the global market.

**Version:** v0.1.0 (User) / v1.2.21 (Dev)

**Status:** Production-ready with market context integration, delta-first analysis, and event calendar

**Author:** Shalom Ormsby

---

## Overview

Sage Stocks is a serverless stock analysis platform built on a **two-layer architecture**:

1. **The math layer:** Deterministic calculations based on professional-grade financial metrics
2. **The interpretation layer:** AI analysis that applies proven investment frameworks (Buffett, Dalio, Lynch) systematically and unemotionally

**Design Philosophy:** *Impeccable but simple.* Built for daily stock analyses, not enterprise scale. Clean architecture, production-grade code, minimal complexity.

### What It Does

- **Analyzes stocks in 18-25 seconds** using real-time market data (FMP + FRED APIs)
- **Market context integration** - Analyzes market regime (Risk-On/Risk-Off/Transition) before individual stock analysis
- **Delta-first analysis** - Shows what changed since last analysis, with 90-day historical tracking
- **Event-aware** - Tracks earnings calls, dividends, splits, and guidance for portfolio stocks
- **Generates composite scores** (1.0-5.0 scale) across 6 dimensions + market alignment
- **Creates AI-generated analysis** with regime-aware narratives and historical context
- **Syncs to Notion databases** for analysis storage and review → PostgreSQL migration planned (v2.0)
- **Multi-tenant architecture** with per-user database isolation and OAuth authentication
- **Enforces rate limits** (10 analyses/user/day) with intelligent bypass system

---

## Architecture

### Technology Stack

**Backend:**

- **Platform:** Vercel Serverless Functions (300s timeout, Node.js 18+)
- **Language:** TypeScript 5.3+
- **Runtime:** Node.js serverless environment

**Data Sources:**

- **Financial Modeling Prep (FMP)** - Stock data, fundamentals, technical indicators ($22-29/month)
- **FRED API** - Macroeconomic data (yield curve, VIX, unemployment) (free)
- **Upstash Redis** - Distributed rate limiting state (REST API, serverless-native)

**LLM Integration:**

- **Provider-agnostic abstraction layer** supporting:
    - Google Gemini (Flash 2.5, Flash 1.5) - Primary, $0.013/analysis
    - OpenAI (GPT-4 Turbo, GPT-3.5 Turbo)
    - Anthropic (Claude 3.5 Sonnet, Claude 3 Haiku)
- **Configurable via environment variable** (`LLM_PROVIDER`)
- **67% token reduction** vs. original prompts (6,000 → 2,000 tokens)

**Integration:**

- **Notion API** - Database operations (v1.2.21, transitioning to PostgreSQL in v2.0)
- **REST APIs** - All external communication via HTTP

### API Endpoints

| Endpoint | Method | Description | Timeout |
| --- | --- | --- | --- |
| `/api/health` | GET | Health check (uptime, version) | 10s |
| `/api/analyze` | POST | Stock analysis (full workflow) | 300s |
| `/api/webhook` | POST | Notion webhook handler | 60s |
| `/api/bypass` | GET/POST | Activate bypass code session | 10s |
| `/api/usage` | GET | Check rate limit usage | 10s |
| `/api/api-status` | GET | API monitoring dashboard | 30s |

### Data Flow (v1.2.21)

```
User (Web Interface)
    ↓ POST /api/analyze {ticker, userId}
Vercel Serverless Function
    ↓ Check rate limit (Redis)
    ↓ [Step 0] Fetch market context (cached, <100ms or 3-5s if cache miss)
        • Market regime classification (Risk-On/Risk-Off/Transition)
        • Sector rotation analysis (11 sector ETFs)
        • Economic indicators (VIX, Fed Funds, Unemployment, Yield Curve)
    ↓ Fetch stock data (FMP + FRED, 3-5s)
    ↓ Calculate scores including market alignment (1s)
    ↓ Query 90-day historical analyses (Notion, 2-5s)
    ↓ Compute deltas with regime context (<1s)
    ↓ Generate delta-first LLM analysis (Gemini, 10-20s)
    ↓ Write to Notion (3 operations, 10-15s)
        • Update Stock Analyses page (metrics + market regime)
        • Create child analysis page (AI content)
        • Archive to Stock History (with market regime)
    ↓ Return {pageUrl, scores, metadata}
User
    → Opens Notion analysis page
```

**Total Latency:** 30-45 seconds (under 300s Vercel Pro timeout)

---

## Key Features

### Multi-Dimensional Analysis

Composite score (1.0-5.0) calculated from weighted categories:

| Category | Weight | Metrics Included |
| --- | --- | --- |
| **Technical** | 28.5% | RSI, MACD, Bollinger Bands, SMA crossovers, volume trends |
| **Fundamental** | 33% | P/E ratio, EPS growth, revenue growth, profit margins, ROE |
| **Macro** | 19% | Market regime, sector rotation, yield curve, VIX, unemployment |
| **Risk** | 14.5% | Beta, volatility, drawdown, correlation to market |
| **Market Alignment** | 5% | Regime fit (beta vs Risk-On/Off), sector leadership, VIX context |
| **Sentiment** | 0% | Calculated score (1.0-5.0) displayed for reference, not included in composite |
| **Sector** | - | Relative sector performance vs. S&P 500 |

**Recommendations:**

- Strong Buy (4.0+)
- Buy (3.5-3.99)
- Moderate Buy (3.0-3.49)
- Hold (2.5-2.99)
- Moderate Sell (2.0-2.49)
- Sell (1.5-1.99)
- Strong Sell (<1.5)

### AI-Generated Analysis

7-section narrative generated by Google Gemini Flash 2.5:

1. **Executive Summary** - Buy/Hold/Sell recommendation with confidence level
2. **Technical Analysis** - Chart patterns, momentum indicators, support/resistance
3. **Fundamental Analysis** - Valuation metrics, earnings quality, growth prospects
4. **Risk Assessment** - Downside risks, volatility analysis, worst-case scenarios
5. **Macro Context** - Economic headwinds/tailwinds, sector trends, market regime
6. **Historical Trends** - Score deltas vs. previous analyses, trend direction
7. **Action Items** - Specific recommendations (e.g., "Wait for RSI < 30 before entry")

**Token Optimization:**

- Original prompts: 6,000 tokens → 30s generation time
- Optimized prompts: 2,000 tokens → 10-15s generation time
- **67% reduction** in tokens, **50% reduction** in latency

### Rate Limiting & Access Control

**User-level quotas:**

- 10 analyses per user per day (resets at midnight UTC)
- Tracked in Upstash Redis (distributed state)
- Graceful degradation (fails open if Redis unavailable)

**Bypass code system:**

- Session-based bypass (one-time code entry)
- Unlimited analyses until midnight UTC
- Stored in Redis with TTL expiry
- Admin bypass via environment variable

**Endpoints:**

- `GET /api/usage` - Check remaining quota (non-consuming)
- `POST /api/bypass` - Activate bypass code session
- `GET /api/bypass?code=XXX` - URL parameter activation

---

## Project Status

**Current Version:** v0.1.0 (User) / v1.2.21 (Dev)

**Completed (v1.0-1.4):**

- ✅ TypeScript/Vercel serverless architecture (~14,800 LOC)
- ✅ Multi-factor scoring engine with market alignment (7 dimensions)
- ✅ FMP + FRED API integration (23 calls for market context, 17 per stock)
- ✅ Market context integration (v1.3.0) - Risk-On/Risk-Off regime classification
- ✅ Delta-first analysis engine (v1.4.0) - 90-day historical tracking
- ✅ Event calendar integration (v1.2.16) - Earnings, dividends, splits, guidance
- ✅ Multi-tenant OAuth authentication with per-user database isolation
- ✅ Rate limiting system (Upstash Redis, timezone-aware)
- ✅ LLM abstraction layer (Google Gemini, OpenAI, Anthropic)
- ✅ Notion database integration (read/write optimizations, sequential deletion)
- ✅ Performance optimizations (67% token reduction, 50% faster execution)

**Current Focus (v1.5-2.0):**

- 🔧 Onboarding robustness (auto-detection fixes, validation surface)
- 🔧 Hallucination prevention system (validation + weekly audits)
- 🔧 Market Context bug fixes (content routing, property population)
- 📋 Production-ready Notion template with examples
- 📋 Event-aware analysis prompts (surface upcoming events in AI context)

**Planned (v2.0+):**

- 📋 Next.js frontend application (responsive, mobile-first)
- 📋 PostgreSQL migration (Supabase, 10-15x faster than Notion)
- 📋 Trend charts (Recharts, score history over time)
- 📋 Portfolio tracking and watchlists

See [ROADMAP.md](ROADMAP.md) and [CHANGELOG.md](CHANGELOG.md) for detailed history.

---

## Cost Structure

**Monthly Operating Costs (v1.2.21):**

| Service | Cost | Usage |
| --- | --- | --- |
| Vercel Pro | $20/month | 300s timeout, unlimited invocations |
| FMP API | $22-29/month | Stock data, fundamentals, technical indicators |
| Google Gemini | $40/month | 3,000 analyses (@ $0.013 each) |
| FRED API | Free | Macroeconomic data |
| Notion | Free | Database storage (v1.2.21 only) |
| Upstash Redis | Free | Rate limiting state (under free tier limits) |
| **Total** | **$82-89/month** | For personal use (up to 3,000 analyses/month) |

**v2.0 Upgrade (PostgreSQL):**

- Add Supabase: $0-25/month (free tier → Pro as needed)
- Total: $102-134/month (10-15x faster database, unlimited scale)

---

## License

**Business Source License 1.1**

### You CAN:

- ✅ Use for personal, educational, and non-commercial purposes
- ✅ View and study the source code
- ✅ Modify for your own personal use
- ✅ Fork on GitHub for non-commercial projects

### You CANNOT:

- ❌ Provide commercial stock analysis services
- ❌ Sell this software or derivative works
- ❌ Compete with the original author's offerings

### Change Date: October 23, 2029

After this date, the software becomes available under the **MIT License** (fully open source).

**Commercial licensing:** Contact [shalom.ormsby@gmail.com](mailto:shalom.ormsby@gmail.com)

See [LICENSE](LICENSE) file for full terms.

---

## Support & Contact

**Author:** Shalom Ormsby

**Email:** [shalom.ormsby@gmail.com](mailto:shalom.ormsby@gmail.com)

**Repository:** [github.com/shalomormsby/sagestocks](https://github.com/shalomormsby/sagestocks)

**For issues:**

- Check documentation in [docs/](docs/) folder first
- Review [CHANGELOG.md](CHANGELOG.md) for recent changes
- Open an issue on GitHub (when repository is public)

**For bugs:**

- Include: ticker, error message, expected vs. actual behavior
- Check [ROADMAP.md](ROADMAP.md) to see if issue is already known
- Provide Vercel function logs if possible

---

## Contributing

This is currently a personal project, but contributions are welcome for:

- Bug reports and fixes
- Documentation improvements
- Performance optimizations
- Feature suggestions (see [ROADMAP.md](ROADMAP.md) first)

**Before contributing:**

1. Read [ARCHITECTURE.md](ARCHITECTURE.md) to understand system design
2. Review [.github/FILE_ORGANIZATION.md](.github/FILE_ORGANIZATION.md) for file organization standards
3. Check [ROADMAP.md](ROADMAP.md) to avoid duplicating planned work

---

## Acknowledgments

Built with:

- Vercel - Serverless platform
- Financial Modeling Prep - Market data
- FRED - Economic data
- Google Gemini - LLM analysis generation
- Notion - Database integration (v1.2.21)
- Upstash - Redis rate limiting

---

*For legacy v0.2.x Python documentation, see [docs/legacy/README_v0.2.x.md](docs/legacy/README_v0.2.x.md)*
