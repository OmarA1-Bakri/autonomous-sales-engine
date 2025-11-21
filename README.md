# BMAD Sales Automation Workflows

Production-ready BMAD workflows for autonomous sales prospecting and outreach.

## 📁 Structure

```
bmad-workflows/
├── src/
│   └── modules/
│       └── sales/
│           ├── agents/               # 4 specialized sales agents
│           │   ├── sales-strategist.agent.yaml
│           │   ├── engagement-analyst.agent.yaml
│           │   ├── conversation-strategist.agent.yaml
│           │   └── outreach-orchestrator.agent.yaml
│           └── workflows/            # 3 core workflows
│               ├── prospect-discovery.workflow.yaml
│               ├── dynamic-outreach.workflow.yaml
│               └── re-engagement.workflow.yaml
└── README.md
```

## 🤖 Sales Agents

### 1. Sales Strategist
**Role**: Strategic planning and ICP definition

**Responsibilities**:
- Define ideal customer profiles
- Analyze market segments
- Set qualification thresholds
- Recommend campaign strategies
- Competitive intelligence

**When to use**:
- Starting new campaigns
- Poor campaign performance
- Entering new markets
- ICP refinement needed

### 2. Engagement Analyst
**Role**: Prospect behavior analysis and signal detection

**Responsibilities**:
- Analyze email engagement (opens, clicks)
- Classify reply sentiment
- Detect buying signals
- Identify objections
- Score urgency and intent

**When to use**:
- Processing prospect replies
- Analyzing engagement patterns
- Detecting high-intent prospects
- Identifying objections

### 3. Conversation Strategist
**Role**: Dynamic messaging and response crafting

**Responsibilities**:
- Craft personalized responses
- Handle objections strategically
- Design email sequences
- Optimize timing and copy
- Determine escalation points

**When to use**:
- Responding to prospect behavior
- Handling objections
- Creating campaign messaging
- Optimizing sequences

### 4. Outreach Orchestrator
**Role**: Multi-channel campaign execution

**Responsibilities**:
- Execute email campaigns (Lemlist)
- Schedule LinkedIn touches
- Optimize send times
- Track engagement across channels
- Monitor deliverability
- Enforce rate limits

**When to use**:
- Sending campaigns
- Tracking engagement
- Managing deliverability
- Coordinating multi-channel outreach

## 📋 Workflows

### 1. Prospect Discovery (`prospect-discovery.workflow.yaml`)

**Purpose**: Full pipeline from ICP definition to campaign-ready prospects

**Phases**:
1. **Discover**: Define ICP, search companies
2. **Enrich**: Find contacts, enrich data
3. **Qualify**: Score ICP fit, segment prospects
4. **Prepare**: Create campaigns, sync to CRM

**Use cases**:
- Starting new outbound campaigns
- Entering new markets
- Building prospect lists
- Account-based targeting

**Slash commands**:
```
/discover-prospects          # General discovery
/discover-uk-saas           # Pre-configured for UK SaaS
```

**Inputs**:
- Industry targets (e.g., "SaaS", "FinTech")
- Geography (e.g., "United Kingdom")
- Company size range (e.g., "50-500 employees")
- Revenue range (e.g., "$5M-$50M")
- Target titles (e.g., "CTO", "VP Engineering")

**Outputs**:
- Auto-approved prospect list (ICP score >= 85)
- Review queue (score 70-84)
- Lemlist campaign created
- HubSpot contacts synced
- Discovery report

**Success criteria**:
- Minimum: 20 auto-approved prospects
- Target: 100 auto-approved prospects
- Stretch: 200 auto-approved prospects

---

### 2. Dynamic Outreach (`dynamic-outreach.workflow.yaml`)

**Purpose**: Event-driven workflow that adapts to prospect behavior in real-time

**Execution mode**: Reactive (event-driven, not sequential)

**Key features**:
- Responds to prospect behavior dynamically
- Analyzes sentiment and intent
- Adapts messaging based on engagement
- Handles objections intelligently
- Escalates hot leads immediately

**Event triggers**:
- `prospect_reply_received` → Analyze and respond
- `email_opened_3_times` → Direct ask
- `link_clicked` → Content progression
- `pricing_page_visited` → Accelerate to demo
- `out_of_office_detected` → Pause and reschedule
- `objection_detected` → Objection handling
- `positive_signal_detected` → Book meeting
- `competitor_mentioned` → Send comparison

**Flows**:
1. **analyze-and-respond-flow**: Main response handler
2. **high-intent-sequence**: For engaged prospects
3. **accelerate-to-demo**: Fast-track to booking
4. **objection-handling-flow**: Strategic objection responses
5. **content-engagement-flow**: Progressive content delivery
6. **pause-and-reschedule**: Respect availability
7. **graceful-exit-flow**: Professional unsubscribe
8. **competitive-differentiation-flow**: Competitor responses

**Slash commands**:
```
/start-dynamic-outreach      # Launch adaptive campaign
/handle-reply               # Process specific reply
/high-intent                # Engage high-intent prospect
```

**Example scenarios**:

**Scenario 1: High-Intent Prospect**
```
Trigger: Email opened 3 times (no reply)
→ Engagement-analyst: "High intent detected"
→ Conversation-strategist: "Send direct ask with calendar link"
→ Outreach-orchestrator: "Send within 30 minutes"
Result: Prospect books demo
```

**Scenario 2: Objection Handling**
```
Trigger: Reply "We use Competitor X"
→ Engagement-analyst: "Competitor objection, neutral sentiment"
→ Conversation-strategist: "Acknowledge + differentiate"
→ Outreach-orchestrator: "Send comparison guide"
Result: Prospect clicks comparison, re-engages
```

---

### 3. Re-engagement (`re-engagement.workflow.yaml`)

**Purpose**: Revive dormant prospects with fresh value

**Phases**:
1. **Identify**: Find cold leads worth pursuing
2. **Analyze**: Understand why they went cold
3. **Re-engage**: Fresh messaging and approach
4. **Monitor**: Track performance and learn

**Criteria for re-engagement**:
- Last contacted > 90 days ago
- Previously contacted (2+ touches)
- Opened at least one email
- ICP score still > 70
- Not unsubscribed/spam

**Segmentation strategies**:
- **High engagement, never replied**: Direct ask with new angle
- **Low engagement, timing issue**: New value prop
- **Engaged then ghosted**: Acknowledge gap, provide update
- **Competitor switched**: New differentiation

**Slash commands**:
```
/re-engage-cold-leads        # General re-engagement
/re-engage-competitor        # Target competitor users
```

**Success metrics**:
- Reply rate > 10% (vs 5% cold outreach)
- Meetings booked > 5
- Better performance than original campaign

---

## 🚀 Integration with Your Sales System

### Directory Structure
```
claude - sales_auto_skill/
├── mcp-server/              # Your existing MCP server
├── bmad-workflows/          # These BMAD workflows
└── src/
    └── autonomous-executor.ts  # Integration layer (to be built)
```

### Integration Architecture

```typescript
// autonomous-executor.ts
import { BMADAdapter } from '../../bmad_mad_claude_code_plugin';
import { SalesAutomationAPI } from './mcp-server/api';

class AutonomousSalesExecutor {
  private bmad: BMADAdapter;
  private salesAPI: SalesAutomationAPI;

  constructor() {
    // Initialize BMAD with your workflows
    this.bmad = new BMADAdapter({
      rootPath: './bmad-workflows'
    });

    // Your existing sales API
    this.salesAPI = new SalesAutomationAPI();
  }

  async initialize() {
    await this.bmad.initialize();
    this.setupEventHandlers();
  }

  // Map BMAD workflow steps to your API calls
  async executeWorkflowStep(step: WorkflowStep) {
    switch (step.action) {
      case 'execute_company_search':
        return await this.salesAPI.searchCompanies({
          ...step.inputs,
          source: 'explorium'
        });

      case 'enrich_with_explorium':
        return await this.salesAPI.enrichContacts({
          contacts: step.inputs.contact_list,
          provider: 'explorium'
        });

      case 'setup_lemlist_campaign':
        return await this.salesAPI.createLemlistCampaign({
          prospects: step.inputs.auto_approve_list,
          config: step.inputs.campaign_config
        });

      case 'classify_response':
        return await this.analyzeReplyWithClaude(step.inputs.reply_content);

      // ... more mappings
    }
  }

  // Event-driven outreach
  private setupEventHandlers() {
    // Lemlist reply webhook → BMAD workflow
    this.salesAPI.onProspectReply(async (reply) => {
      const workflow = await this.bmad.getWorkflow('dynamic-outreach');
      await this.executeFlow(workflow.flows['analyze-and-respond-flow'], {
        reply_content: reply.body,
        prospect_email: reply.from
      });
    });

    // Email tracking → BMAD workflow
    this.salesAPI.onEmailOpened(async (event) => {
      if (event.openCount >= 3) {
        const workflow = await this.bmad.getWorkflow('dynamic-outreach');
        await this.executeFlow(workflow.flows['high-intent-sequence'], {
          prospect_email: event.email
        });
      }
    });
  }
}
```

### Using with Your MCP Server

```typescript
// In your mcp-server/src/autonomous-mode.ts
import { AutonomousSalesExecutor } from '../autonomous-executor';

class YOLOMode {
  private executor: AutonomousSalesExecutor;

  async runDiscoveryPipeline() {
    // Use BMAD prospect-discovery workflow
    const workflow = await this.executor.bmad.getWorkflow('prospect-discovery');

    const results = await this.executor.executeWorkflow(workflow, {
      industry: this.config.targetIndustry,
      geography: this.config.targetGeography,
      titles: this.config.targetTitles
    });

    console.log(`Discovered ${results.auto_approve_list.length} qualified prospects`);
    console.log(`Campaign ID: ${results.campaign_id}`);
  }

  async handleProspectEngagement(event) {
    // Use BMAD dynamic-outreach workflow
    await this.executor.handleEvent({
      type: event.type,  // 'prospect_reply_received', 'email_opened_3_times', etc.
      prospectId: event.email,
      data: event
    });
  }
}
```

## 🎯 Usage Examples

### Example 1: Discover UK SaaS CTOs

```bash
# Via slash command
/discover-uk-saas

# Or programmatically
const workflow = await bmad.getWorkflow('prospect-discovery');
const results = await executor.executeWorkflow(workflow, {
  icp: {
    industry: ['SaaS'],
    geography: ['United Kingdom'],
    company_size: '50-500',
    revenue: '$5M-$50M',
    titles: ['CTO', 'VP Engineering']
  }
});

// Results:
// ✅ 234 companies found
// ✅ 412 contacts extracted
// ✅ 387 emails verified
// ✅ 142 auto-approved (ICP >= 85)
// ✅ Campaign created in Lemlist
// ✅ Contacts synced to HubSpot
```

### Example 2: Handle Prospect Reply

```bash
# Automatic via webhook
POST /webhooks/lemlist/reply
{
  "email": "john@company.com",
  "body": "Interesting, but we're using Competitor X..."
}

# Triggers:
# 1. engagement-analyst: Classifies as "competitor objection, neutral"
# 2. conversation-strategist: Crafts competitor response
# 3. outreach-orchestrator: Sends comparison guide
# 4. Monitors for engagement on comparison guide
```

### Example 3: Re-engage Cold Leads

```bash
/re-engage-cold-leads

# Workflow:
# 1. Find prospects: last contact >90 days, ICP >70, opened emails
# 2. Analyze: Why did they go cold?
# 3. Validate: Still relevant?
# 4. Segment: High engagement vs timing issues vs competitor
# 5. Craft: Fresh messaging angles
# 6. Launch: 2-touch re-engagement sequence
# 7. Monitor: Track performance vs original campaign
```

## 📊 Metrics and Monitoring

### Workflow Performance Tracking

```typescript
// Track workflow execution
const metrics = await executor.getWorkflowMetrics('prospect-discovery');

console.log({
  total_runs: metrics.totalRuns,
  avg_completion_time: metrics.avgCompletionTime,
  success_rate: metrics.successRate,
  avg_prospects_found: metrics.avgProspectsFound,
  cost_per_qualified_prospect: metrics.costPerProspect
});
```

### Agent Performance

```typescript
// See which agents are performing well
const agentMetrics = await executor.getAgentMetrics();

console.log({
  engagement_analyst: {
    classifications: 1234,
    accuracy: 0.92,
    avg_confidence: 0.85
  },
  conversation_strategist: {
    messages_crafted: 567,
    reply_rate: 0.18,
    positive_reply_rate: 0.12
  }
});
```

## 🔧 Configuration

### Environment Variables

```bash
# BMAD Workflows
export BMAD_ROOT_PATH="./bmad-workflows"
export BMAD_LOG_LEVEL="info"

# API Keys (your existing)
export EXPLORIUM_API_KEY="..."
export LEMLIST_API_KEY="..."
export HUBSPOT_API_KEY="..."
export ANTHROPIC_API_KEY="..."  # For Claude analysis
```

### Workflow Customization

```yaml
# Customize thresholds in workflows
# Edit: bmad-workflows/src/modules/sales/workflows/prospect-discovery.workflow.yaml

success_criteria:
  minimum:
    auto_approved_prospects: 50  # Increase from 20
  target:
    auto_approved_prospects: 200  # Increase from 100

guardrails:
  volume_limits:
    max_campaign_size: 1000  # Increase from 500
```

## 🎓 Next Steps

1. **Week 1**: Build integration layer
   - Map BMAD workflow steps to your existing APIs
   - Set up event handlers (Lemlist webhooks → BMAD flows)
   - Test end-to-end with 10 prospects

2. **Week 2**: Deploy prospect discovery
   - Run `/discover-prospects` workflow
   - Validate quality gates
   - Launch first campaign

3. **Week 3**: Enable dynamic outreach
   - Connect webhook handlers
   - Test objection handling
   - Monitor adaptive responses

4. **Week 4**: Full autonomous mode
   - Enable YOLO mode with BMAD workflows
   - Monitor performance
   - Iterate on messaging

## 📚 Documentation

- [BMAD Adapter Docs](../../bmad_mad_claude_code_plugin/README.md)
- [Sales Automation Docs](../docs/)
- [MCP Server API](../mcp-server/README.md)

## 🤝 Support

Questions? Check the workflows for inline documentation or review the agent persona files for detailed capabilities.

---

**Built with**: BMAD v6, TypeScript, Claude Sonnet 4.5
**Status**: Production-ready workflows ✅
**Version**: 1.0.0
