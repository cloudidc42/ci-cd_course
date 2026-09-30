# Part 74: Continuous Feedback Loops

## บทนำ

Continuous Feedback คือแนวทางที่ทำให้ข้อมูลจาก production ไหลกลับมาถึงทีม development ได้อย่างรวดเร็วและต่อเนื่อง ทำให้สามารถตัดสินใจได้ดีขึ้นและปรับปรุง product ได้เร็วขึ้น

ในบทนี้เราจะเรียนรู้:
- Production Feedback ใน Development Cycle
- Feature Usage Analytics
- Error Tracking ด้วย Sentry
- User Feedback Collection
- Deployment Impact Measurement
- แบบฝึกหัดปฏิบัติ

---

## 74.1 ทำไม Continuous Feedback ถึงสำคัญ?

### Feedback Loop Cycle

```
┌─────────────────────────────────────────────────────────────┐
│                  Continuous Feedback Loop                    │
│                                                             │
│    Code → Build → Test → Deploy → Monitor → Feedback       │
│      ↑                                          │           │
│      └──────────────────────────────────────────┘           │
│                    (Fast Loop)                               │
│                                                             │
│    Business Outcomes → Product Decisions → Features         │
│      ↑                                          │           │
│      └──────────────────────────────────────────┘           │
│                   (Slow Loop)                                │
└─────────────────────────────────────────────────────────────┘
```

### ประเภทของ Feedback

```
1. Technical Feedback (เร็ว: นาที-ชั่วโมง)
   - Unit test results
   - Build failures
   - Performance regressions
   - Error rates

2. User Experience Feedback (ปานกลาง: ชั่วโมง-วัน)
   - Error tracking (Sentry)
   - Performance metrics (Core Web Vitals)
   - Feature usage analytics
   - User sessions (FullStory, Hotjar)

3. Business Feedback (ช้า: วัน-สัปดาห์)
   - Feature adoption rates
   - Conversion rates
   - Revenue impact
   - Customer satisfaction (NPS)
```

---

## 74.2 Feature Usage Analytics

### การ Track Feature Usage

```typescript
// analytics/feature-tracker.ts
import { Analytics } from '@segment/analytics-next';

interface FeatureEvent {
  featureId: string;
  featureName: string;
  action: string;
  userId: string;
  properties?: Record<string, unknown>;
  timestamp?: Date;
}

class FeatureAnalytics {
  private analytics: Analytics;
  private featureFlags: Record<string, boolean> = {};

  constructor(writeKey: string) {
    this.analytics = new Analytics({ writeKey });
  }

  // Track feature interaction
  track(event: FeatureEvent): void {
    this.analytics.track(`Feature ${event.action}`, {
      feature_id: event.featureId,
      feature_name: event.featureName,
      user_id: event.userId,
      timestamp: event.timestamp || new Date(),
      ...event.properties,
    });
  }

  // Track feature impression (user saw the feature)
  trackImpression(featureId: string, userId: string): void {
    this.analytics.track('Feature Impression', {
      feature_id: featureId,
      user_id: userId,
      timestamp: new Date(),
    });
  }

  // Track feature adoption
  trackAdoption(featureId: string, userId: string, adopted: boolean): void {
    this.analytics.track('Feature Adoption', {
      feature_id: featureId,
      user_id: userId,
      adopted,
      timestamp: new Date(),
    });
  }
}

// React Hook สำหรับ feature tracking
export function useFeatureTracking(featureId: string) {
  const analytics = useAnalytics();

  const trackFeatureUsed = useCallback(
    (action: string, properties?: Record<string, unknown>) => {
      analytics.track({
        featureId,
        action,
        properties,
      });
    },
    [featureId, analytics]
  );

  // Track เมื่อ component mount (impression)
  useEffect(() => {
    analytics.trackImpression(featureId, currentUser.id);
  }, [featureId]);

  return { trackFeatureUsed };
}

// ตัวอย่างการใช้ใน Component
export function PaymentButton({ orderId }: { orderId: string }) {
  const { trackFeatureUsed } = useFeatureTracking('new-payment-flow');

  const handlePayment = async () => {
    trackFeatureUsed('clicked', { orderId });

    try {
      await processPayment(orderId);
      trackFeatureUsed('success', { orderId });
    } catch (error) {
      trackFeatureUsed('error', { orderId, error: error.message });
    }
  };

  return <button onClick={handlePayment}>ชำระเงิน</button>;
}
```

### Feature Flag + Analytics Integration

```typescript
// feature-flags/with-analytics.ts
import { OpenFeature } from '@openfeature/server-sdk';
import { FeatureAnalytics } from './feature-tracker';

interface FeatureFlagWithTracking {
  enabled: boolean;
  variant?: string;
  trackExposure: () => void;
}

class AnalyticsAwareFeatureFlags {
  private client = OpenFeature.getClient();
  private analytics: FeatureAnalytics;

  constructor(analytics: FeatureAnalytics) {
    this.analytics = analytics;
  }

  async getFlag(
    flagKey: string,
    userId: string,
    defaultValue: boolean = false,
  ): Promise<FeatureFlagWithTracking> {
    const evaluation = await this.client.getBooleanDetails(flagKey, defaultValue, {
      targetingKey: userId,
    });

    const enabled = evaluation.value;

    return {
      enabled,
      variant: evaluation.variant,
      trackExposure: () => {
        // Track เมื่อ user เห็น feature flag
        this.analytics.track({
          featureId: flagKey,
          featureName: flagKey,
          action: 'exposure',
          userId,
          properties: {
            enabled,
            variant: evaluation.variant,
            reason: evaluation.reason,
          },
        });
      },
    };
  }
}

// ตัวอย่างการวิเคราะห์ Feature Adoption
interface FeatureMetrics {
  featureId: string;
  impressions: number;
  interactions: number;
  adoptionRate: number;
  errorRate: number;
}

async function calculateFeatureMetrics(
  featureId: string,
  period: '7d' | '30d' | '90d',
): Promise<FeatureMetrics> {
  // Query จาก analytics database (e.g., BigQuery, Snowflake)
  const query = `
    SELECT
      feature_id,
      COUNT(DISTINCT CASE WHEN action = 'impression' THEN user_id END) as impressions,
      COUNT(DISTINCT CASE WHEN action IN ('clicked', 'used') THEN user_id END) as interactions,
      COUNT(DISTINCT CASE WHEN action = 'error' THEN user_id END) as errors
    FROM feature_events
    WHERE
      feature_id = '${featureId}'
      AND timestamp >= NOW() - INTERVAL ${period}
    GROUP BY feature_id
  `;

  const result = await analyticsDb.query(query);
  const row = result[0];

  return {
    featureId,
    impressions: row.impressions,
    interactions: row.interactions,
    adoptionRate: row.interactions / row.impressions,
    errorRate: row.errors / row.interactions,
  };
}
```

---

## 74.3 Error Tracking ด้วย Sentry

### ติดตั้ง Sentry

```bash
# Node.js/TypeScript
npm install @sentry/node @sentry/profiling-node

# React
npm install @sentry/react

# Go
go get github.com/getsentry/sentry-go

# Python
pip install sentry-sdk
```

### Sentry สำหรับ Node.js

```typescript
// sentry/setup.ts
import * as Sentry from '@sentry/node';
import { nodeProfilingIntegration } from '@sentry/profiling-node';

export function initSentry(environment: string): void {
  Sentry.init({
    dsn: process.env.SENTRY_DSN,
    environment,
    release: process.env.APP_VERSION || 'unknown',

    // Performance monitoring
    tracesSampleRate: environment === 'production' ? 0.1 : 1.0,
    profilesSampleRate: environment === 'production' ? 0.05 : 1.0,

    integrations: [
      nodeProfilingIntegration(),
      // HTTP request tracking
      new Sentry.Integrations.Http({ tracing: true }),
      // Express middleware
      new Sentry.Integrations.Express({ app }),
    ],

    // กรอง errors ที่ไม่สำคัญ
    beforeSend(event, hint) {
      // ไม่ส่ง 4xx errors ไปยัง Sentry (ยกเว้น 429)
      const statusCode = event.contexts?.response?.status_code;
      if (statusCode && statusCode >= 400 && statusCode < 500 && statusCode !== 429) {
        return null;
      }

      // เพิ่ม context
      event.tags = {
        ...event.tags,
        service: process.env.SERVICE_NAME,
        team: process.env.TEAM_NAME,
      };

      return event;
    },

    // กำหนด user context
    initialScope: {
      tags: {
        service: process.env.SERVICE_NAME || 'unknown',
        k8s_pod: process.env.POD_NAME || 'unknown',
      },
    },
  });
}

// Express middleware
export function setupSentryMiddleware(app: Express.Application): void {
  // Request handler (ต้องเป็น first middleware)
  app.use(Sentry.Handlers.requestHandler());

  // Tracing handler
  app.use(Sentry.Handlers.tracingHandler());

  // Error handler (ต้องเป็น last middleware ก่อน default error handler)
  app.use(Sentry.Handlers.errorHandler({
    shouldHandleError(error) {
      // ส่งแค่ 500 errors
      if (error.status >= 500) {
        return true;
      }
      return false;
    },
  }));
}
```

### Custom Error Tracking

```typescript
// error-tracking/custom.ts
import * as Sentry from '@sentry/node';

// Custom error boundary
export function captureWithContext(
  error: Error,
  context: Record<string, unknown>,
): void {
  Sentry.withScope(scope => {
    // เพิ่ม context
    scope.setExtras(context);

    // จัดหมวดหมู่ error
    if (context.userId) {
      scope.setUser({ id: context.userId as string });
    }

    if (context.orderId) {
      scope.setTag('order_id', context.orderId as string);
    }

    Sentry.captureException(error);
  });
}

// Business logic error tracking
export class PaymentService {
  async processPayment(orderId: string, amount: number): Promise<void> {
    const transaction = Sentry.startTransaction({
      op: 'payment.process',
      name: 'Process Payment',
    });

    Sentry.setContext('payment', {
      orderId,
      amount,
      currency: 'THB',
    });

    try {
      const span = transaction.startChild({
        op: 'payment.validate',
        description: 'Validate payment details',
      });

      await this.validatePayment(orderId, amount);
      span.finish();

      const chargeSpan = transaction.startChild({
        op: 'payment.charge',
        description: 'Charge customer',
      });

      await this.chargeCustomer(orderId, amount);
      chargeSpan.finish();

      transaction.setStatus('ok');
    } catch (error) {
      transaction.setStatus('internal_error');

      captureWithContext(error as Error, {
        orderId,
        amount,
        step: 'payment_processing',
      });

      throw error;
    } finally {
      transaction.finish();
    }
  }
}
```

### Sentry Alerts และ CI/CD Integration

```yaml
# .sentry/sentry.properties
defaults.org=mycompany
defaults.project=payment-service
auth.token=${SENTRY_AUTH_TOKEN}
```

```yaml
# .github/workflows/sentry-release.yaml
name: Sentry Release

on:
  push:
    branches: [main]

jobs:
  create-release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Create Sentry Release
        uses: getsentry/action-release@v1
        env:
          SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
          SENTRY_ORG: mycompany
          SENTRY_PROJECT: payment-service
        with:
          environment: production
          version: ${{ github.sha }}
          sourcemaps: ./dist
          url_prefix: '~/static/js'
          set_commits: auto
```

### Sentry Alerting Rules

```json
{
  "name": "High Error Rate Alert",
  "conditions": [
    {
      "id": "sentry.rules.conditions.event_frequency:EventFrequencyCondition",
      "value": 100,
      "interval": "5m"
    }
  ],
  "filters": [
    {
      "id": "sentry.rules.filters.level_filter:LevelFilter",
      "match": "gte",
      "level": "error"
    }
  ],
  "actions": [
    {
      "id": "sentry.integrations.pagerduty.notify_service:PagerDutyNotifyServiceAction",
      "account": "mycompany",
      "service": "payment-service"
    },
    {
      "id": "sentry.integrations.slack.notify_action:SlackNotifyServiceAction",
      "workspace": "T12345",
      "channel": "#alerts-payment",
      "channel_id": "C12345"
    }
  ]
}
```

---

## 74.4 User Feedback Collection

### In-app Feedback Widget

```typescript
// feedback/widget.tsx
import React, { useState } from 'react';
import * as Sentry from '@sentry/react';

interface FeedbackForm {
  type: 'bug' | 'feature' | 'improvement';
  description: string;
  email?: string;
  screenshot?: boolean;
}

export function FeedbackWidget() {
  const [isOpen, setIsOpen] = useState(false);
  const [form, setForm] = useState<FeedbackForm>({
    type: 'bug',
    description: '',
  });
  const [submitted, setSubmitted] = useState(false);

  const handleSubmit = async () => {
    // ส่งไปยัง Sentry
    const feedback = await Sentry.captureFeedback({
      name: currentUser?.name,
      email: form.email || currentUser?.email,
      message: form.description,
      tags: {
        feedback_type: form.type,
        page: window.location.pathname,
      },
    });

    // ส่งไปยัง internal API
    await fetch('/api/feedback', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        ...form,
        userId: currentUser?.id,
        page: window.location.pathname,
        userAgent: navigator.userAgent,
        timestamp: new Date().toISOString(),
      }),
    });

    setSubmitted(true);
    setTimeout(() => {
      setIsOpen(false);
      setSubmitted(false);
      setForm({ type: 'bug', description: '' });
    }, 2000);
  };

  if (!isOpen) {
    return (
      <button
        className="feedback-trigger"
        onClick={() => setIsOpen(true)}
        title="ส่ง Feedback"
      >
        💬
      </button>
    );
  }

  return (
    <div className="feedback-widget">
      <div className="feedback-header">
        <h3>ส่ง Feedback</h3>
        <button onClick={() => setIsOpen(false)}>✕</button>
      </div>

      {submitted ? (
        <div className="feedback-success">
          ✅ ขอบคุณสำหรับ feedback ของคุณ!
        </div>
      ) : (
        <div className="feedback-form">
          <div className="form-group">
            <label>ประเภท</label>
            <select
              value={form.type}
              onChange={e => setForm({ ...form, type: e.target.value as any })}
            >
              <option value="bug">🐛 พบข้อผิดพลาด</option>
              <option value="feature">✨ ขอฟีเจอร์ใหม่</option>
              <option value="improvement">🔧 ข้อเสนอแนะ</option>
            </select>
          </div>

          <div className="form-group">
            <label>รายละเอียด</label>
            <textarea
              value={form.description}
              onChange={e => setForm({ ...form, description: e.target.value })}
              placeholder="อธิบายสิ่งที่คุณพบหรือต้องการ..."
              rows={4}
            />
          </div>

          <div className="form-group">
            <label>อีเมล (ไม่บังคับ)</label>
            <input
              type="email"
              value={form.email || ''}
              onChange={e => setForm({ ...form, email: e.target.value })}
              placeholder="เพื่อรับการแจ้งเตือนเมื่อแก้ไขแล้ว"
            />
          </div>

          <button
            className="submit-button"
            onClick={handleSubmit}
            disabled={!form.description}
          >
            ส่ง Feedback
          </button>
        </div>
      )}
    </div>
  );
}
```

### NPS Survey Integration

```typescript
// nps/survey.ts
interface NPSSurvey {
  userId: string;
  score: number;  // 0-10
  followUp?: string;
  context: {
    trigger: string;  // e.g., 'post-deployment', 'weekly', 'feature-use'
    featureId?: string;
    sessionCount: number;
  };
}

class NPSTracker {
  private readonly SURVEY_COOLDOWN_DAYS = 90;

  shouldShowSurvey(userId: string): boolean {
    const lastSurvey = this.getLastSurveyDate(userId);
    if (!lastSurvey) return true;

    const daysSince = (Date.now() - lastSurvey.getTime()) / (1000 * 60 * 60 * 24);
    return daysSince >= this.SURVEY_COOLDOWN_DAYS;
  }

  async submitSurvey(survey: NPSSurvey): Promise<void> {
    // บันทึกไปยัง analytics
    await fetch('/api/nps', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(survey),
    });

    // Update last survey date
    localStorage.setItem(`nps_survey_${survey.userId}`, new Date().toISOString());

    // Alert ถ้า detractor (score 0-6)
    if (survey.score <= 6) {
      await this.alertCustomerSuccess(survey);
    }
  }

  private async alertCustomerSuccess(survey: NPSSurvey): Promise<void> {
    await fetch('/api/slack/webhook', {
      method: 'POST',
      body: JSON.stringify({
        channel: '#customer-feedback',
        text: `⚠️ NPS Detractor Alert!\nUser: ${survey.userId}\nScore: ${survey.score}/10\nFeedback: ${survey.followUp || 'ไม่มี'}`,
      }),
    });
  }

  calculateNPS(surveys: NPSSurvey[]): number {
    if (surveys.length === 0) return 0;

    const promoters = surveys.filter(s => s.score >= 9).length;
    const detractors = surveys.filter(s => s.score <= 6).length;

    return ((promoters - detractors) / surveys.length) * 100;
  }
}
```

---

## 74.5 Deployment Impact Measurement

### A/B Testing Framework

```typescript
// ab-testing/experiment.ts
interface Experiment {
  id: string;
  name: string;
  variants: Variant[];
  allocation: number;  // % ของ users ที่เข้า experiment
  startDate: Date;
  endDate?: Date;
  targetingRules?: TargetingRule[];
}

interface Variant {
  id: string;
  name: string;
  weight: number;  // relative weight
  featureFlags?: Record<string, unknown>;
}

class ExperimentationEngine {
  private experiments: Map<string, Experiment> = new Map();

  async assignVariant(
    experimentId: string,
    userId: string,
  ): Promise<Variant | null> {
    const experiment = this.experiments.get(experimentId);
    if (!experiment) return null;

    // Check targeting rules
    if (!this.checkTargetingRules(experiment.targetingRules, userId)) {
      return null;
    }

    // Hash userId สำหรับ consistent assignment
    const hash = this.hashUser(userId + experimentId);
    const bucket = hash % 100;

    // ตรวจสอบว่า user อยู่ใน experiment population
    if (bucket >= experiment.allocation) {
      return null;
    }

    // หา variant ตาม weight
    return this.selectVariant(experiment.variants, hash);
  }

  private hashUser(input: string): number {
    let hash = 0;
    for (let i = 0; i < input.length; i++) {
      const char = input.charCodeAt(i);
      hash = (hash << 5) - hash + char;
      hash = hash & hash;
    }
    return Math.abs(hash);
  }

  private selectVariant(variants: Variant[], hash: number): Variant {
    const totalWeight = variants.reduce((sum, v) => sum + v.weight, 0);
    const bucket = (hash % totalWeight);

    let cumWeight = 0;
    for (const variant of variants) {
      cumWeight += variant.weight;
      if (bucket < cumWeight) {
        return variant;
      }
    }

    return variants[variants.length - 1];
  }
}

// Deployment-triggered experiment
async function measureDeploymentImpact(
  deploymentId: string,
  metrics: string[],
) {
  const preDeployment = await fetchMetrics(metrics, {
    period: '7d',
    before: deploymentTime,
  });

  const postDeployment = await fetchMetrics(metrics, {
    period: '7d',
    after: deploymentTime,
  });

  const impact = {};
  for (const metric of metrics) {
    const before = preDeployment[metric];
    const after = postDeployment[metric];
    impact[metric] = {
      before,
      after,
      change: ((after - before) / before) * 100,
      significant: isStatisticallySignificant(before, after),
    };
  }

  return impact;
}
```

### Deployment Annotations ใน Grafana

```typescript
// grafana/annotations.ts
class GrafanaAnnotationClient {
  private baseUrl: string;
  private apiKey: string;

  constructor(baseUrl: string, apiKey: string) {
    this.baseUrl = baseUrl;
    this.apiKey = apiKey;
  }

  async createDeploymentAnnotation(deployment: {
    service: string;
    version: string;
    environment: string;
    deployer: string;
    deployedAt: Date;
  }): Promise<void> {
    await fetch(`${this.baseUrl}/api/annotations`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        time: deployment.deployedAt.getTime(),
        tags: ['deployment', deployment.environment, deployment.service],
        text: `🚀 Deploy ${deployment.service}@${deployment.version} by ${deployment.deployer}`,
        isRegion: false,
      }),
    });
  }

  async createIncidentAnnotation(incident: {
    id: string;
    startedAt: Date;
    resolvedAt?: Date;
    severity: string;
    service: string;
  }): Promise<void> {
    if (incident.resolvedAt) {
      // Region annotation (start → end)
      await fetch(`${this.baseUrl}/api/annotations`, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${this.apiKey}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          time: incident.startedAt.getTime(),
          timeEnd: incident.resolvedAt.getTime(),
          tags: ['incident', incident.severity, incident.service],
          text: `🔴 Incident ${incident.id} - ${incident.severity}`,
          isRegion: true,
        }),
      });
    }
  }
}

// GitHub Actions integration
// .github/workflows/annotate-deployment.yaml
const ANNOTATE_SCRIPT = `
import * as core from '@actions/core';

const grafanaUrl = process.env.GRAFANA_URL;
const grafanaKey = process.env.GRAFANA_API_KEY;
const service = process.env.SERVICE_NAME;
const version = process.env.VERSION;

const client = new GrafanaAnnotationClient(grafanaUrl, grafanaKey);
await client.createDeploymentAnnotation({
  service,
  version,
  environment: 'production',
  deployer: process.env.GITHUB_ACTOR,
  deployedAt: new Date(),
});

core.info(\`Deployment annotation created for \${service}@\${version}\`);
`;
```

---

## 74.6 Production Feedback Pipeline

### Event-Driven Feedback Collection

```yaml
# kafka-feedback-pipeline.yaml
# ใช้ Kafka สำหรับ event streaming

version: '3.8'
services:
  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
    depends_on:
      - zookeeper

  feedback-collector:
    build: ./feedback-collector
    environment:
      KAFKA_BOOTSTRAP: kafka:9092
      SENTRY_DSN: ${SENTRY_DSN}
    depends_on:
      - kafka

  feedback-processor:
    build: ./feedback-processor
    environment:
      KAFKA_BOOTSTRAP: kafka:9092
      SLACK_WEBHOOK: ${SLACK_WEBHOOK}
      PAGERDUTY_KEY: ${PAGERDUTY_KEY}
    depends_on:
      - kafka

  feedback-dashboard:
    build: ./feedback-dashboard
    ports:
      - "3000:3000"
    environment:
      KAFKA_BOOTSTRAP: kafka:9092
```

```typescript
// feedback-collector/src/collector.ts
import { Kafka, Producer } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'feedback-collector',
  brokers: [process.env.KAFKA_BOOTSTRAP],
});

const producer: Producer = kafka.producer();

interface FeedbackEvent {
  type: 'error' | 'user_feedback' | 'feature_usage' | 'performance';
  source: string;
  data: unknown;
  timestamp: string;
  metadata: {
    service: string;
    version: string;
    environment: string;
    userId?: string;
  };
}

export async function publishFeedbackEvent(event: FeedbackEvent): Promise<void> {
  await producer.send({
    topic: `feedback.${event.type}`,
    messages: [
      {
        key: event.metadata.service,
        value: JSON.stringify(event),
        headers: {
          'content-type': 'application/json',
          source: event.source,
        },
      },
    ],
  });
}

// Sentry webhook → Kafka bridge
app.post('/webhooks/sentry', async (req, res) => {
  const sentryEvent = req.body;

  await publishFeedbackEvent({
    type: 'error',
    source: 'sentry',
    data: {
      id: sentryEvent.id,
      title: sentryEvent.culprit,
      level: sentryEvent.level,
      count: sentryEvent.times_seen,
      url: sentryEvent.url,
    },
    timestamp: new Date().toISOString(),
    metadata: {
      service: sentryEvent.project,
      version: sentryEvent.release,
      environment: sentryEvent.environment,
    },
  });

  res.status(200).json({ ok: true });
});
```

```typescript
// feedback-processor/src/processor.ts
import { Kafka, Consumer, EachMessagePayload } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'feedback-processor',
  brokers: [process.env.KAFKA_BOOTSTRAP],
});

const consumer: Consumer = kafka.consumer({ groupId: 'feedback-processors' });

async function processErrorFeedback(data: unknown): Promise<void> {
  const error = data as { level: string; count: number; title: string };

  // Alert ถ้า error rate สูง
  if (error.level === 'fatal' || error.count > 100) {
    await sendSlackAlert({
      channel: '#alerts-critical',
      message: `🚨 Critical Error Spike!\n${error.title}\nCount: ${error.count}`,
    });

    if (error.count > 500) {
      // Create PagerDuty incident
      await createPagerDutyIncident({
        title: `Error Spike: ${error.title}`,
        urgency: 'high',
        details: JSON.stringify(error),
      });
    }
  }
}

async function processFeatureUsage(data: unknown): Promise<void> {
  const usage = data as { featureId: string; action: string; userId: string };

  // อัพเดท real-time feature usage counter
  await redis.incr(`feature:${usage.featureId}:${usage.action}`);

  // ถ้า feature ไม่มีการใช้งาน 7 วัน ส่ง notification
  const lastUsed = await redis.get(`feature:${usage.featureId}:last_used`);
  if (!lastUsed) {
    await redis.set(
      `feature:${usage.featureId}:last_used`,
      new Date().toISOString()
    );
  }
}

await consumer.subscribe({
  topics: ['feedback.error', 'feedback.feature_usage', 'feedback.performance'],
  fromBeginning: false,
});

await consumer.run({
  eachMessage: async ({ topic, message }: EachMessagePayload) => {
    const data = JSON.parse(message.value?.toString() || '{}');

    switch (topic) {
      case 'feedback.error':
        await processErrorFeedback(data.data);
        break;
      case 'feedback.feature_usage':
        await processFeatureUsage(data.data);
        break;
      case 'feedback.performance':
        await processPerformanceFeedback(data.data);
        break;
    }
  },
});
```

---

## 74.7 Core Web Vitals Monitoring

### การ Track Performance

```typescript
// performance/web-vitals.ts
import { onCLS, onFID, onLCP, onTTFB, onFCP } from 'web-vitals';

interface VitalMetric {
  name: string;
  value: number;
  rating: 'good' | 'needs-improvement' | 'poor';
  delta: number;
  id: string;
}

function sendVitalToAnalytics(metric: VitalMetric): void {
  // ส่งไปยัง analytics
  fetch('/api/vitals', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      ...metric,
      page: window.location.pathname,
      userId: currentUser?.id,
      timestamp: new Date().toISOString(),
    }),
    // ใช้ keepalive เพื่อให้ส่งแม้ page จะ unload
    keepalive: true,
  });

  // ส่งไปยัง Google Analytics 4
  if (window.gtag) {
    window.gtag('event', metric.name, {
      event_category: 'Web Vitals',
      event_label: metric.id,
      value: Math.round(metric.name === 'CLS' ? metric.value * 1000 : metric.value),
      non_interaction: true,
    });
  }
}

// ลงทะเบียน all vitals
onCLS(sendVitalToAnalytics);
onFID(sendVitalToAnalytics);
onLCP(sendVitalToAnalytics);
onTTFB(sendVitalToAnalytics);
onFCP(sendVitalToAnalytics);
```

### Deployment-Correlated Performance

```python
# deployment-performance-correlation.py
import pandas as pd
import numpy as np
from datetime import datetime, timedelta

def analyze_deployment_performance_impact(
    deployments: list,
    performance_metrics: pd.DataFrame,
) -> dict:
    """
    วิเคราะห์ว่า deployment ส่งผลต่อ performance อย่างไร
    """
    results = []

    for deployment in deployments:
        deploy_time = deployment['deployed_at']

        # Baseline: 24 ชั่วโมงก่อน deploy
        baseline_start = deploy_time - timedelta(hours=24)
        baseline_end = deploy_time

        # Post-deploy: 24 ชั่วโมงหลัง deploy
        post_start = deploy_time
        post_end = deploy_time + timedelta(hours=24)

        # กรองข้อมูล
        baseline = performance_metrics[
            (performance_metrics['timestamp'] >= baseline_start) &
            (performance_metrics['timestamp'] < baseline_end)
        ]

        post_deploy = performance_metrics[
            (performance_metrics['timestamp'] >= post_start) &
            (performance_metrics['timestamp'] < post_end)
        ]

        if baseline.empty or post_deploy.empty:
            continue

        # คำนวณ change
        for metric in ['lcp_ms', 'fid_ms', 'cls', 'ttfb_ms', 'error_rate']:
            if metric not in performance_metrics.columns:
                continue

            baseline_p50 = baseline[metric].quantile(0.5)
            post_p50 = post_deploy[metric].quantile(0.5)
            baseline_p95 = baseline[metric].quantile(0.95)
            post_p95 = post_deploy[metric].quantile(0.95)

            change_pct = ((post_p50 - baseline_p50) / baseline_p50) * 100

            results.append({
                'deployment_id': deployment['id'],
                'service': deployment['service'],
                'version': deployment['version'],
                'deployed_at': deploy_time,
                'metric': metric,
                'baseline_p50': round(baseline_p50, 2),
                'post_deploy_p50': round(post_p50, 2),
                'change_pct': round(change_pct, 2),
                'baseline_p95': round(baseline_p95, 2),
                'post_deploy_p95': round(post_p95, 2),
                'regression': change_pct > 10,  # >10% worse = regression
            })

    return pd.DataFrame(results)


def send_regression_alerts(results: pd.DataFrame) -> None:
    """ส่ง alert ถ้าพบ regression"""
    regressions = results[results['regression'] == True]

    if regressions.empty:
        return

    for _, row in regressions.iterrows():
        message = f"""
⚠️ Performance Regression Detected!

Deployment: {row['deployment_id']}
Service: {row['service']} v{row['version']}
Metric: {row['metric']}
Change: +{row['change_pct']:.1f}% worse

Baseline p50: {row['baseline_p50']}ms
Post-deploy p50: {row['post_deploy_p50']}ms
"""
        send_slack_message('#perf-regressions', message)
```

---

## 74.8 Feedback Dashboard

### Real-time Feedback Dashboard

```typescript
// dashboard/FeedbackDashboard.tsx
import React, { useEffect, useState } from 'react';

interface FeedbackStats {
  errorRate: number;
  errorRateChange: number;
  p95Latency: number;
  p95LatencyChange: number;
  userSatisfaction: number;
  featureAdoptionRate: number;
  activeUsers: number;
}

export function FeedbackDashboard() {
  const [stats, setStats] = useState<FeedbackStats | null>(null);
  const [deployments, setDeployments] = useState([]);
  const [recentErrors, setRecentErrors] = useState([]);

  // Real-time updates ผ่าน WebSocket
  useEffect(() => {
    const ws = new WebSocket('/ws/feedback');

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);

      switch (data.type) {
        case 'stats_update':
          setStats(data.stats);
          break;
        case 'new_error':
          setRecentErrors(prev => [data.error, ...prev].slice(0, 20));
          break;
        case 'deployment':
          setDeployments(prev => [data.deployment, ...prev].slice(0, 10));
          break;
      }
    };

    return () => ws.close();
  }, []);

  if (!stats) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      {/* Key Metrics */}
      <div className="metrics-row">
        <MetricCard
          title="Error Rate"
          value={`${stats.errorRate.toFixed(2)}%`}
          change={stats.errorRateChange}
          isGoodWhenDecreasing
        />
        <MetricCard
          title="P95 Latency"
          value={`${stats.p95Latency}ms`}
          change={stats.p95LatencyChange}
          isGoodWhenDecreasing
        />
        <MetricCard
          title="User Satisfaction"
          value={`${stats.userSatisfaction}/10`}
          change={0}
        />
        <MetricCard
          title="Feature Adoption"
          value={`${stats.featureAdoptionRate.toFixed(1)}%`}
          change={0}
        />
      </div>

      {/* Recent Deployments */}
      <div className="section">
        <h3>Recent Deployments</h3>
        <DeploymentTimeline deployments={deployments} />
      </div>

      {/* Recent Errors */}
      <div className="section">
        <h3>Recent Errors</h3>
        <ErrorList errors={recentErrors} />
      </div>
    </div>
  );
}
```

---

## 74.9 แบบฝึกหัด

### แบบฝึกหัดที่ 1: ติดตั้ง Sentry

```bash
# 1. สร้าง Sentry account ที่ sentry.io
# 2. สร้าง project ใหม่
# 3. ติดตั้ง SDK ในแอปของคุณ

# Node.js
npm install @sentry/node

# สร้างไฟล์ instrument.js
cat > instrument.js << 'EOF'
const Sentry = require("@sentry/node");

Sentry.init({
  dsn: "YOUR_SENTRY_DSN",
  tracesSampleRate: 1.0,
});
EOF

# เพิ่มใน package.json scripts:
# "start": "node --require ./instrument.js app.js"
```

### แบบฝึกหัดที่ 2: สร้าง Feature Tracking

```typescript
// exercises/feature-tracking.ts
// สร้าง simple feature tracking สำหรับ React app

// 1. สร้าง analytics context
// 2. สร้าง useTrack hook
// 3. Instrument 3 components ด้วย tracking
// 4. สร้าง dashboard แสดง feature usage

// Template:
import React, { createContext, useContext, useCallback } from 'react';

interface AnalyticsContextType {
  track: (event: string, properties?: object) => void;
}

const AnalyticsContext = createContext<AnalyticsContextType>({
  track: () => {},
});

export function AnalyticsProvider({ children }: { children: React.ReactNode }) {
  const track = useCallback((event: string, properties?: object) => {
    // TODO: implement - ส่งไปยัง analytics service
    console.log('Track:', event, properties);
  }, []);

  return (
    <AnalyticsContext.Provider value={{ track }}>
      {children}
    </AnalyticsContext.Provider>
  );
}

export const useTrack = () => useContext(AnalyticsContext);
```

### แบบฝึกหัดที่ 3: Deployment Impact Analysis

```python
# exercises/deployment-impact.py
# วิเคราะห์ผลกระทบของ deployment ต่อ metrics

import random
from datetime import datetime, timedelta
import pandas as pd

# สร้างข้อมูลตัวอย่าง
def generate_sample_data():
    """สร้างข้อมูล metrics สำหรับฝึก"""
    data = []
    base_time = datetime.now() - timedelta(days=7)

    # ข้อมูลก่อน deployment (ปกติ)
    for i in range(1000):
        data.append({
            'timestamp': base_time + timedelta(hours=i/24),
            'lcp_ms': random.normalvariate(2500, 500),
            'fid_ms': random.normalvariate(100, 20),
            'error_rate': random.normalvariate(0.5, 0.1),
        })

    # จำลอง deployment เกิดขึ้น (3 วันที่แล้ว)
    deploy_time = datetime.now() - timedelta(days=3)

    # ข้อมูลหลัง deployment (regression)
    for i in range(500):
        data.append({
            'timestamp': deploy_time + timedelta(hours=i/24),
            'lcp_ms': random.normalvariate(3500, 500),  # worse!
            'fid_ms': random.normalvariate(150, 30),     # worse!
            'error_rate': random.normalvariate(0.5, 0.1),  # same
        })

    return pd.DataFrame(data)

# TODO: วิเคราะห์ว่ามี regression หรือไม่
df = generate_sample_data()
print("ข้อมูลทั้งหมด:")
print(df.describe())

# ค้นหา deployment ที่ทำให้เกิด regression
deployments = [
    {
        'id': 'deploy-001',
        'service': 'frontend',
        'version': 'v2.0.0',
        'deployed_at': datetime.now() - timedelta(days=3),
    }
]

# TODO: เรียกใช้ analyze_deployment_performance_impact
```

### แบบฝึกหัดที่ 4: สร้าง Feedback Loop

```yaml
# exercises/feedback-loop-design.yaml
# ออกแบบ Continuous Feedback Loop สำหรับ feature ของคุณ

feature_name: "ระบบชำระเงินใหม่"

feedback_sources:
  - name: Error Tracking
    tool: Sentry
    metrics:
      - error_rate
      - p95_latency
    alert_threshold:
      error_rate: "> 1%"
      p95_latency: "> 3000ms"

  - name: User Feedback
    tool: In-app widget
    metrics:
      - satisfaction_score
      - nps_score
    collection_trigger: "after successful payment"

  - name: Feature Analytics
    tool: Segment
    metrics:
      - adoption_rate
      - completion_rate
      - drop_off_step
    success_threshold:
      adoption_rate: "> 60%"
      completion_rate: "> 80%"

  - name: Performance
    tool: Web Vitals
    metrics:
      - lcp
      - fid
      - cls
    budget:
      lcp: "< 2500ms"
      fid: "< 100ms"
      cls: "< 0.1"

feedback_actions:
  - trigger: "error_rate > 2%"
    action: "create PagerDuty incident"
    owner: "on-call engineer"

  - trigger: "adoption_rate < 30% after 2 weeks"
    action: "schedule UX review meeting"
    owner: "product team"

  - trigger: "nps_score < 6"
    action: "send to customer success team"
    owner: "CS team"

reporting:
  frequency: weekly
  recipients:
    - product-team@company.com
    - engineering-leads@company.com
  format: dashboard + email summary
```

---

## สรุป

Continuous Feedback Loops ช่วยให้ทีมตัดสินใจได้ดีขึ้นโดยอาศัยข้อมูลจริงจาก production:

1. **Error Tracking** (Sentry) ช่วยรู้ปัญหาเร็วขึ้น
2. **Feature Analytics** ช่วยเข้าใจว่า feature ถูกใช้จริงหรือไม่
3. **User Feedback** ช่วยเข้าใจ pain points ของ user
4. **Performance Monitoring** ช่วย detect regression หลัง deployment
5. **Deployment Annotations** เชื่อมโยง deployments กับ metrics changes

ผลลัพธ์: ทีมสามารถ iterate ได้เร็วขึ้น โดยมีข้อมูลสนับสนุนการตัดสินใจ

### ขั้นตอนถัดไป

- ศึกษา [Part 75: Distributed Tracing](./part-75-distributed-tracing.md)
- Sentry documentation: https://docs.sentry.io
- Web Vitals: https://web.dev/vitals
