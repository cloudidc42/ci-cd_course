# Part 96: Case Study — Gaming Platform CI/CD

## บทนำ

Gaming platform มีความท้าทายพิเศษด้าน CI/CD เพราะต้องรองรับผู้เล่นหลายล้านคน patch game โดยไม่ขัดกิจกรรมผู้เล่น และ rollout features ใหม่โดยไม่กระทบ game balance บทนี้ศึกษา case study ของ **"ArenaX"** — online multiplayer gaming platform ที่มีผู้เล่นกว่า 10 ล้านคนในเอเชียตะวันออกเฉียงใต้

---

## 96.1 Context และ Requirements

### ข้อมูลองค์กร

```yaml
organization:
  name: "ArenaX Gaming Platform"
  games:
    - "ArenaX Battle Royale (30M downloads)"
    - "ArenaX RPG (5M players)"
    - "ArenaX Racing (3M players)"
    
  scale:
    total_players: 10_000_000
    concurrent_players_peak: 500_000
    daily_active_players: 2_000_000
    transactions_per_second_peak: 50_000
    
  infrastructure:
    game_servers: 2_000  # physical/virtual game servers
    regions: ["Thailand", "Singapore", "Philippines", "Vietnam", "Indonesia"]
    server_regions: 5
    data_centers: 3
    
  engineering:
    developers: 200
    game_devs: 120
    platform_engineers: 30
    qa_team: 50
    
  special_requirements:
    - "Player data must never be corrupted during update"
    - "Game state must be preserved across patches"
    - "Anti-cheat system must not be disrupted"
    - "Ranked season data is critical (no loss allowed)"
    - "Micro-transactions must work 24/7"
```

### Gaming-Specific Challenges

```
Gaming CI/CD ความท้าทายพิเศษ
═════════════════════════════

1. LIVE GAME UPDATES
   - ไม่สามารถหยุด server ระหว่าง match
   - Server update ต้อง wait ให้ match จบ
   - Player sessions ต้องถูก migrate หรือ preserve

2. GAME CLIENT DISTRIBUTION
   - Client updates ผ่าน app stores (review process)
   - Patch sizes ต้องเล็กที่สุด (mobile bandwidth)
   - Backward compatibility กับ old clients

3. ANTI-CHEAT CONCERNS
   - Anti-cheat update ต้อง deploy ก่อน cheat เผยแพร่
   - Emergency patch workflow
   - Confidential update (ไม่เปิดเผย mechanism)

4. GAME BALANCE
   - Feature flags สำหรับ balance changes
   - A/B testing สำหรับ economy changes
   - Rollback ต้องไม่กระทบ player inventory

5. SEASONAL EVENTS
   - Deploy ก่อน event เริ่ม
   - Auto-disable เมื่อ event จบ
   - Time-limited content management
```

---

## 96.2 Game Server Deployment Architecture

### Deployment Architecture

```
ArenaX Deployment Architecture
════════════════════════════════

┌─────────────────────────────────────────────────┐
│               Game Services                       │
│                                                   │
│  ┌──────────────┐  ┌──────────────┐             │
│  │  Matchmaking  │  │  Game State  │             │
│  │   Service    │  │   Service    │             │
│  └──────────────┘  └──────────────┘             │
│                                                   │
│  ┌──────────────┐  ┌──────────────┐             │
│  │ Game Server  │  │  Leaderboard │             │
│  │  Fleet (K8s) │  │   Service    │             │
│  └──────────────┘  └──────────────┘             │
│                                                   │
│  ┌──────────────┐  ┌──────────────┐             │
│  │  In-Game     │  │  Anti-Cheat  │             │
│  │  Store       │  │   Engine     │             │
│  └──────────────┘  └──────────────┘             │
└─────────────────────────────────────────────────┘

Deployment Strategies by Service Type:
┌─────────────────────────────────────────────────┐
│ Service Type        │ Strategy                  │
├─────────────────────┼───────────────────────────┤
│ Stateless services  │ Rolling update            │
│ Matchmaking         │ Blue/Green                │
│ Game servers        │ Graceful rolling update   │
│ Leaderboard         │ Blue/Green (no downtime)  │
│ Anti-cheat          │ Emergency fast-track      │
│ Game client         │ Progressive rollout       │
└─────────────────────┴───────────────────────────┘
```

### Game Server Rolling Update

```python
# game_server_deployment.py
# Deployment ที่ aware ของ game state

import asyncio
import time
from typing import List, Dict
from dataclasses import dataclass
from enum import Enum

class ServerState(Enum):
    ACTIVE = "active"           # มี active matches
    DRAINING = "draining"       # ไม่รับ match ใหม่
    IDLE = "idle"               # ไม่มี matches
    UPDATING = "updating"       # กำลัง update
    UPDATED = "updated"         # Update แล้ว

@dataclass
class GameServer:
    server_id: str
    region: str
    state: ServerState
    active_matches: int
    players_online: int
    version: str
    
    @property
    def can_be_updated(self) -> bool:
        return self.state == ServerState.IDLE and self.active_matches == 0

class GameServerFleetManager:
    """
    จัดการ fleet ของ game servers พร้อม game-aware deployment
    """
    
    def __init__(self, fleet_api_client):
        self.fleet = fleet_api_client
        self.update_in_progress = False
    
    async def rolling_update_fleet(self, new_version: str, region: str = None) -> dict:
        """
        Rolling update ทั้ง fleet โดย:
        1. Drain servers ทีละกลุ่ม (ไม่รับ match ใหม่)
        2. รอให้ matches ที่มีอยู่จบ
        3. Update
        4. Bring back online
        """
        
        if self.update_in_progress:
            raise Exception("Another update is already in progress")
        
        self.update_in_progress = True
        update_stats = {
            'total_servers': 0,
            'updated': 0,
            'failed': 0,
            'start_time': time.time()
        }
        
        try:
            servers = await self.fleet.get_servers(region=region)
            update_stats['total_servers'] = len(servers)
            
            print(f"🎮 Starting fleet update to {new_version}")
            print(f"   Servers to update: {len(servers)}")
            
            # แบ่ง servers เป็น batches (20% ต่อครั้ง)
            batch_size = max(1, len(servers) // 5)
            batches = [servers[i:i+batch_size] for i in range(0, len(servers), batch_size)]
            
            for batch_num, batch in enumerate(batches, 1):
                print(f"\n📦 Processing batch {batch_num}/{len(batches)} ({len(batch)} servers)")
                
                # Phase 1: Drain batch (หยุดรับ matches ใหม่)
                await self.drain_servers(batch)
                
                # Phase 2: รอให้ matches จบ (หรือ timeout)
                drained = await self.wait_for_drain(batch, timeout=1800)  # 30 min timeout
                
                if not drained:
                    print(f"⚠️ Timeout waiting for drain on batch {batch_num}, forcing update on idle servers")
                
                # Phase 3: Update drained servers
                for server in batch:
                    if server.state in [ServerState.IDLE, ServerState.DRAINING]:
                        try:
                            await self.update_server(server, new_version)
                            update_stats['updated'] += 1
                            print(f"   ✅ Updated {server.server_id}")
                        except Exception as e:
                            update_stats['failed'] += 1
                            print(f"   ❌ Failed to update {server.server_id}: {e}")
                
                # Phase 4: Verify batch health
                if not await self.verify_batch_health(batch):
                    print(f"❌ Batch {batch_num} health check failed! Rolling back batch...")
                    await self.rollback_batch(batch)
                    raise Exception(f"Fleet update failed at batch {batch_num}")
                
                print(f"✅ Batch {batch_num} updated successfully")
                
                # รอ 2 นาทีก่อน batch ถัดไป (observe behavior)
                if batch_num < len(batches):
                    print(f"   Waiting 2 minutes before next batch...")
                    await asyncio.sleep(120)
            
            update_stats['duration'] = time.time() - update_stats['start_time']
            print(f"\n🎉 Fleet update complete!")
            print(f"   Updated: {update_stats['updated']}/{update_stats['total_servers']}")
            print(f"   Duration: {update_stats['duration']:.0f}s")
            
            return update_stats
            
        finally:
            self.update_in_progress = False
    
    async def drain_servers(self, servers: List[GameServer]):
        """Drain: หยุดรับ matches ใหม่"""
        for server in servers:
            await self.fleet.set_server_state(server.server_id, ServerState.DRAINING)
            print(f"   🚫 {server.server_id}: Draining (no new matches)")
    
    async def wait_for_drain(self, servers: List[GameServer], timeout: int = 1800) -> bool:
        """รอให้ servers drain (matches จบทั้งหมด)"""
        start = time.time()
        
        while time.time() - start < timeout:
            still_active = []
            for server in servers:
                refreshed = await self.fleet.get_server(server.server_id)
                if refreshed.active_matches > 0:
                    still_active.append(f"{refreshed.server_id} ({refreshed.active_matches} matches)")
            
            if not still_active:
                return True
            
            print(f"   ⏳ Waiting for matches to finish: {', '.join(still_active)}")
            await asyncio.sleep(30)
        
        return False
    
    async def update_server(self, server: GameServer, new_version: str):
        """Update single game server"""
        await self.fleet.update_server(
            server_id=server.server_id,
            version=new_version,
            strategy='in-place'  # หรือ 'replace' สำหรับ major updates
        )
        
        # รอให้ server ready
        await self.wait_server_ready(server.server_id)
        
        # Bring back online
        await self.fleet.set_server_state(server.server_id, ServerState.ACTIVE)
    
    async def verify_batch_health(self, servers: List[GameServer]) -> bool:
        """ตรวจสอบ health หลัง update"""
        await asyncio.sleep(60)  # รอ 1 นาทีหลัง update
        
        for server in servers:
            refreshed = await self.fleet.get_server(server.server_id)
            if refreshed.state != ServerState.ACTIVE:
                return False
        
        return True
```

---

## 96.3 Game Client Distribution

### Client Update Pipeline

```yaml
# .github/workflows/game-client-pipeline.yml
# Pipeline สำหรับ game client builds

name: Game Client CI/CD

on:
  push:
    branches: [main, 'release/**']
    tags: ['v*']
  pull_request:
    branches: [main]

jobs:
  # ===== BUILD =====
  build-android:
    name: Build Android Client
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          lfs: true  # Game assets ใหญ่ใช้ Git LFS

      - name: Setup Unity
        uses: game-ci/unity-builder@v4
        with:
          unity-version: ${{ env.UNITY_VERSION }}
          target-platform: Android
          build-method: BuildScript.BuildAndroid
          versioning: Semantic
        env:
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
          UNITY_EMAIL: ${{ secrets.UNITY_EMAIL }}
          UNITY_PASSWORD: ${{ secrets.UNITY_PASSWORD }}

      - name: Run Game Tests
        uses: game-ci/unity-test-runner@v4
        with:
          unity-version: ${{ env.UNITY_VERSION }}
          test-mode: All
          coverage-options: generateAdditionalMetrics;generateHtmlReport;generateBadgeReport

      - name: Sign APK
        run: |
          # Sign ด้วย release keystore
          jarsigner -verbose \
            -sigalg SHA1withRSA \
            -digestalg SHA1 \
            -keystore $ANDROID_KEYSTORE \
            -storepass $KEYSTORE_PASSWORD \
            build/ArenaX.apk \
            arenax-release
        env:
          ANDROID_KEYSTORE: ${{ secrets.ANDROID_KEYSTORE }}
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}

      - name: Calculate Patch Delta
        run: |
          # สร้าง delta patch จาก previous version
          # ทำให้ download size เล็กลง
          python scripts/create_delta_patch.py \
            --base-version ${{ env.PREVIOUS_VERSION }} \
            --new-version ${{ github.ref_name }} \
            --output patches/
          
          echo "Delta patch size: $(du -sh patches/*.patch | head -1)"

  build-ios:
    name: Build iOS Client
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
        with:
          lfs: true
          
      - name: Setup Unity
        uses: game-ci/unity-builder@v4
        with:
          unity-version: ${{ env.UNITY_VERSION }}
          target-platform: iOS
          
      - name: Build Xcode Project
        run: |
          cd build/iOS
          xcodebuild \
            -workspace Unity-iPhone.xcworkspace \
            -scheme Unity-iPhone \
            -configuration Release \
            -archivePath ArenaX.xcarchive \
            archive

      - name: Export IPA
        run: |
          xcodebuild -exportArchive \
            -archivePath ArenaX.xcarchive \
            -exportOptionsPlist ExportOptions.plist \
            -exportPath ./ipa
            
      - name: Upload to TestFlight
        run: |
          xcrun altool \
            --upload-app \
            --type ios \
            --file ipa/ArenaX.ipa \
            --username ${{ secrets.APPLE_ID }} \
            --password ${{ secrets.APPLE_APP_SPECIFIC_PASSWORD }}

  # ===== GAME CONTENT PIPELINE =====
  content-pipeline:
    name: Game Content Processing
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          lfs: true

      - name: Validate Game Assets
        run: |
          python scripts/validate_assets.py \
            --check-texture-sizes \
            --check-audio-formats \
            --check-3d-model-polycounts \
            --max-texture-size 2048 \
            --max-audio-duration 300

      - name: Compress Assets
        run: |
          python scripts/compress_assets.py \
            --input assets/ \
            --output compressed_assets/ \
            --formats "texture=etc2,audio=ogg,3d=draco"

      - name: Upload to CDN
        run: |
          aws s3 sync compressed_assets/ \
            s3://arenax-cdn/assets/${{ github.ref_name }}/ \
            --cache-control "max-age=86400"
          
          # Invalidate CDN cache
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CDN_DISTRIBUTION_ID }} \
            --paths "/assets/*"

  # ===== SERVER DEPLOYMENT =====
  deploy-game-servers:
    name: Deploy Game Servers
    runs-on: ubuntu-latest
    needs: [build-android, build-ios]
    if: startsWith(github.ref, 'refs/tags/')
    
    steps:
      - name: Check Active Player Count
        id: player-check
        run: |
          PLAYERS=$(curl -s https://metrics.arenax.internal/v1/active-players | jq '.count')
          echo "active_players=${PLAYERS}" >> $GITHUB_OUTPUT
          echo "Current active players: ${PLAYERS}"
          
      - name: Determine Deployment Window
        run: |
          # ตรวจสอบว่าเป็นช่วง off-peak หรือไม่ (02:00-06:00 ICT)
          HOUR=$(date -u +%-H)  # UTC hour
          ICT_HOUR=$(( (HOUR + 7) % 24 ))
          
          if [ $ICT_HOUR -ge 2 ] && [ $ICT_HOUR -le 6 ]; then
            echo "✅ Within deployment window (ICT ${ICT_HOUR}:00)"
          else
            echo "⚠️ Outside off-peak window. Consider scheduling."
            # ไม่ fail แต่ warn
          fi
          
      - name: Start Fleet Update
        run: |
          python scripts/fleet_update.py \
            --version ${{ github.ref_name }} \
            --regions all \
            --batch-size 20 \
            --wait-for-idle \
            --timeout 3600
```

---

## 96.4 Feature Flags สำหรับ Game Features

### Game Feature Flag System

```typescript
// game-feature-flags.ts
// Feature flag system ที่ optimize สำหรับ gaming

interface PlayerContext {
  playerId: string;
  level: number;
  region: string;
  platform: 'mobile' | 'pc' | 'console';
  isPremium: boolean;
  isInRankedSeason: boolean;
}

interface FeatureFlag {
  key: string;
  type: 'boolean' | 'percentage' | 'variant';
  value: any;
  rolloutPercentage?: number;
  targetAudience?: Partial<PlayerContext>;
}

class GameFeatureFlagService {
  private flags: Map<string, FeatureFlag> = new Map();
  
  async isEnabled(flagKey: string, player: PlayerContext): Promise<boolean> {
    const flag = await this.getFlag(flagKey);
    
    if (!flag) return false;
    
    // Check rollout percentage
    if (flag.rolloutPercentage !== undefined) {
      const playerHash = this.hashPlayer(player.playerId + flagKey);
      if (playerHash > flag.rolloutPercentage) {
        return false;
      }
    }
    
    // Check target audience
    if (flag.targetAudience) {
      if (!this.matchesAudience(player, flag.targetAudience)) {
        return false;
      }
    }
    
    return Boolean(flag.value);
  }
  
  async getVariant(flagKey: string, player: PlayerContext): Promise<string> {
    const flag = await this.getFlag(flagKey);
    if (!flag || flag.type !== 'variant') return 'control';
    
    // Consistent assignment - same player gets same variant
    const variants = flag.value as string[];
    const index = Math.abs(this.hashPlayer(player.playerId + flagKey)) % variants.length;
    return variants[index];
  }
  
  private hashPlayer(input: string): number {
    let hash = 0;
    for (let i = 0; i < input.length; i++) {
      const char = input.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash; // Convert to 32-bit integer
    }
    return Math.abs(hash) % 100;
  }
  
  private matchesAudience(player: PlayerContext, audience: Partial<PlayerContext>): boolean {
    return Object.entries(audience).every(([key, value]) => 
      player[key as keyof PlayerContext] === value
    );
  }
  
  private async getFlag(key: string): Promise<FeatureFlag | undefined> {
    // Cache ใน memory, refresh ทุก 5 นาที
    // Production: ดึงจาก Redis/feature flag service
    return this.flags.get(key);
  }
}

// ตัวอย่างการใช้งาน Game Feature
class BattleRoyaleGameMode {
  constructor(private featureFlags: GameFeatureFlagService) {}
  
  async getGameSettings(player: PlayerContext): Promise<object> {
    const settings: any = {
      mapSize: 'standard',
      playerCount: 100,
      weapons: 'standard'
    };
    
    // New map (rolling out to 10% ก่อน)
    if (await this.featureFlags.isEnabled('new-desert-map', player)) {
      settings.map = 'desert_zone';
    }
    
    // New weapon balance (testing กับ non-ranked players)
    if (!player.isInRankedSeason && 
        await this.featureFlags.isEnabled('new-weapon-balance', player)) {
      settings.weapons = 'balanced_v2';
    }
    
    // A/B test: Loot distribution algorithm
    const lootVariant = await this.featureFlags.getVariant('loot-distribution', player);
    settings.lootAlgorithm = lootVariant;
    
    return settings;
  }
}
```

---

## 96.5 Live Game Patching

### Emergency Patch Workflow

```python
# emergency_patch.py
# ระบบ emergency patching สำหรับ critical bugs หรือ exploits

from enum import Enum
from datetime import datetime
import subprocess

class PatchPriority(Enum):
    CRITICAL = "critical"    # Exploit/cheat ที่ทำลาย game economy
    HIGH = "high"            # Major bug กระทบ gameplay
    MEDIUM = "medium"        # Balance issue
    LOW = "low"              # Minor bug

class EmergencyPatchManager:
    """
    จัดการ emergency patches ที่ต้อง deploy ด่วน
    """
    
    def __init__(self, fleet_manager, notification_service, anti_cheat_team):
        self.fleet = fleet_manager
        self.notify = notification_service
        self.anti_cheat = anti_cheat_team
    
    async def deploy_emergency_patch(self, 
                                      patch_id: str,
                                      priority: PatchPriority,
                                      build_artifact: str) -> dict:
        """
        Fast-track deployment สำหรับ emergency patches
        """
        
        print(f"🚨 EMERGENCY PATCH DEPLOYMENT")
        print(f"   Patch ID: {patch_id}")
        print(f"   Priority: {priority.value}")
        
        start_time = datetime.now()
        
        # Step 1: Notify stakeholders
        await self.notify_stakeholders(patch_id, priority)
        
        # Step 2: Deploy based on priority
        if priority == PatchPriority.CRITICAL:
            # Critical: Deploy ด่วน แม้มี active players
            # Force drain + fast update
            result = await self.critical_deploy(build_artifact)
            
        elif priority == PatchPriority.HIGH:
            # High: Deploy ใน off-peak หรือ force drain
            result = await self.high_priority_deploy(build_artifact)
            
        else:
            # Medium/Low: ใช้ normal process
            result = await self.normal_deploy(build_artifact)
        
        duration = (datetime.now() - start_time).total_seconds()
        
        # Step 3: Verify patch success
        if result['success']:
            await self.verify_patch(patch_id)
            print(f"✅ Emergency patch deployed in {duration:.0f}s")
        else:
            await self.alert_patch_failure(patch_id, result)
        
        return {
            'patch_id': patch_id,
            'success': result['success'],
            'duration_seconds': duration,
            'affected_players': result.get('affected_players', 0)
        }
    
    async def critical_deploy(self, build_artifact: str) -> dict:
        """
        Critical patch: Deploy ทันที
        - Force disconnect players (แจ้งเตือนล่วงหน้า 5 นาที)
        - Update ทั้ง fleet พร้อมกัน
        """
        
        # แจ้งเตือน players ใน game
        active_players = await self.fleet.get_active_player_count()
        
        await self.notify.broadcast_ingame(
            message="⚠️ Emergency maintenance in 5 minutes. Please save progress.",
            duration=300  # 5 minutes
        )
        
        import asyncio
        await asyncio.sleep(300)  # รอ 5 นาที
        
        # Force close all matches (preserve player data)
        await self.preserve_player_data()
        await self.fleet.close_all_matches(grace_period=60)
        
        # Update all servers พร้อมกัน (emergency ยอมรับ downtime สั้น)
        success = await self.fleet.mass_update(build_artifact, timeout=600)
        
        return {
            'success': success,
            'affected_players': active_players,
            'strategy': 'critical_mass_update'
        }
    
    async def preserve_player_data(self):
        """บันทึก player state ก่อน emergency update"""
        print("💾 Preserving player data...")
        
        # Force save ของ players ทั้งหมด
        await self.fleet.broadcast_save_all_players()
        
        # รอให้ save สำเร็จ
        import asyncio
        await asyncio.sleep(30)
        
        # Verify saves
        failed_saves = await self.fleet.get_failed_saves()
        if failed_saves:
            print(f"⚠️ {len(failed_saves)} players had save failures")
            # Log สำหรับ manual recovery
            for player_id in failed_saves:
                await self.log_save_failure(player_id)
        
        print(f"✅ Player data preserved")

# Anti-Cheat Emergency Update
class AntiCheatDeployment:
    """
    Special deployment สำหรับ anti-cheat updates
    ต้อง deploy โดยไม่เปิดเผย mechanism
    """
    
    def __init__(self, secret_build_server):
        # Anti-cheat builds on isolated, air-gapped server
        self.build_server = secret_build_server
    
    async def deploy_silent_update(self, new_detection_rules: str) -> bool:
        """
        Deploy anti-cheat update โดยไม่แจ้ง changelog สาธารณะ
        """
        
        # Build บน isolated server
        build_result = await self.build_server.build(
            rules=new_detection_rules,
            obfuscate=True,  # Obfuscate detection logic
            strip_symbols=True
        )
        
        if not build_result['success']:
            return False
        
        # Deploy โดยใช้ signed package ที่ verify integrity
        deploy_result = await self.deploy_to_fleet(
            artifact=build_result['artifact'],
            signature=build_result['signature'],
            update_type='silent',  # ไม่แจ้งใน patch notes
            changelog_entry='System improvements'  # Generic description
        )
        
        return deploy_result['success']
```

---

## 96.6 Player Impact Monitoring

### Deployment Impact Dashboard

```python
# player_impact_monitor.py
# Monitor impact ของ deployment ต่อ player experience

class PlayerImpactMonitor:
    def __init__(self, metrics_client, alerting_client):
        self.metrics = metrics_client
        self.alerting = alerting_client
        
        # Thresholds ที่กำหนด
        self.thresholds = {
            'connection_error_rate': 0.005,   # 0.5%
            'match_crash_rate': 0.001,         # 0.1%
            'latency_p99_ms': 100,             # 100ms
            'login_failure_rate': 0.01,        # 1%
            'purchase_failure_rate': 0.001,    # 0.1% (critical!)
        }
    
    async def monitor_deployment(self, deployment_id: str, duration_minutes: int = 30) -> dict:
        """Monitor player experience หลัง deployment"""
        
        import asyncio
        
        start = datetime.now()
        alerts_triggered = []
        
        print(f"👁️ Monitoring player impact for {duration_minutes} minutes...")
        
        while (datetime.now() - start).seconds < duration_minutes * 60:
            metrics = await self.collect_metrics()
            
            # ตรวจสอบ threshold violations
            for metric_name, threshold in self.thresholds.items():
                current_value = metrics.get(metric_name, 0)
                
                if current_value > threshold:
                    alert = {
                        'metric': metric_name,
                        'current': current_value,
                        'threshold': threshold,
                        'timestamp': datetime.now().isoformat()
                    }
                    
                    if alert not in alerts_triggered:
                        alerts_triggered.append(alert)
                        
                        if metric_name == 'purchase_failure_rate':
                            # Purchase failures = immediate rollback trigger
                            await self.trigger_emergency_rollback(deployment_id, alert)
                        else:
                            await self.alerting.send_alert(
                                severity='high',
                                message=f"Player impact alert: {metric_name} = {current_value:.4f}"
                            )
            
            # แสดง real-time status
            print(f"\r📊 Active players: {metrics.get('active_players', 0):,} | "
                  f"Errors: {metrics.get('connection_error_rate', 0):.3%} | "
                  f"Latency p99: {metrics.get('latency_p99_ms', 0):.0f}ms | "
                  f"Alerts: {len(alerts_triggered)}", end='')
            
            await asyncio.sleep(60)
        
        print()  # New line
        
        return {
            'deployment_id': deployment_id,
            'monitoring_duration_minutes': duration_minutes,
            'alerts_triggered': alerts_triggered,
            'deployment_healthy': len(alerts_triggered) == 0
        }
    
    async def collect_metrics(self) -> dict:
        """เก็บ metrics จาก monitoring systems"""
        return {
            'active_players': await self.metrics.query('sum(active_players)'),
            'connection_error_rate': await self.metrics.query(
                'rate(connection_errors_total[5m]) / rate(connection_attempts_total[5m])'
            ),
            'match_crash_rate': await self.metrics.query(
                'rate(match_crashes_total[5m]) / rate(matches_started_total[5m])'
            ),
            'latency_p99_ms': await self.metrics.query(
                'histogram_quantile(0.99, rate(game_packet_latency_seconds_bucket[5m])) * 1000'
            ),
            'login_failure_rate': await self.metrics.query(
                'rate(login_failures_total[5m]) / rate(login_attempts_total[5m])'
            ),
            'purchase_failure_rate': await self.metrics.query(
                'rate(purchase_failures_total[5m]) / rate(purchase_attempts_total[5m])'
            )
        }
```

---

## 96.7 Seasonal Event Management

```python
# seasonal_events.py
# จัดการ seasonal events ใน game

from datetime import datetime, timezone
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class SeasonalEvent:
    event_id: str
    name: str
    start_time: datetime
    end_time: datetime
    content_version: str  # version ของ game content
    feature_flags: List[str]  # flags ที่ต้อง enable
    auto_disable_after: bool = True

class SeasonalEventManager:
    """จัดการ lifecycle ของ seasonal events"""
    
    def __init__(self, feature_flag_service, content_service, notification_service):
        self.flags = feature_flag_service
        self.content = content_service
        self.notify = notification_service
    
    async def activate_event(self, event: SeasonalEvent):
        """เปิดใช้งาน seasonal event"""
        
        print(f"🎉 Activating event: {event.name}")
        
        # 1. Deploy event content (assets, game data)
        await self.content.deploy_version(event.content_version)
        
        # 2. Enable feature flags
        for flag in event.feature_flags:
            await self.flags.enable(flag, rollout_percentage=100)
        
        # 3. Update game config
        await self.update_game_config({
            'active_event': event.event_id,
            'event_end_time': event.end_time.isoformat(),
            'event_theme': event.name
        })
        
        # 4. Notify players
        await self.notify.push_notification(
            title=f"🎮 {event.name} has started!",
            body="Log in now to claim your rewards"
        )
        
        # 5. Schedule auto-disable
        if event.auto_disable_after:
            await self.schedule_event_end(event)
        
        print(f"✅ Event {event.name} is now LIVE!")
    
    async def deactivate_event(self, event: SeasonalEvent):
        """ปิด seasonal event"""
        
        print(f"🔚 Deactivating event: {event.name}")
        
        # 1. Disable feature flags
        for flag in event.feature_flags:
            await self.flags.disable(flag)
        
        # 2. Handle players still in event content
        active_in_event = await self.get_players_in_event_content(event.event_id)
        if active_in_event:
            print(f"   Safely moving {len(active_in_event)} players from event content...")
            await self.safely_exit_players(active_in_event)
        
        # 3. Archive event data
        await self.archive_event_data(event.event_id)
        
        print(f"✅ Event {event.name} ended successfully")
    
    async def schedule_event_end(self, event: SeasonalEvent):
        """Schedule event end เมื่อถึงเวลา"""
        # ใช้ scheduler เช่น Celery, APScheduler
        import asyncio
        
        time_until_end = (event.end_time - datetime.now(timezone.utc)).total_seconds()
        
        if time_until_end > 0:
            print(f"   ⏰ Event will end automatically in {time_until_end/3600:.1f} hours")
            await asyncio.sleep(time_until_end)
            await self.deactivate_event(event)

# ตัวอย่าง Seasonal Events
events = [
    SeasonalEvent(
        event_id='songkran_2024',
        name='Songkran Festival 2024',
        start_time=datetime(2024, 4, 13, 0, 0, tzinfo=timezone.utc),
        end_time=datetime(2024, 4, 15, 23, 59, tzinfo=timezone.utc),
        content_version='songkran-assets-v1.2',
        feature_flags=[
            'songkran-map-decorations',
            'songkran-water-gun-weapon',
            'songkran-limited-skin',
            'songkran-bonus-xp'
        ]
    )
]
```

---

## 96.8 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Game Server Rolling Update

**โจทย์:** Implement simple game server rolling update

```python
# TODO: Implement GameServerRollingUpdate

class MockGameServer:
    def __init__(self, server_id: str):
        self.server_id = server_id
        self.active_matches = 0
        self.state = ServerState.ACTIVE
        self.version = "1.0.0"
    
    def simulate_match_end(self):
        """Simulate match ending"""
        if self.active_matches > 0:
            self.active_matches -= 1

class GameServerRollingUpdate:
    def __init__(self, servers: list):
        self.servers = servers
    
    def update(self, new_version: str, batch_size: int = 2):
        """
        TODO: Implement rolling update:
        1. แบ่ง servers เป็น batches
        2. Drain each batch
        3. รอให้ matches จบ (simulate ด้วย time.sleep)
        4. Update server version
        5. ตรวจสอบ health
        6. ดำเนิน batch ถัดไป
        """
        pass
    
    def drain_server(self, server: MockGameServer):
        """TODO: Set server to not accept new matches"""
        pass
    
    def wait_for_drain(self, server: MockGameServer, timeout: int = 10) -> bool:
        """TODO: Wait for all matches to complete"""
        pass

# Test
servers = [MockGameServer(f"server-{i}") for i in range(10)]
updater = GameServerRollingUpdate(servers)
updater.update("2.0.0")
```

### แบบฝึกหัดที่ 2: Feature Flag สำหรับ Weapon Balance

**โจทย์:** Implement feature flag system สำหรับ weapon balance testing

```typescript
// TODO: Implement weapon balance A/B test

interface WeaponStats {
  damage: number;
  fireRate: number;
  accuracy: number;
  reloadTime: number;
}

class WeaponBalanceService {
  constructor(private featureFlags: GameFeatureFlagService) {}
  
  async getWeaponStats(weaponId: string, player: PlayerContext): Promise<WeaponStats> {
    // TODO: 
    // 1. ถ้า player อยู่ใน ranked season → ใช้ stats ปัจจุบัน (ไม่ทดสอบกับ ranked)
    // 2. ถ้า feature flag 'new-weapon-balance' enabled → ใช้ balanced stats
    // 3. อื่นๆ → ใช้ current stats
    
    const currentStats: WeaponStats = {
      damage: 45,
      fireRate: 600,
      accuracy: 0.85,
      reloadTime: 2.5
    };
    
    // TODO: implement logic
    return currentStats;
  }
}
```

### แบบฝึกหัดที่ 3: Seasonal Event Pipeline

**โจทย์:** สร้าง GitHub Actions workflow สำหรับ seasonal event deployment

```yaml
# .github/workflows/seasonal-event.yml
# TODO: สร้าง pipeline สำหรับ seasonal event

name: Seasonal Event Deployment

on:
  workflow_dispatch:
    inputs:
      event_id:
        description: 'Event ID (e.g., songkran_2024)'
        required: true
      action:
        description: 'activate or deactivate'
        required: true
        type: choice
        options:
          - activate
          - deactivate

jobs:
  # TODO: สร้าง jobs สำหรับ:
  # 1. Validate event config
  # 2. Deploy event assets to CDN (ถ้า activate)
  # 3. Enable/disable feature flags
  # 4. Notify players (ถ้า activate)
  # 5. Verify event is working (smoke test)
  
  validate-event:
    runs-on: ubuntu-latest
    steps:
      - name: Validate event configuration
        run: |
          # TODO: implement validation
          echo "Validating event: ${{ github.event.inputs.event_id }}"
```

---

## สรุป

Gaming Platform CI/CD มีความพิเศษที่:

1. **Game-Aware Deployment** — ต้องรู้ว่ามี active matches หรือไม่
2. **Emergency Patching** — Fast-track สำหรับ exploits และ cheats
3. **Feature Flags** — Safe rollout ของ game features และ balance changes
4. **Player Impact** — Monitor player experience ตลอดเวลา
5. **Seasonal Events** — Automated activation/deactivation
6. **Anti-Cheat** — Silent updates เพื่อป้องกัน counter-measures
7. **Client Distribution** — Delta patches สำหรับลด download size

---

**ต่อไป:** Part 97 - Open Source CI/CD Tools Landscape
