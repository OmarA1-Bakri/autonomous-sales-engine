# BMAD Workflows Integration Guide

Quick guide to integrate BMAD workflows with your RTGS Sales Automation system.

## 🎯 What You Just Got

### ✅ 4 Specialized Sales Agents
1. **sales-strategist** - ICP definition, market analysis, strategy
2. **engagement-analyst** - Sentiment analysis, buying signals, objection detection
3. **conversation-strategist** - Dynamic messaging, objection handling, personalization
4. **outreach-orchestrator** - Campaign execution, tracking, deliverability

### ✅ 3 Production Workflows
1. **prospect-discovery** - Full pipeline: ICP → Search → Enrich → Qualify → Campaign
2. **dynamic-outreach** - Event-driven adaptive outreach with real-time responses
3. **re-engagement** - Revive cold leads with fresh value

## 🚀 Quick Start Integration

### Step 1: Install BMAD Adapter (5 minutes)

```bash
cd /home/omar/bmad_mad_claude_code_plugin
npm install
npm run build

# Verify it works
npm test
# Expected: 393 tests passing
```

### Step 2: Create Integration Layer (30 minutes)

Create `/home/omar/claude - sales_auto_skill/src/bmad-integration.ts`:

```typescript
import { BMADAdapter } from '../../bmad_mad_claude_code_plugin/src/adapters/bmad/adapter';
import { db } from './mcp-server/src/database';
import { explorium } from './mcp-server/src/services/explorium';
import { lemlist } from './mcp-server/src/services/lemlist';

export class BMADSalesIntegration {
  private bmad: BMADAdapter;

  constructor() {
    this.bmad = new BMADAdapter({
      rootPath: './bmad-workflows'
    });
  }

  async initialize() {
    await this.bmad.initialize();
    console.log('BMAD workflows loaded:', this.bmad.getStats());
  }

  // Execute prospect discovery workflow
  async discoverProspects(icpCriteria: any) {
    const workflow = await this.bmad.searchWorkflows({
      name: 'prospect-discovery'
    })[0];

    const results = {
      companies: [],
      contacts: [],
      qualified: []
    };

    // Step 1: Search companies (Explorium)
    console.log('Searching companies with Explorium...');
    const companies = await explorium.searchCompanies({
      industry: icpCriteria.industry,
      employeeRange: icpCriteria.companySize,
      country: icpCriteria.geography
    });
    results.companies = companies;

    // Step 2: Find decision-makers (Explorium)
    console.log(`Found ${companies.length} companies, finding contacts...`);
    for (const company of companies.slice(0, 100)) { // Limit for testing
      const contacts = await explorium.findContacts({
        companyDomain: company.domain,
        titles: icpCriteria.titles
      });
      results.contacts.push(...contacts);
    }

    // Step 3: Enrich with Explorium
    console.log(`Found ${results.contacts.length} contacts, enriching...`);
    const enriched = await explorium.enrichContacts(results.contacts);

    // Step 4: Score and qualify
    console.log('Scoring ICP fit...');
    const scored = enriched.map(contact => ({
      ...contact,
      icpScore: this.calculateICPScore(contact, icpCriteria)
    }));

    results.qualified = scored.filter(c => c.icpScore >= 70);

    // Step 5: Create Lemlist campaign
    const autoApproved = results.qualified.filter(c => c.icpScore >= 85);
    console.log(`Auto-approved: ${autoApproved.length} prospects`);

    if (autoApproved.length > 0) {
      const campaign = await lemlist.createCampaign({
        name: `${icpCriteria.industry} Discovery ${new Date().toISOString().split('T')[0]}`,
        prospects: autoApproved
      });

      console.log(`Campaign created: ${campaign.id}`);
      results.campaignId = campaign.id;
    }

    return results;
  }

  // Handle prospect reply with dynamic workflow
  async handleProspectReply(reply: any) {
    const workflow = await this.bmad.searchWorkflows({
      name: 'dynamic-outreach'
    })[0];

    // Get engagement analyst agent
    const analyst = this.bmad.getAgent('sales', 'engagement-analyst');
    console.log(`Analyzing reply from ${reply.email} with ${analyst.metadata.title}...`);

    // Analyze sentiment (using Claude)
    const analysis = await this.analyzeReplyWithClaude(reply.body);

    console.log('Analysis:', {
      sentiment: analysis.sentiment,
      intent: analysis.intent,
      urgency: analysis.urgency_score,
      buyingSignals: analysis.buying_signals
    });

    // Determine next action based on analysis
    if (analysis.urgency_score > 70 && analysis.sentiment === 'positive') {
      console.log('🔥 HOT LEAD - Accelerating to demo');
      return await this.accelerateToDemo(reply.email);
    }

    if (analysis.sentiment === 'objection') {
      console.log('⚠️ Objection detected:', analysis.objection_type);
      return await this.handleObjection(reply.email, analysis.objection_type);
    }

    if (analysis.sentiment === 'negative') {
      console.log('🛑 Not interested - graceful exit');
      return await this.gracefulExit(reply.email);
    }

    // Default: neutral engagement
    console.log('📧 Neutral engagement - contextual follow-up');
    return await this.sendContextualFollowUp(reply.email, analysis);
  }

  private async analyzeReplyWithClaude(replyBody: string) {
    // Use Claude to analyze reply
    // This would integrate with your existing Claude API
    const response = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: {
        'x-api-key': process.env.ANTHROPIC_API_KEY,
        'anthropic-version': '2023-06-01',
        'content-type': 'application/json'
      },
      body: JSON.stringify({
        model: 'claude-haiku-4-5',
        max_tokens: 1024,
        messages: [{
          role: 'user',
          content: `Analyze this sales prospect reply and return JSON:

Reply: "${replyBody}"

Return JSON with:
{
  "sentiment": "positive" | "neutral" | "negative" | "objection",
  "intent": "interested" | "not_interested" | "need_info" | "timing",
  "urgency_score": 0-100,
  "buying_signals": ["budget_mentioned", "timeline_mentioned", etc],
  "objection_type": "price" | "timing" | "competitor" | "authority" | null
}`
        }]
      })
    });

    const result = await response.json();
    return JSON.parse(result.content[0].text);
  }

  private calculateICPScore(contact: any, icp: any): number {
    let score = 0;

    // Company size match (15 points)
    if (this.matchesRange(contact.company.employeeCount, icp.companySize)) {
      score += 15;
    }

    // Industry match (15 points)
    if (icp.industry.includes(contact.company.industry)) {
      score += 15;
    }

    // Title match (20 points)
    if (icp.titles.some(title => contact.title.includes(title))) {
      score += 20;
    }

    // Email verified (20 points)
    if (contact.emailVerified) {
      score += 20;
    }

    // LinkedIn profile found (10 points)
    if (contact.linkedinUrl) {
      score += 10;
    }

    // Geography match (10 points)
    if (icp.geography.includes(contact.company.country)) {
      score += 10;
    }

    // Has phone (10 points)
    if (contact.phone) {
      score += 10;
    }

    return score;
  }

  private matchesRange(value: number, range: string): boolean {
    const [min, max] = range.split('-').map(n => parseInt(n.replace(/\D/g, '')));
    return value >= min && value <= max;
  }

  private async accelerateToDemo(email: string) {
    // Send calendar link immediately
    return await lemlist.sendEmail({
      to: email,
      template: 'calendar_link',
      variables: { urgency: 'high' }
    });
  }

  private async handleObjection(email: string, type: string) {
    const templates = {
      price: 'roi_calculator',
      timing: 'acknowledge_timing',
      competitor: 'comparison_guide',
      authority: 'shareable_deck'
    };

    return await lemlist.sendEmail({
      to: email,
      template: templates[type] || 'general_objection',
      variables: { objection_type: type }
    });
  }

  private async gracefulExit(email: string) {
    await lemlist.pauseSequence(email);
    await db.updateContactStatus(email, 'not_interested');
  }

  private async sendContextualFollowUp(email: string, analysis: any) {
    return await lemlist.sendEmail({
      to: email,
      template: 'contextual_followup',
      variables: {
        intent: analysis.intent,
        engagement_level: analysis.urgency_score > 50 ? 'high' : 'medium'
      }
    });
  }
}
```

### Step 3: Add to Your YOLO Mode (15 minutes)

Update `/home/omar/claude - sales_auto_skill/mcp-server/src/yolo-mode.ts`:

```typescript
import { BMADSalesIntegration } from '../../src/bmad-integration';

export class YOLOMode {
  private bmad: BMADSalesIntegration;

  async initialize() {
    this.bmad = new BMADSalesIntegration();
    await this.bmad.initialize();
    console.log('✅ BMAD workflows ready');
  }

  async runDiscoveryPipeline() {
    console.log('🚀 Starting BMAD-powered discovery...');

    const results = await this.bmad.discoverProspects({
      industry: ['SaaS', 'FinTech'],
      geography: ['United Kingdom'],
      companySize: '50-500',
      titles: ['CTO', 'VP Engineering', 'Head of Product']
    });

    console.log(`
    ✅ Discovery complete:
    - Companies found: ${results.companies.length}
    - Contacts extracted: ${results.contacts.length}
    - Qualified prospects: ${results.qualified.length}
    - Campaign created: ${results.campaignId}
    `);
  }
}
```

### Step 4: Set Up Webhook Handler (10 minutes)

Update `/home/omar/claude - sales_auto_skill/mcp-server/src/webhooks/lemlist.ts`:

```typescript
import { BMADSalesIntegration } from '../../../src/bmad-integration';

const bmad = new BMADSalesIntegration();

export async function handleLemlistReply(req, res) {
  const { email, body, firstName } = req.body;

  console.log(`📨 Reply received from ${firstName} (${email})`);

  // Use BMAD dynamic-outreach workflow
  const response = await bmad.handleProspectReply({
    email,
    body,
    firstName
  });

  res.json({ success: true, action: response });
}
```

## 🎯 Testing Your Integration

### Test 1: Discover Prospects

```bash
cd /home/omar/claude\ -\ sales_auto_skill

# Create test script
cat > test-bmad-discovery.js << 'EOF'
const { BMADSalesIntegration } = require('./src/bmad-integration');

async function test() {
  const bmad = new BMADSalesIntegration();
  await bmad.initialize();

  console.log('Running discovery workflow...');
  const results = await bmad.discoverProspects({
    industry: ['SaaS'],
    geography: ['United Kingdom'],
    companySize: '50-200',
    titles: ['CTO', 'VP Engineering']
  });

  console.log('Results:', {
    companies: results.companies.length,
    contacts: results.contacts.length,
    qualified: results.qualified.length,
    campaignId: results.campaignId
  });
}

test().catch(console.error);
EOF

node test-bmad-discovery.js
```

### Test 2: Handle Reply

```bash
cat > test-bmad-reply.js << 'EOF'
const { BMADSalesIntegration } = require('./src/bmad-integration');

async function test() {
  const bmad = new BMADSalesIntegration();
  await bmad.initialize();

  // Test positive reply
  console.log('\n=== Testing positive reply ===');
  await bmad.handleProspectReply({
    email: 'test@company.com',
    body: 'This looks interesting! Can you send pricing for 100 users? We need to decide by end of Q1.',
    firstName: 'John'
  });

  // Test objection
  console.log('\n=== Testing competitor objection ===');
  await bmad.handleProspectReply({
    email: 'test2@company.com',
    body: 'Thanks but we are currently using Competitor X and fairly happy.',
    firstName: 'Jane'
  });

  // Test negative
  console.log('\n=== Testing negative response ===');
  await bmad.handleProspectReply({
    email: 'test3@company.com',
    body: 'Not interested, please remove me.',
    firstName: 'Bob'
  });
}

test().catch(console.error);
EOF

node test-bmad-reply.js
```

Expected output:
```
=== Testing positive reply ===
Analyzing reply from test@company.com with Engagement Analyst...
Analysis: {
  sentiment: 'positive',
  intent: 'interested',
  urgency: 85,
  buyingSignals: [ 'pricing_requested', 'timeline_mentioned' ]
}
🔥 HOT LEAD - Accelerating to demo

=== Testing competitor objection ===
Analyzing reply from test2@company.com with Engagement Analyst...
Analysis: {
  sentiment: 'objection',
  intent: 'objection',
  urgency: 40,
  buyingSignals: [ 'competitor_mentioned' ]
}
⚠️ Objection detected: competitor

=== Testing negative response ===
Analyzing reply from test3@company.com with Engagement Analyst...
Analysis: { sentiment: 'negative', intent: 'not_interested', urgency: 0 }
🛑 Not interested - graceful exit
```

## 📊 Monitor BMAD Workflows

Add to your desktop app dashboard:

```typescript
// In desktop-app/src/views/Dashboard.jsx

function BMADWorkflowStats() {
  const [stats, setStats] = useState(null);

  useEffect(() => {
    fetch('/api/bmad/stats')
      .then(r => r.json())
      .then(setStats);
  }, []);

  return (
    <div className="bg-slate-800 rounded-lg p-4">
      <h3 className="text-lg font-semibold mb-3">BMAD Workflows</h3>
      <div className="grid grid-cols-3 gap-4">
        <MetricCard
          label="Prospects Discovered"
          value={stats?.prospectsDiscovered || 0}
          icon="🔍"
        />
        <MetricCard
          label="Replies Analyzed"
          value={stats?.repliesAnalyzed || 0}
          icon="🧠"
        />
        <MetricCard
          label="Hot Leads Detected"
          value={stats?.hotLeadsDetected || 0}
          icon="🔥"
        />
      </div>
    </div>
  );
}
```

## 🎓 Next Steps

1. **Week 1**: Build basic integration
   - ✅ Install BMAD adapter
   - ✅ Create integration layer
   - ✅ Test discovery workflow

2. **Week 2**: Connect dynamic outreach
   - Set up Lemlist webhooks
   - Connect reply handler
   - Test objection handling

3. **Week 3**: Full autonomous mode
   - Enable YOLO mode with BMAD
   - Monitor performance
   - Iterate on workflows

4. **Week 4**: Optimize and scale
   - Tune ICP scoring
   - Refine messaging templates
   - Scale up campaign volumes

## 🤝 Need Help?

Check the README.md in this directory for full documentation of agents and workflows.

---

**You now have autonomous, adaptive sales workflows powered by BMAD!** 🚀
