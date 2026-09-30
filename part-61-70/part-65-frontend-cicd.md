# Part 65: Frontend CI/CD

## บทนำ: ความซับซ้อนของ Frontend CI/CD

Frontend development สมัยใหม่ไม่ใช่แค่เขียน HTML/CSS/JS อีกต่อไป ความซับซ้อนของ build process, การทดสอบ visual regression, performance budgets, และ CDN management ทำให้ Frontend CI/CD เป็นเรื่องที่ต้องออกแบบอย่างรอบคอบ

### Frontend CI/CD Challenges

1. **Build Time**: Next.js/React apps อาจใช้เวลา build นาน
2. **Bundle Size**: ต้องควบคุมไม่ให้ bundle ใหญ่เกินไป
3. **Visual Regression**: UI เปลี่ยนโดยไม่ตั้งใจ
4. **Preview Deployments**: ทุก PR ควรมี preview URL
5. **CDN Invalidation**: หลัง deploy ต้อง clear cache
6. **SEO**: Verify meta tags, OG tags, structured data

---

## 1. Next.js CI/CD

### 1.1 Next.js Build Configuration

```javascript
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  // Output mode
  output: process.env.NEXT_OUTPUT_MODE || 'standalone', // หรือ 'export' สำหรับ static
  
  // Image optimization
  images: {
    domains: ['cdn.company.com', 'images.unsplash.com'],
    formats: ['image/avif', 'image/webp'],
    minimumCacheTTL: 60 * 60 * 24 * 7, // 7 days
  },
  
  // Performance
  swcMinify: true,
  compiler: {
    removeConsole: process.env.NODE_ENV === 'production',
  },
  
  // Bundle analysis
  webpack: (config, { isServer, dev }) => {
    if (!dev && !isServer && process.env.ANALYZE === 'true') {
      const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');
      config.plugins.push(
        new BundleAnalyzerPlugin({
          analyzerMode: 'static',
          reportFilename: '../analyze/client.html',
          openAnalyzer: false,
        })
      );
    }
    return config;
  },
  
  // Environment variables validation
  env: {
    APP_VERSION: process.env.NEXT_PUBLIC_APP_VERSION || 'dev',
    BUILD_TIME: new Date().toISOString(),
  },
  
  // Headers
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
          {
            key: 'Content-Security-Policy',
            value: [
              "default-src 'self'",
              "script-src 'self' 'unsafe-inline' 'unsafe-eval'",
              "style-src 'self' 'unsafe-inline' fonts.googleapis.com",
              "font-src 'self' fonts.gstatic.com",
              "img-src 'self' data: cdn.company.com",
              "connect-src 'self' api.company.com",
            ].join('; '),
          },
        ],
      },
      {
        // Cache static assets aggressively
        source: '/_next/static/(.*)',
        headers: [
          { key: 'Cache-Control', value: 'public, max-age=31536000, immutable' },
        ],
      },
    ];
  },
  
  // Redirects
  async redirects() {
    return [
      {
        source: '/old-path',
        destination: '/new-path',
        permanent: true,
      },
    ];
  },
};

module.exports = nextConfig;
```

### 1.2 Package.json Scripts

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "lint:fix": "next lint --fix",
    "type-check": "tsc --noEmit",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:ci": "jest --ci --coverage --watchAll=false",
    "e2e": "playwright test",
    "e2e:ci": "playwright test --reporter=github",
    "analyze": "ANALYZE=true next build",
    "storybook": "storybook dev -p 6006",
    "build-storybook": "storybook build",
    "chromatic": "chromatic --project-token=$CHROMATIC_PROJECT_TOKEN",
    "lighthouse": "lhci autorun",
    "bundle-size": "bundlesize"
  }
}
```

---

## 2. GitHub Actions สำหรับ Frontend

### 2.1 Complete Frontend Pipeline

```yaml
# .github/workflows/frontend.yml
name: Frontend CI/CD Pipeline

on:
  push:
    branches: [main, develop]
    paths:
      - 'frontend/**'
      - '.github/workflows/frontend.yml'
  pull_request:
    branches: [main, develop]
    paths: ['frontend/**']

env:
  NODE_VERSION: '20'
  PNPM_VERSION: '8'
  WORKING_DIR: ./frontend

jobs:
  # ==================== Lint & Type Check ====================
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: ${{ env.PNPM_VERSION }}
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
          cache-dependency-path: frontend/pnpm-lock.yaml
      
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
        working-directory: ${{ env.WORKING_DIR }}
      
      - name: TypeScript type check
        run: pnpm type-check
        working-directory: ${{ env.WORKING_DIR }}
      
      - name: ESLint
        run: pnpm lint
        working-directory: ${{ env.WORKING_DIR }}
      
      - name: Prettier check
        run: pnpm prettier --check "src/**/*.{ts,tsx,css,json}"
        working-directory: ${{ env.WORKING_DIR }}

  # ==================== Unit Tests ====================
  unit-test:
    name: Unit Tests
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: ${{ env.PNPM_VERSION }}
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
          cache-dependency-path: frontend/pnpm-lock.yaml
      
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
        working-directory: ${{ env.WORKING_DIR }}
      
      - name: Run unit tests
        run: pnpm test:ci
        working-directory: ${{ env.WORKING_DIR }}
        env:
          CI: true
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: frontend/coverage/lcov.info
          flags: frontend

  # ==================== Build ====================
  build:
    name: Build Next.js
    runs-on: ubuntu-latest
    needs: [quality, unit-test]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: ${{ env.PNPM_VERSION }}
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
          cache-dependency-path: frontend/pnpm-lock.yaml
      
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
        working-directory: ${{ env.WORKING_DIR }}
      
      - name: Build
        run: pnpm build
        working-directory: ${{ env.WORKING_DIR }}
        env:
          NEXT_PUBLIC_APP_VERSION: ${{ github.sha }}
          NEXT_PUBLIC_API_URL: ${{ vars.API_URL }}
      
      - name: Analyze bundle size
        run: pnpm analyze
        working-directory: ${{ env.WORKING_DIR }}
        env:
          NEXT_PUBLIC_API_URL: ${{ vars.API_URL }}
      
      - name: Check bundle size limits
        run: pnpm bundlesize
        working-directory: ${{ env.WORKING_DIR }}
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: nextjs-build-${{ github.sha }}
          path: frontend/.next/
          retention-days: 7
      
      - name: Upload bundle analysis
        uses: actions/upload-artifact@v4
        with:
          name: bundle-analysis
          path: frontend/analyze/

  # ==================== Visual Regression ====================
  chromatic:
    name: Visual Regression Tests
    runs-on: ubuntu-latest
    needs: build
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Required for Chromatic
      
      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: ${{ env.PNPM_VERSION }}
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
          cache-dependency-path: frontend/pnpm-lock.yaml
      
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
        working-directory: ${{ env.WORKING_DIR }}
      
      - name: Run Chromatic
        id: chromatic
        uses: chromaui/action@v1
        with:
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
          workingDir: frontend
          exitZeroOnChanges: true  # ไม่ fail CI แต่ review ใน Chromatic
          exitOnceUploaded: false
          autoAcceptChanges: "main"  # Auto-accept changes บน main branch
      
      - name: Comment PR with Chromatic results
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const storybookUrl = '${{ steps.chromatic.outputs.storybookUrl }}';
            const buildUrl = '${{ steps.chromatic.outputs.buildUrl }}';
            const changeCount = '${{ steps.chromatic.outputs.changeCount }}';
            const errorCount = '${{ steps.chromatic.outputs.errorCount }}';
            
            const body = `## 🎨 Visual Regression Tests
            
            | Metric | Value |
            |--------|-------|
            | 📸 UI Changes | ${changeCount} |
            | ❌ Errors | ${errorCount} |
            
            - [View Storybook](${storybookUrl})
            - [Review Changes in Chromatic](${buildUrl})
            
            ${changeCount > 0 ? '⚠️ Visual changes detected. Please review in Chromatic.' : '✅ No visual changes detected.'}
            `;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body
            });

  # ==================== E2E Tests ====================
  e2e:
    name: E2E Tests
    runs-on: ubuntu-latest
    needs: build
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: ${{ env.PNPM_VERSION }}
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'
          cache-dependency-path: frontend/pnpm-lock.yaml
      
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
        working-directory: ${{ env.WORKING_DIR }}
      
      - name: Install Playwright browsers
        run: pnpm exec playwright install --with-deps chromium firefox
        working-directory: ${{ env.WORKING_DIR }}
      
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: nextjs-build-${{ github.sha }}
          path: frontend/.next/
      
      - name: Start Next.js server
        run: |
          pnpm start &
          sleep 5
          curl -f http://localhost:3000 || exit 1
        working-directory: ${{ env.WORKING_DIR }}
        env:
          PORT: 3000
      
      - name: Run Playwright tests
        run: pnpm e2e:ci
        working-directory: ${{ env.WORKING_DIR }}
        env:
          PLAYWRIGHT_BASE_URL: http://localhost:3000
      
      - name: Upload Playwright report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: frontend/playwright-report/

  # ==================== Lighthouse CI ====================
  lighthouse:
    name: Lighthouse Performance
    runs-on: ubuntu-latest
    needs: build
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
      
      - name: Install LHCI
        run: npm install -g @lhci/cli
      
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: nextjs-build-${{ github.sha }}
          path: frontend/.next/
      
      - name: Run Lighthouse CI
        run: |
          cd frontend
          pnpm install --frozen-lockfile
          pnpm start &
          sleep 5
          lhci autorun
        env:
          LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}

  # ==================== Deploy Preview ====================
  deploy-preview:
    name: Deploy Preview
    runs-on: ubuntu-latest
    needs: [build, chromatic]
    if: github.event_name == 'pull_request'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Vercel (Preview)
        id: vercel-preview
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          working-directory: frontend
          github-token: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Comment PR with preview URL
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## 🚀 Preview Deployment\n\n**Preview URL**: ${{ steps.vercel-preview.outputs.preview-url }}\n\nDeployment is ready for review!`
            });

  # ==================== Deploy Production ====================
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [build, chromatic, e2e, lighthouse]
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Vercel (Production)
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'
          working-directory: frontend
      
      - name: Purge CDN Cache
        run: |
          curl -X POST "https://api.cloudflare.com/client/v4/zones/${{ secrets.CLOUDFLARE_ZONE_ID }}/purge_cache" \
            -H "Authorization: Bearer ${{ secrets.CLOUDFLARE_API_TOKEN }}" \
            -H "Content-Type: application/json" \
            --data '{"purge_everything":true}'
      
      - name: Run post-deploy checks
        run: |
          sleep 30  # Wait for deployment to propagate
          
          # Check main pages
          for path in "/" "/products" "/about" "/contact"; do
            status=$(curl -s -o /dev/null -w "%{http_code}" "https://www.company.com${path}")
            if [ "$status" != "200" ]; then
              echo "ERROR: $path returned $status"
              exit 1
            fi
          done
          
          echo "All pages returned 200 OK"
```

---

## 3. Vercel และ Netlify Deployment

### 3.1 Vercel Configuration

```json
// vercel.json
{
  "version": 2,
  "name": "my-nextjs-app",
  
  "builds": [
    {
      "src": "package.json",
      "use": "@vercel/next"
    }
  ],
  
  "routes": [
    {
      "src": "/api/(.*)",
      "dest": "/api/$1",
      "headers": {
        "cache-control": "no-store"
      }
    },
    {
      "src": "/(.*)",
      "dest": "/$1"
    }
  ],
  
  "env": {
    "DATABASE_URL": "@database-url",
    "API_SECRET": "@api-secret"
  },
  
  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        {
          "key": "Access-Control-Allow-Origin",
          "value": "https://www.company.com"
        }
      ]
    }
  ],
  
  "redirects": [
    {
      "source": "/blog/(.*)",
      "destination": "/articles/$1",
      "permanent": true
    }
  ],
  
  "regions": ["sin1"],
  
  "framework": "nextjs",
  
  "functions": {
    "api/**/*.ts": {
      "memory": 512,
      "maxDuration": 10
    }
  }
}
```

### 3.2 Netlify Configuration

```toml
# netlify.toml
[build]
  command = "pnpm build"
  publish = "out"  # สำหรับ static export
  
[build.environment]
  NODE_VERSION = "20"
  PNPM_VERSION = "8"
  NEXT_TELEMETRY_DISABLED = "1"

# Production context
[context.production]
  command = "pnpm build"
  environment = { NEXT_PUBLIC_ENV = "production", NEXT_PUBLIC_API_URL = "https://api.company.com" }

# Deploy Preview (PR)
[context.deploy-preview]
  command = "pnpm build"
  environment = { NEXT_PUBLIC_ENV = "preview", NEXT_PUBLIC_API_URL = "https://staging-api.company.com" }

# Branch deploys
[context.develop]
  command = "pnpm build"
  environment = { NEXT_PUBLIC_ENV = "develop" }

# Headers
[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-Content-Type-Options = "nosniff"
    
[[headers]]
  for = "/static/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"

# Redirects
[[redirects]]
  from = "/api/*"
  to = "https://api.company.com/:splat"
  status = 200

# Edge Functions (Netlify)
[[edge_functions]]
  path = "/api/auth/*"
  function = "auth"
```

---

## 4. Visual Regression Testing ด้วย Chromatic

### 4.1 Storybook Setup

```typescript
// .storybook/main.ts
import type { StorybookConfig } from '@storybook/nextjs';

const config: StorybookConfig = {
  stories: ['../src/**/*.stories.@(js|jsx|ts|tsx|mdx)'],
  
  addons: [
    '@storybook/addon-links',
    '@storybook/addon-essentials',
    '@storybook/addon-interactions',
    '@storybook/addon-a11y',
    'storybook-addon-designs',
  ],
  
  framework: {
    name: '@storybook/nextjs',
    options: {},
  },
  
  staticDirs: ['../public'],
  
  docs: {
    autodocs: 'tag',
  },
};

export default config;
```

### 4.2 Writing Stories

```typescript
// src/components/ProductCard/ProductCard.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { ProductCard } from './ProductCard';

const meta: Meta<typeof ProductCard> = {
  title: 'Components/ProductCard',
  component: ProductCard,
  
  // Tag สำหรับ auto-docs
  tags: ['autodocs'],
  
  // Default args
  args: {
    id: '1',
    name: 'Premium Headphones',
    price: 2990,
    currency: 'THB',
    imageUrl: 'https://via.placeholder.com/300x300',
    rating: 4.5,
    reviewCount: 128,
  },
  
  // Parameters
  parameters: {
    // Layout: centered, fullscreen, padded
    layout: 'centered',
    
    // Design link (Figma)
    design: {
      type: 'figma',
      url: 'https://www.figma.com/file/...',
    },
    
    // Chromatic viewport configuration
    chromatic: {
      viewports: [320, 768, 1200],
    },
  },
};

export default meta;
type Story = StoryObj<typeof ProductCard>;

// ==================== Stories ====================

export const Default: Story = {};

export const OutOfStock: Story = {
  args: {
    inStock: false,
  },
};

export const OnSale: Story = {
  args: {
    originalPrice: 3990,
    price: 2990,
    discountPercent: 25,
  },
};

export const LongProductName: Story = {
  args: {
    name: 'Sony WH-1000XM5 Wireless Noise Canceling Headphones - สีดำ พร้อมระบบตัดเสียงรบกวน',
  },
};

export const MobileView: Story = {
  parameters: {
    viewport: {
      defaultViewport: 'mobile1',
    },
    chromatic: {
      viewports: [320],
    },
  },
};

export const DarkMode: Story = {
  parameters: {
    backgrounds: { default: 'dark' },
    chromatic: {
      modes: {
        dark: { backgrounds: { default: 'dark' } },
        light: { backgrounds: { default: 'light' } },
      },
    },
  },
};
```

### 4.3 Chromatic Configuration

```javascript
// chromatic.config.js
module.exports = {
  projectId: 'your-project-id',
  
  // Automatically accept changes on main branch
  autoAcceptChanges: 'main',
  
  // Exit code on changes (สำหรับ CI)
  exitZeroOnChanges: true,
  
  // Skip stories with certain parameters
  skip: 'skip-chromatic',
  
  // Ignore minor pixel differences
  diffThreshold: 0.063,
  
  // Viewport
  viewports: [375, 768, 1280],
  
  // Turbosnap: only build/test changed components
  onlyChanged: true,
  
  // Tracing changes across commits
  traceChanged: true,
};
```

---

## 5. Bundle Size Analysis

### 5.1 Bundle Size Configuration

```json
// .bundlesizerc.json
{
  "files": [
    {
      "path": ".next/static/chunks/pages/index*.js",
      "maxSize": "100 kB",
      "compression": "gzip"
    },
    {
      "path": ".next/static/chunks/pages/_app*.js",
      "maxSize": "150 kB",
      "compression": "gzip"
    },
    {
      "path": ".next/static/css/*.css",
      "maxSize": "50 kB",
      "compression": "gzip"
    }
  ]
}
```

### 5.2 Next.js Bundle Analyzer

```javascript
// scripts/analyze-bundle.js
const { execSync } = require('child_process');
const fs = require('fs');
const path = require('path');

async function analyzeBundles() {
  // Build ด้วย ANALYZE flag
  execSync('ANALYZE=true next build', { stdio: 'inherit' });
  
  // Read build manifest
  const buildManifest = JSON.parse(
    fs.readFileSync('.next/build-manifest.json', 'utf-8')
  );
  
  // Calculate sizes
  const chunks = {};
  
  for (const [page, assets] of Object.entries(buildManifest.pages)) {
    let totalSize = 0;
    
    for (const asset of assets) {
      const assetPath = path.join('.next', asset);
      if (fs.existsSync(assetPath)) {
        const stats = fs.statSync(assetPath);
        totalSize += stats.size;
      }
    }
    
    chunks[page] = {
      size: totalSize,
      sizeKB: (totalSize / 1024).toFixed(2),
      sizeMB: (totalSize / 1024 / 1024).toFixed(3)
    };
  }
  
  // Sort by size
  const sorted = Object.entries(chunks)
    .sort(([, a], [, b]) => b.size - a.size);
  
  console.log('\n📦 Bundle Size Analysis\n');
  console.log('Page'.padEnd(40) + 'Size (gzip)');
  console.log('-'.repeat(60));
  
  for (const [page, info] of sorted) {
    console.log(`${page.padEnd(40)}${info.sizeKB} KB`);
  }
  
  // Check against budget
  const budget = {
    '/': 100,       // 100 KB
    '/_app': 150,   // 150 KB
    '/products': 80  // 80 KB
  };
  
  let exceededBudget = false;
  
  for (const [page, maxSize] of Object.entries(budget)) {
    const chunk = chunks[page];
    if (chunk && parseFloat(chunk.sizeKB) > maxSize) {
      console.error(`❌ ${page}: ${chunk.sizeKB} KB exceeds budget of ${maxSize} KB`);
      exceededBudget = true;
    }
  }
  
  if (exceededBudget) {
    process.exit(1);
  }
  
  console.log('\n✅ All pages within budget!');
}

analyzeBundles().catch(console.error);
```

---

## 6. Lighthouse CI Configuration

### 6.1 lighthouserc.js

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      numberOfRuns: 3,
      startServerCommand: 'pnpm start',
      url: [
        'http://localhost:3000',
        'http://localhost:3000/products',
        'http://localhost:3000/about',
      ],
      settings: {
        // Simulate mobile device
        formFactor: 'mobile',
        throttling: {
          rttMs: 40,
          throughputKbps: 10240,
          cpuSlowdownMultiplier: 4,
        },
        screenEmulation: {
          mobile: true,
          width: 375,
          height: 667,
          deviceScaleFactor: 2,
        },
      },
    },
    
    assert: {
      preset: 'lighthouse:recommended',
      assertions: {
        // Performance
        'categories:performance': ['error', { minScore: 0.85 }],
        'categories:accessibility': ['error', { minScore: 0.90 }],
        'categories:best-practices': ['error', { minScore: 0.90 }],
        'categories:seo': ['error', { minScore: 0.90 }],
        
        // Specific metrics
        'first-contentful-paint': ['warn', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 3000 }],
        'total-blocking-time': ['warn', { maxNumericValue: 300 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'speed-index': ['warn', { maxNumericValue: 3500 }],
        
        // Avoid
        'uses-http2': 'warn',
        'uses-long-cache-ttl': 'warn',
        'efficient-animated-content': 'error',
      },
    },
    
    upload: {
      target: 'lhci',
      serverBaseUrl: process.env.LHCI_SERVER_URL,
      token: process.env.LHCI_BUILD_TOKEN,
    },
  },
};
```

---

## 7. Testing Strategies สำหรับ Frontend

### 7.1 Component Testing ด้วย React Testing Library

```typescript
// src/components/ProductCard/__tests__/ProductCard.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { ProductCard } from '../ProductCard';

const defaultProps = {
  id: '1',
  name: 'Test Product',
  price: 1999,
  currency: 'THB',
  imageUrl: '/test-image.jpg',
  rating: 4.5,
  reviewCount: 100,
};

describe('ProductCard', () => {
  it('renders product information correctly', () => {
    render(<ProductCard {...defaultProps} />);
    
    expect(screen.getByText('Test Product')).toBeInTheDocument();
    expect(screen.getByText('฿1,999')).toBeInTheDocument();
    expect(screen.getByAltText('Test Product')).toBeInTheDocument();
    expect(screen.getByText('100 reviews')).toBeInTheDocument();
  });
  
  it('shows out of stock badge when not in stock', () => {
    render(<ProductCard {...defaultProps} inStock={false} />);
    
    expect(screen.getByText('Out of Stock')).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /add to cart/i })).toBeDisabled();
  });
  
  it('shows discount when originalPrice is provided', () => {
    render(
      <ProductCard
        {...defaultProps}
        originalPrice={2999}
        discountPercent={33}
      />
    );
    
    expect(screen.getByText('฿2,999')).toHaveClass('line-through');
    expect(screen.getByText('-33%')).toBeInTheDocument();
  });
  
  it('calls onAddToCart when button clicked', async () => {
    const onAddToCart = jest.fn();
    const user = userEvent.setup();
    
    render(<ProductCard {...defaultProps} onAddToCart={onAddToCart} />);
    
    await user.click(screen.getByRole('button', { name: /add to cart/i }));
    
    expect(onAddToCart).toHaveBeenCalledWith({
      id: '1',
      quantity: 1
    });
  });
  
  it('shows loading state while adding to cart', async () => {
    const onAddToCart = jest.fn().mockImplementation(
      () => new Promise(resolve => setTimeout(resolve, 100))
    );
    
    const user = userEvent.setup();
    render(<ProductCard {...defaultProps} onAddToCart={onAddToCart} />);
    
    await user.click(screen.getByRole('button', { name: /add to cart/i }));
    
    expect(screen.getByRole('button')).toBeDisabled();
    
    await waitFor(() => {
      expect(screen.getByRole('button')).not.toBeDisabled();
    });
  });
  
  it('meets accessibility standards', async () => {
    const { container } = render(<ProductCard {...defaultProps} />);
    const results = await axe(container);
    
    expect(results).toHaveNoViolations();
  });
});
```

### 7.2 E2E Testing ด้วย Playwright

```typescript
// e2e/product-search.spec.ts
import { test, expect, Page } from '@playwright/test';

test.describe('Product Search', () => {
  
  test.beforeEach(async ({ page }) => {
    await page.goto('/products');
    await page.waitForLoadState('networkidle');
  });
  
  test('should search for products', async ({ page }) => {
    // Type search query
    await page.fill('[data-testid="search-input"]', 'headphones');
    await page.press('[data-testid="search-input"]', 'Enter');
    
    // Wait for results
    await page.waitForSelector('[data-testid="product-card"]');
    
    const results = await page.locator('[data-testid="product-card"]').count();
    expect(results).toBeGreaterThan(0);
    
    // Verify all results contain search term
    const titles = await page.locator('[data-testid="product-name"]').allTextContents();
    expect(titles.every(t => t.toLowerCase().includes('headphone'))).toBeTruthy();
  });
  
  test('should filter by price range', async ({ page }) => {
    // Set price range
    await page.fill('[data-testid="min-price"]', '1000');
    await page.fill('[data-testid="max-price"]', '3000');
    await page.click('[data-testid="apply-filters"]');
    
    await page.waitForLoadState('networkidle');
    
    // Verify prices are within range
    const prices = await page.locator('[data-testid="product-price"]').allTextContents();
    
    for (const priceText of prices) {
      const price = parseFloat(priceText.replace(/[^0-9.]/g, ''));
      expect(price).toBeGreaterThanOrEqual(1000);
      expect(price).toBeLessThanOrEqual(3000);
    }
  });
  
  test('should add product to cart', async ({ page }) => {
    // Click on first product
    await page.click('[data-testid="product-card"]:first-child');
    
    // Wait for product detail page
    await page.waitForURL(/\/products\/\d+/);
    
    // Add to cart
    await page.click('[data-testid="add-to-cart"]');
    
    // Verify cart updated
    await expect(page.locator('[data-testid="cart-count"]')).toHaveText('1');
  });
  
  test('should work on mobile viewport', async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 667 });
    
    // Mobile menu
    await page.click('[data-testid="mobile-menu-button"]');
    await expect(page.locator('[data-testid="mobile-menu"]')).toBeVisible();
    
    // Screenshot
    await expect(page).toHaveScreenshot('mobile-product-search.png');
  });
});
```

---

## 8. CDN Invalidation Strategy

### 8.1 Cloudflare Purge

```javascript
// scripts/cdn-invalidate.js
const https = require('https');

async function purgeCloudflareCache(paths = []) {
  const zoneId = process.env.CLOUDFLARE_ZONE_ID;
  const apiToken = process.env.CLOUDFLARE_API_TOKEN;
  
  const body = paths.length > 0
    ? JSON.stringify({ files: paths.map(p => `https://www.company.com${p}`) })
    : JSON.stringify({ purge_everything: true });
  
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${zoneId}/purge_cache`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${apiToken}`,
        'Content-Type': 'application/json',
      },
      body,
    }
  );
  
  const result = await response.json();
  
  if (!result.success) {
    throw new Error(`CDN purge failed: ${JSON.stringify(result.errors)}`);
  }
  
  console.log('CDN cache purged successfully');
  return result;
}

// Selective purge สำหรับ specific pages
async function purgeChangedPages(changedFiles) {
  // Map changed files to affected routes
  const affectedRoutes = new Set();
  
  for (const file of changedFiles) {
    if (file.includes('src/pages/products')) {
      affectedRoutes.add('/products');
    }
    if (file.includes('src/components/Header')) {
      // Header is on all pages - purge everything
      return purgeCloudflareCache([]);
    }
    if (file.match(/src\/pages\/(.+)\.tsx?/)) {
      const route = file.match(/src\/pages\/(.+)\.tsx?/)[1]
        .replace(/\/index$/, '')
        .replace(/\[.*?\]/g, '*');
      affectedRoutes.add(`/${route}`);
    }
  }
  
  await purgeCloudflareCache([...affectedRoutes]);
}
```

---

## 9. แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Next.js Full Pipeline

```
Task: สร้าง complete CI/CD pipeline สำหรับ Next.js e-commerce site

Requirements:
1. GitHub Actions:
   - Lint + Type check
   - Unit tests + Coverage >= 80%
   - Build + Bundle size check
   - Visual regression ด้วย Chromatic
   - E2E ด้วย Playwright
   - Lighthouse >=85 performance

2. Preview deployments:
   - ทุก PR ต้องมี preview URL ด้วย Vercel
   - Comment PR ด้วย preview URL + metrics

3. Production deployment:
   - Main branch → Vercel production
   - Purge Cloudflare CDN หลัง deploy
   - Smoke tests หลัง deploy

Bundle size budgets:
- Home page: <= 120 KB (gzip)
- Product list: <= 100 KB (gzip)
- CSS: <= 30 KB (gzip)
```

### แบบฝึกหัดที่ 2: Visual Regression Setup

```
สร้าง Storybook สำหรับ component library:

1. สร้าง stories สำหรับ:
   - Button (default, primary, secondary, danger, disabled)
   - Input (text, password, error state)
   - Card (product card, info card)
   - Modal (confirmation, form)

2. Configure Chromatic:
   - Setup project
   - Run baseline build
   - Test ว่า change ใน CSS ถูก detect

3. Integration ใน CI:
   - Auto-accept บน main branch
   - Block PR ถ้า unreviewed changes
```

### แบบฝึกหัดที่ 3: Performance Budget Enforcement

```javascript
// สร้าง script ที่:
// 1. Build Next.js app
// 2. Measure bundle sizes
// 3. Compare กับ previous build (จาก git tag)
// 4. Fail ถ้า:
//    - Any page increases > 20%
//    - Any page exceeds absolute limit
// 5. Comment PR ด้วย size diff table

// ตัวอย่าง output:
// Page                   Old Size   New Size   Diff
// /                      95 KB      98 KB      +3%  ✅
// /products              110 KB     135 KB     +23% ❌ EXCEEDS 20% INCREASE
```

### สรุปบทที่ 65

ในบทนี้เราได้เรียนรู้:
- **Next.js Build**: Configuration, output modes, optimization
- **GitHub Actions**: Complete frontend pipeline พร้อม parallel jobs
- **Vercel/Netlify**: Preview deployments และ production deployment
- **Chromatic**: Visual regression testing ด้วย Storybook
- **Bundle Analysis**: การ monitor และ enforce bundle size budgets
- **Lighthouse CI**: Automated performance testing ใน CI
- **CDN Invalidation**: Cloudflare cache purge หลัง deploy
- **Playwright**: E2E testing สำหรับ frontend

บทถัดไปเราจะเรียนรู้ Infrastructure Testing ด้วย Terratest
