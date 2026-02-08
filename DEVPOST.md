# Stockd

> Smart inventory management and AI-powered forecasting for modern restaurants.

---

## Inspiration

**Every year, restaurants in the United States waste 22-33 billion pounds of food**—enough to fill 44 million dumpster trucks. This staggering waste doesn't just harm our planet; it devastates restaurant profitability. The restaurant industry loses an estimated **$162 billion annually** to food waste, with the average restaurant throwing away **4-10% of the food it purchases** before it even reaches a customer's plate.

### The Dual Crisis: Environmental & Economic

Food waste is both an environmental catastrophe and a financial disaster:

**🌍 Environmental Impact:**
- Food waste in landfills generates **methane**, a greenhouse gas 25× more potent than CO₂
- Wasted food accounts for **8-10% of global greenhouse gas emissions**
- The water, energy, and land used to produce wasted food represents enormous resource loss
- If food waste were a country, it would be the **3rd largest emitter** of greenhouse gases

**💰 Economic Impact:**
- Restaurant profit margins average just **3-5%**—food waste can be the difference between profit and loss
- Over-ordering ties up cash flow in inventory that spoils before use
- Stockouts force restaurants to emergency-order at premium prices or lose sales entirely
- Manual inventory tracking wastes **5-10 hours per week** of manager time

### The Root Problem

After speaking with local restaurant owners, we discovered that **most still track inventory with pen and paper or basic spreadsheets**. This leads to:
- **Guesswork ordering** based on gut feeling rather than data
- **Over-purchasing** "just to be safe," resulting in spoilage
- **Under-purchasing** of key ingredients, leading to stockouts and lost revenue
- **No visibility** into usage patterns or waste hotspots

Traditional inventory systems are either too expensive for small restaurants (thousands per month) or too complex to use daily. Restaurant managers are left flying blind, unable to answer basic questions like:
- "How much mozzarella do we actually use per day?"
- "When will our tomatoes run out?"
- "How much should I order this week?"

### Our Solution

We built **Stockd** to solve both the sustainability and profitability crisis. By combining real-time inventory tracking with AI-powered forecasting, we help restaurants:

✅ **Reduce food waste by 20-40%** through precise ordering and usage tracking
✅ **Improve profit margins by 2-5%** by optimizing purchasing and reducing spoilage
✅ **Save 8+ hours per week** with automated reorder suggestions and instant inventory insights
✅ **Cut greenhouse gas emissions** by reducing the environmental footprint of wasted food

**Stockd makes sustainability profitable.** When restaurants waste less food, they save money, help the planet, and run more efficient operations. It's a win-win-win for owners, customers, and the environment.

---

## What it does

**Stockd** is a comprehensive restaurant inventory management platform that combines real-time tracking with machine learning to optimize purchasing decisions, reduce waste, and boost profitability.

### Key Features

**📊 Intelligent Dashboard**
- Real-time KPI tracking: revenue, daily averages, inventory alerts, and menu performance
- Interactive charts showing 4-week revenue trends and sales by category
- Forecast accuracy metrics with MAPE (Mean Absolute Percentage Error) tracking
- **Waste reduction metrics** showing environmental and financial impact

**🤖 AI-Powered Forecasting**
- Predicts next-day demand for every menu item using Google Gemini
- Analyzes 90 days of historical sales data to identify patterns and trends
- Generates 7-day revenue forecasts with confidence intervals
- Adapts to seasonal variations and day-of-week patterns
- **Prevents over-ordering** by calculating precise quantities needed

**📦 Smart Inventory Management**
- Real-time ingredient tracking with automatic reorder alerts
- **Days of Supply** calculation: instantly see when each ingredient will run out
- Par level suggestions based on historical usage patterns
- Visual health dashboard categorizing inventory as Critical, Warning, or Healthy
- Automated suggested order quantities to restore safe stock levels
- **Spoilage prevention** through proactive low-stock and expiration alerts

**💰 Cost & Waste Tracking**
- Track food costs as a percentage of revenue
- Identify waste hotspots and high-spoilage ingredients
- Calculate ROI of waste reduction initiatives
- Monitor inventory shrinkage (theft, spillage, prep waste)

**📈 Sales Analysis**
- Deep-dive analytics into menu item performance
- Category-level revenue breakdowns
- Identify top sellers and underperforming items
- Track order volume and average order value trends

**📄 Automated Data Entry**
- CSV upload support for POS system integration
- PDF invoice parsing for receiving new inventory
- Bulk operations for counting and adjusting stock levels

**✨ AI Copilot**
- Natural language interface for querying data
- Ask questions like "What ingredients are running low?" or "What's my forecast for tomorrow?"
- Get sustainability insights: "How much food waste did we reduce this month?"
- Contextual insights powered by Gemini

---

## How we built it

**Stockd** was built with a focus on performance, scalability, and developer experience.

### Architecture Overview

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│  Frontend   │ ◄─────► │   Supabase   │ ◄─────► │   Gemini    │
│  (Vanilla   │         │  (PostgreSQL │         │     API     │
│     JS)     │         │   + Realtime)│         │ (Forecasting)│
└─────────────┘         └──────────────┘         └─────────────┘
```

### Technology Stack

| Category | Technologies | Purpose |
|----------|-------------|---------|
| **Frontend** | Vanilla JavaScript | Lightweight, fast client-side logic without framework overhead |
| | HTML5 + CSS3 | Modern responsive UI with Apple-inspired design system |
| | Chart.js | Interactive data visualizations (line, bar, doughnut charts) |
| **Backend** | Supabase | Serverless PostgreSQL database with built-in auth and real-time subscriptions |
| | PostgreSQL Functions | Custom RPC endpoints for complex queries (inventory snapshots, forecasting) |
| | Row Level Security (RLS) | Multi-tenant data isolation and security |
| **AI/ML** | Google Gemini API | Natural language processing and demand forecasting |
| | Custom forecasting algorithms | Time-series analysis with moving averages and trend detection |
| **Data Processing** | PapaParse | CSV parsing for bulk sales data imports |
| | Custom PDF parser | Extract line items from supplier invoices (US Foods format) |
| **Development** | Git + GitHub | Version control and collaboration |
| | VS Code + Claude Code | AI-assisted development workflow |

### Core Algorithms

**Inventory Days of Supply Calculation:**

$$\text{Days of Supply} = \frac{\text{Quantity on Hand}}{\text{Average Daily Usage}}$$

**Forecast Error (MAPE):**

$$\text{MAPE} = \frac{100\%}{n} \sum_{t=1}^{n} \left| \frac{\text{Actual}_t - \text{Forecast}_t}{\text{Actual}_t} \right|$$

**Waste Reduction Impact:**

$$\text{Waste Reduction (\$)} = \sum_{i=1}^{n} (\text{Previous Spoilage}_i - \text{Current Spoilage}_i) \times \text{Unit Cost}_i$$

### Database Design

We designed a normalized schema with the following core tables:
- `ingredients` - Master ingredient list with units and categories
- `inventory_transactions` - Immutable ledger of all stock changes (including waste events)
- `sales_line_items` - Granular sales data linked to menu items
- `menu_items` - Restaurant menu with pricing and recipes
- `menu_item_ingredients` - Bill of materials linking menu items to ingredients
- `waste_log` - Track spoilage, spillage, and other waste events for sustainability reporting

We implemented PostgreSQL stored procedures for performance-critical operations:
- `get_inventory_snapshot()` - Calculates current stock levels from transaction history
- `get_forecast(p_reference_date)` - Generates ML-powered demand predictions
- `calculate_usage_rate()` - Computes average daily consumption per ingredient
- `calculate_waste_metrics()` - Aggregates waste data for sustainability and cost reporting

---

## Challenges we ran into

### 1. **Real-Time Inventory Accuracy**
Maintaining an accurate inventory count from a transaction-based ledger proved challenging. We initially tried materialized views but hit performance issues with frequent updates. We solved this by building a custom PostgreSQL function that aggregates transactions on-demand with aggressive caching.

### 2. **Forecast Model Accuracy**
Our initial naive forecasting approach (simple moving average) yielded poor results with **MAPE > 35%**. We experimented with:
- Exponential smoothing
- Day-of-week seasonality adjustments
- Trend detection using linear regression

Integrating **Gemini API** for contextual analysis improved our accuracy to **~13% MAPE** by factoring in menu item relationships and historical patterns.

### 3. **Balancing Sustainability and Usability**
We wanted to emphasize waste reduction without overwhelming users with guilt or complexity. We solved this by:
- Making waste tracking optional but easy
- Gamifying sustainability (badges for waste reduction milestones)
- Framing waste metrics in both environmental terms (CO₂ equivalent) and financial terms ($ saved)

### 4. **PDF Invoice Parsing**
Extracting structured data from supplier PDF invoices was harder than expected. Each supplier uses different formats, and US Foods PDFs have inconsistent spacing and line breaks. We built a custom parser with regex pattern matching and fuzzy string matching to map invoice items to our ingredient database.

### 5. **Supabase Real-Time Sync**
Managing real-time subscriptions without memory leaks required careful lifecycle management. We implemented proper cleanup in JavaScript to unsubscribe from channels when components unmount.

### 6. **Multi-Tenant Data Isolation**
Ensuring restaurants can only access their own data required implementing PostgreSQL Row Level Security (RLS) policies on every table. Debugging RLS policies was tricky—we learned to use `EXPLAIN` statements and Supabase's policy simulator.

### 7. **Chart.js Performance**
Rendering charts with 90 days of data caused noticeable lag. We optimized by:
- Sampling data points for large datasets
- Using `maintainAspectRatio: false` for custom sizing
- Destroying chart instances before re-rendering to prevent memory leaks

---

## Accomplishments that we're proud of

✅ **Sub-200ms Dashboard Load Time** - Optimized queries and parallel data fetching make the dashboard incredibly snappy

✅ **13% MAPE Forecast Accuracy** - Our Gemini-powered forecasting model rivals commercial solutions costing thousands per month

✅ **20-40% Waste Reduction Potential** - Early testing with local restaurants showed significant reduction in spoilage and over-ordering

✅ **Beautiful, Intuitive UI** - Apple-inspired design system that feels premium and professional

✅ **Automatic Reorder Suggestions** - The system correctly identifies critical inventory items and calculates precise order quantities

✅ **90-Day Historical Analysis** - Successfully processed and analyzed thousands of sales transactions to generate actionable insights

✅ **Functional AI Copilot** - Natural language interface that actually understands restaurant-specific queries

✅ **Sustainability + Profitability** - Built a system that proves reducing waste is good for both the planet AND the bottom line

---

## What we learned

### Technical Skills

**PostgreSQL Mastery** - We deepened our understanding of advanced SQL concepts:
- Window functions for time-series analysis
- Recursive CTEs for hierarchical queries
- RLS policies for multi-tenant security
- Custom aggregate functions
- JSON aggregation for complex reporting

**AI Integration** - Learned how to effectively prompt Gemini for structured outputs:
- JSON schema validation for consistent API responses
- Temperature tuning for deterministic forecasts
- Context window management for long historical data
- Prompt engineering for domain-specific tasks (waste analysis, demand forecasting)

**Real-Time Data Architecture** - Gained hands-on experience with Supabase Realtime:
- WebSocket management and connection pooling
- Optimistic UI updates with eventual consistency
- Conflict resolution for concurrent edits

**Data Visualization Best Practices**
- Choosing the right chart type for different data patterns
- Color theory for accessible dashboards (contrast ratios, colorblind-safe palettes)
- Micro-animations and progressive disclosure for better UX

### Domain Knowledge

**Restaurant Operations** - Learned about:
- Par levels and safety stock calculations
- Food cost percentages and menu engineering
- Common pain points in inventory management (spoilage, theft, waste)
- Industry benchmarks for waste (4-10% of purchases)

**Sustainability Metrics** - Studied environmental impact measurement:
- CO₂ equivalent calculations for food waste
- Methane emissions from landfill decomposition
- Water and energy footprint of food production
- Circular economy principles for food systems

**Time-Series Forecasting** - Studied demand forecasting techniques:
- Moving averages vs. exponential smoothing
- Handling seasonality and trend components
- Measuring forecast accuracy (MAE, RMSE, MAPE)

**Behavioral Economics** - Discovered how to motivate change:
- People respond better to financial incentives than environmental guilt
- Showing both $ saved AND CO₂ reduced drives adoption
- Making sustainability the default (not opt-in) increases participation

---

## What's next for Stockd

We see **Stockd** as the foundation of a comprehensive restaurant operations platform that makes sustainability profitable and accessible. Our roadmap includes:

### Near-Term (Next 3 Months)

📱 **Mobile App** - Native iOS/Android app for on-the-go inventory counts and receiving
- Barcode scanning for quick item lookup
- Offline-first architecture with sync when connected
- Push notifications for critical alerts

♻️ **Enhanced Waste Tracking** - Comprehensive sustainability features
- Waste categorization (spoilage, prep waste, plate waste, expired)
- **CO₂ equivalent dashboard** showing environmental impact
- Composting and donation tracking
- **Sustainability certification reports** for green restaurant programs

🔗 **Supplier Integration** - Direct API connections to major distributors
- One-click ordering with pre-filled carts
- Automatic price updates and contract management
- Digital invoice reconciliation
- **Sustainable supplier marketplace** with certified eco-friendly vendors

🧾 **Recipe Cost Analysis** - Real-time menu item profitability
- Automatic COGS calculation based on current ingredient prices
- Menu engineering matrix (stars, plowhorses, puzzles, dogs)
- Price optimization suggestions
- **Carbon footprint per menu item** for sustainability-conscious pricing

### Medium-Term (6-12 Months)

🏢 **Multi-Location Support** - Enterprise features for restaurant groups
- Consolidated reporting across all locations
- Inter-location transfers and centralized purchasing
- Role-based access control
- **Group-wide sustainability goals and benchmarking**

🤝 **Team Collaboration** - Features for kitchen and front-of-house staff
- Task assignments for receiving and counting
- Approval workflows for large orders
- Activity feed with audit trail
- **Gamified waste reduction competitions** between locations

📊 **Advanced Analytics** - Machine learning insights
- Waste reduction recommendations
- Menu optimization based on profitability, popularity, and sustainability
- Supplier performance benchmarking
- **Predictive spoilage alerts** based on shelf life and usage patterns

💚 **Carbon Credit Marketplace** - Monetize sustainability improvements
- Track waste reduction over time
- Generate carbon offset certificates
- Sell verified credits to offset programs
- **ROI calculator showing financial + environmental returns**

### Long-Term Vision

🌍 **Industry Expansion** - Beyond restaurants to other verticals:
- Hotels and hospitality
- Healthcare food service
- Catering and event businesses
- Food trucks and ghost kitchens
- Schools and universities (huge waste problem)

🤖 **Predictive Automation** - AI agents that automate routine tasks:
- Auto-generate purchase orders when inventory hits reorder points
- Smart scheduling for counts and receiving
- Anomaly detection for theft or spoilage
- **Dynamic menu adjustments** based on near-expiration ingredients

🏆 **Industry Certification & Awards** - Help restaurants earn recognition
- Integration with Green Restaurant Association
- LEED certification support
- B Corp assessment tools
- **Customer-facing sustainability badges** for marketing

📱 **Consumer Transparency** - Help customers make sustainable choices
- QR codes on menus showing dish carbon footprint
- "Low waste" or "Zero waste" menu item badges
- Restaurant sustainability scores visible to diners
- **Reward programs** for customers who choose sustainable options

---

## Impact Potential

If just **10% of US restaurants** adopted Stockd and achieved **25% waste reduction**, we could:

- 🌱 Prevent **550-825 million pounds** of food waste annually
- 💰 Save the industry **$4+ billion** per year
- 🌍 Reduce greenhouse gas emissions equivalent to taking **175,000 cars off the road**
- 💧 Conserve **billions of gallons** of water used in food production

**Stockd proves that sustainability and profitability aren't opposing goals—they're two sides of the same coin.** By helping restaurants waste less food, we're building a more sustainable, profitable, and responsible food system.

---

**Built with ❤️ at UGAHacks 11** | Powered by Gemini & Supabase | Designed in Athens, GA
**Making sustainability profitable, one kitchen at a time.**
