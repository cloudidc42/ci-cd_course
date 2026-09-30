# Part 64: Mobile App CI/CD

## บทนำ: ความท้าทายของ Mobile CI/CD

Mobile app development มีความซับซ้อนเฉพาะตัวที่ทำให้ CI/CD แตกต่างจาก web applications:

1. **Platform-specific builds**: iOS ต้องการ macOS, Android ต้องการ JDK
2. **Code Signing**: certificates, provisioning profiles, keystores ที่ซับซ้อน
3. **App Store Reviews**: ต้องรอ review ก่อน release
4. **Device Fragmentation**: ทดสอบบน devices หลายรุ่น
5. **Binary Size**: ต้องควบคุมขนาด binary ให้ไม่ใหญ่เกินไป

---

## 1. iOS CI/CD ด้วย Fastlane

### 1.1 Fastlane คืออะไร?

Fastlane เป็น open-source platform ที่ automate การ build, test, และ release mobile apps ทั้ง iOS และ Android

### 1.2 Setup Fastlane สำหรับ iOS

```bash
# ติดตั้ง Fastlane
gem install fastlane

# หรือด้วย Bundler (แนะนำ)
# สร้าง Gemfile
cat > Gemfile << 'EOF'
source "https://rubygems.org"

gem "fastlane"
gem "cocoapods"

plugins_path = File.join(File.dirname(__FILE__), 'fastlane', 'Pluginfile')
eval_gemfile(plugins_path) if File.exist?(plugins_path)
EOF

bundle install

# Initialize Fastlane
bundle exec fastlane init
```

### 1.3 Appfile

```ruby
# fastlane/Appfile
app_identifier "com.company.myapp"
apple_id "developer@company.com"
team_id "ABCD1234EF"

# App Store Connect App ID
itc_team_id "123456789"

# For multiple environments
for_platform :ios do
  for_lane :beta do
    app_identifier "com.company.myapp.beta"
  end
  
  for_lane :release do
    app_identifier "com.company.myapp"
  end
end
```

### 1.4 Fastfile สำหรับ iOS

```ruby
# fastlane/Fastfile
require 'json'

default_platform(:ios)

# ==================== Constants ====================
APP_NAME = "MyApp"
WORKSPACE = "MyApp.xcworkspace"
SCHEME = "MyApp"
BUILD_NUMBER_KEY = "APP_BUILD_NUMBER"

# ==================== Before All ====================
before_all do |lane|
  # ตรวจสอบ Xcode version
  ensure_xcode_version(version: "15.0")
  
  # Setup environment
  setup_ci if ENV['CI']
end

# ==================== Tests ====================
lane :test do |options|
  desc "Run all tests"
  
  # Run unit tests
  run_tests(
    workspace: WORKSPACE,
    scheme: SCHEME,
    devices: ["iPhone 15", "iPad Pro (12.9-inch)"],
    code_coverage: true,
    output_directory: "fastlane/test_output",
    output_files: "junit-report.xml",
    fail_build: true,
    xcargs: "-maximum-parallel-testing-workers 4"
  )
  
  # Generate coverage report
  slather(
    cobertura_xml: true,
    proj: "MyApp.xcodeproj",
    scheme: SCHEME,
    output_directory: "fastlane/coverage"
  )
end

# ==================== Beta (TestFlight) ====================
lane :beta do |options|
  desc "Build and upload to TestFlight"
  
  # Increment build number
  build_number = increment_build_number_in_xcconfig(
    xcconfig_paths: ["MyApp/Configurations/Shared.xcconfig"],
    bump_type: "patch"
  )
  
  # Setup certificates via Match
  match(
    type: "appstore",
    readonly: is_ci,
    app_identifier: "com.company.myapp"
  )
  
  # Build
  gym(
    workspace: WORKSPACE,
    scheme: SCHEME,
    configuration: "Release",
    export_method: "app-store",
    output_directory: "build/",
    output_name: "MyApp-#{build_number}.ipa",
    xcargs: {
      "CODE_SIGN_STYLE" => "Manual",
      "DEVELOPMENT_TEAM" => ENV["TEAM_ID"]
    }
  )
  
  # Upload to TestFlight
  pilot(
    app_identifier: "com.company.myapp",
    skip_waiting_for_build_processing: false,
    distribute_external: options[:distribute] || false,
    groups: options[:groups] || ["Internal Testers"],
    changelog: generate_changelog,
    notify_external_testers: options[:notify] || false
  )
  
  # Notify team
  slack(
    message: "New iOS beta #{build_number} uploaded to TestFlight!",
    channel: "#mobile-releases",
    payload: {
      "Build Number" => build_number,
      "Branch" => git_branch,
      "Commit" => last_git_commit[:abbreviated_commit_hash]
    },
    default_payloads: []
  )
end

# ==================== Production Release ====================
lane :release do |options|
  desc "Build and submit to App Store"
  
  # Verify we're on main branch
  ensure_git_branch(branch: "main")
  ensure_git_status_clean
  
  # Get version from tag
  version = options[:version] || prompt(text: "Version number: ")
  
  # Update version
  increment_version_number(
    version_number: version,
    xcodeproj: "MyApp.xcodeproj"
  )
  
  # Increment build number
  increment_build_number(xcodeproj: "MyApp.xcodeproj")
  
  # Run tests
  test
  
  # Setup certs
  match(type: "appstore", readonly: is_ci)
  
  # Build
  gym(
    workspace: WORKSPACE,
    scheme: SCHEME,
    configuration: "Release",
    export_method: "app-store"
  )
  
  # Upload to App Store
  deliver(
    app_version: version,
    submit_for_review: true,
    automatic_release: false,
    force: true,
    metadata_path: "./fastlane/metadata",
    screenshots_path: "./fastlane/screenshots",
    release_notes: {
      "en-US" => options[:release_notes] || generate_changelog,
      "th" => options[:release_notes_th] || "อัพเดทระบบและแก้ไขข้อผิดพลาด"
    }
  )
  
  # Tag release
  add_git_tag(
    tag: "ios-v#{version}",
    message: "iOS Release v#{version}"
  )
  push_git_tags
  
  # Create GitHub Release
  github_release = set_github_release(
    repository_name: "company/myapp",
    api_token: ENV["GITHUB_TOKEN"],
    name: "iOS v#{version}",
    tag_name: "ios-v#{version}",
    description: generate_changelog
  )
end

# ==================== Code Signing ====================
lane :setup_certs do |options|
  desc "Setup code signing certificates"
  
  match(
    type: options[:type] || "development",
    app_identifier: [
      "com.company.myapp",
      "com.company.myapp.beta"
    ],
    git_url: ENV["MATCH_GIT_URL"],
    git_basic_authorization: Base64.strict_encode64(
      "#{ENV['MATCH_GIT_USERNAME']}:#{ENV['MATCH_GIT_TOKEN']}"
    )
  )
end

# ==================== Screenshots ====================
lane :screenshots do
  desc "Generate App Store screenshots"
  
  capture_screenshots(
    workspace: WORKSPACE,
    scheme: "MyAppUITests",
    devices: [
      "iPhone 14 Pro Max",
      "iPhone SE (3rd generation)",
      "iPad Pro (12.9-inch)"
    ],
    languages: ["en-US", "th"],
    output_directory: "./fastlane/screenshots",
    clear_previous_screenshots: true
  )
  
  frame_screenshots(
    silver: true,
    path: "./fastlane/screenshots"
  )
end

# ==================== Helpers ====================
def generate_changelog
  # ดึง commits ตั้งแต่ tag ล่าสุด
  changelog_from_git_commits(
    between: [last_git_tag, "HEAD"],
    pretty: "- %s",
    date_format: "short",
    match_lightweight_tag: false
  )
rescue
  "Bug fixes and improvements"
end

# ==================== Error Handler ====================
error do |lane, exception, options|
  slack(
    message: "Lane #{lane} failed: #{exception.message}",
    channel: "#mobile-alerts",
    success: false,
    payload: {
      "Lane" => lane.to_s,
      "Error" => exception.message,
      "Branch" => git_branch
    }
  )
end
```

### 1.5 Match สำหรับ Code Signing

```ruby
# fastlane/Matchfile
git_url ENV["MATCH_GIT_URL"]
storage_mode "git"  # หรือ "s3", "google_cloud"

type "development"

app_identifier [
  "com.company.myapp",
  "com.company.myapp.beta",
  "com.company.myapp.dev"
]

# สำหรับ CI
readonly ENV['CI'] ? true : false

# Generate certificates ในรูปแบบใหม่
generate_apple_certs true
```

---

## 2. GitHub Actions สำหรับ iOS

### 2.1 Complete iOS CI/CD Pipeline

```yaml
# .github/workflows/ios.yml
name: iOS CI/CD Pipeline

on:
  push:
    branches: [main, develop]
    paths:
      - 'iOS/**'
      - '.github/workflows/ios.yml'
  pull_request:
    branches: [main, develop]
    paths: ['iOS/**']
  release:
    types: [published]

env:
  DEVELOPER_DIR: /Applications/Xcode_15.0.app/Contents/Developer
  FASTLANE_XCODEBUILD_SETTINGS_TIMEOUT: 120
  FASTLANE_SKIP_UPDATE_CHECK: "1"

jobs:
  # ==================== Test ====================
  test:
    name: Run iOS Tests
    runs-on: macos-13
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true
          working-directory: iOS/
      
      - name: Cache CocoaPods
        uses: actions/cache@v4
        with:
          path: iOS/Pods
          key: pods-${{ hashFiles('iOS/Podfile.lock') }}
          restore-keys: pods-
      
      - name: Install CocoaPods
        run: |
          cd iOS
          bundle exec pod install --repo-update
      
      - name: Run tests
        run: |
          cd iOS
          bundle exec fastlane test
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: ios-test-results
          path: iOS/fastlane/test_output/
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: iOS/fastlane/coverage/cobertura.xml
          flags: ios

  # ==================== Beta Build ====================
  beta:
    name: Build and Upload to TestFlight
    runs-on: macos-13
    needs: test
    if: github.ref == 'refs/heads/develop' || github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true
          working-directory: iOS/
      
      - name: Install CocoaPods
        run: |
          cd iOS
          bundle exec pod install
      
      - name: Import signing certificates
        env:
          CERTIFICATES_P12: ${{ secrets.IOS_CERTIFICATES_P12 }}
          CERTIFICATES_PASSWORD: ${{ secrets.IOS_CERTIFICATES_PASSWORD }}
          PROVISIONING_PROFILE: ${{ secrets.IOS_PROVISIONING_PROFILE }}
        run: |
          # Create keychain
          KEYCHAIN_PATH=$RUNNER_TEMP/app-signing.keychain-db
          KEYCHAIN_PASSWORD=$(openssl rand -base64 32)
          
          security create-keychain -p "$KEYCHAIN_PASSWORD" "$KEYCHAIN_PATH"
          security set-keychain-settings -lut 21600 "$KEYCHAIN_PATH"
          security unlock-keychain -p "$KEYCHAIN_PASSWORD" "$KEYCHAIN_PATH"
          
          # Import certificate
          echo -n "$CERTIFICATES_P12" | base64 --decode -o certificate.p12
          security import certificate.p12 -P "$CERTIFICATES_PASSWORD" \
            -A -t cert -f pkcs12 -k "$KEYCHAIN_PATH"
          security list-keychain -d user -s "$KEYCHAIN_PATH"
          
          # Import provisioning profile
          echo -n "$PROVISIONING_PROFILE" | base64 --decode -o profile.mobileprovision
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          cp profile.mobileprovision \
            ~/Library/MobileDevice/Provisioning\ Profiles/
      
      - name: Build and upload beta
        env:
          APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.ASC_KEY_ID }}
          APP_STORE_CONNECT_API_ISSUER_ID: ${{ secrets.ASC_ISSUER_ID }}
          APP_STORE_CONNECT_API_KEY_CONTENT: ${{ secrets.ASC_PRIVATE_KEY }}
        run: |
          cd iOS
          bundle exec fastlane beta \
            distribute:true \
            groups:"Internal Testers,QA Team"
      
      - name: Cleanup keychain
        if: always()
        run: |
          security delete-keychain $RUNNER_TEMP/app-signing.keychain-db || true

  # ==================== Production Release ====================
  release:
    name: Submit to App Store
    runs-on: macos-13
    needs: beta
    if: github.event_name == 'release'
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true
          working-directory: iOS/
      
      - name: Install CocoaPods
        run: |
          cd iOS
          bundle exec pod install
      
      - name: Setup signing
        env:
          MATCH_GIT_URL: ${{ secrets.MATCH_GIT_URL }}
          MATCH_GIT_TOKEN: ${{ secrets.MATCH_GIT_TOKEN }}
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
        run: |
          cd iOS
          bundle exec fastlane setup_certs type:appstore
      
      - name: Release to App Store
        env:
          APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.ASC_KEY_ID }}
          APP_STORE_CONNECT_API_ISSUER_ID: ${{ secrets.ASC_ISSUER_ID }}
          APP_STORE_CONNECT_API_KEY_CONTENT: ${{ secrets.ASC_PRIVATE_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          cd iOS
          VERSION=$(echo "${{ github.ref }}" | sed 's/refs\/tags\/v//')
          bundle exec fastlane release version:"$VERSION"
```

---

## 3. Xcode Cloud

### 3.1 Xcode Cloud Workflow

```yaml
# .xcode/workflows/ci.yml (Xcode Cloud CI definition)
# สร้างผ่าน Xcode IDE หรือ App Store Connect

name: iOS CI

# Start conditions
on:
  - pullRequestTargeting: main
  - branchChanges:
    - main
    - develop
  - tagChanges:
    - 'v*'

environment:
  macOS: 13
  xcode: 15.0
  
actions:
  build:
    scheme: MyApp
    buildForTesting: true
  
  test:
    scheme: MyApp
    testPlan: MyAppTests
    
  analyzeAction:
    scheme: MyApp
    
  archiveAction:
    scheme: MyApp
    configuration: Release
```

---

## 4. Android CI/CD ด้วย Gradle และ Fastlane

### 4.1 Android Fastfile

```ruby
# fastlane/Fastfile (Android)
default_platform(:android)

PACKAGE_NAME = "com.company.myapp"

platform :android do
  
  # ==================== Tests ====================
  lane :test do
    desc "Run Android tests"
    
    gradle(
      task: "test",
      build_type: "Debug"
    )
    
    # Run instrumented tests (ต้องมี emulator)
    gradle(
      task: "connectedAndroidTest",
      build_type: "Debug"
    ) if ENV['RUN_INSTRUMENTED_TESTS']
  end
  
  # ==================== Build ====================
  lane :build do |options|
    desc "Build Android app"
    
    build_type = options[:build_type] || "Release"
    flavor = options[:flavor] || ""
    
    gradle(
      task: "assemble",
      flavor: flavor,
      build_type: build_type,
      properties: {
        "android.injected.signing.store.file" => ENV["KEYSTORE_PATH"],
        "android.injected.signing.store.password" => ENV["KEYSTORE_PASSWORD"],
        "android.injected.signing.key.alias" => ENV["KEY_ALIAS"],
        "android.injected.signing.key.password" => ENV["KEY_PASSWORD"]
      }
    )
  end
  
  # ==================== Firebase App Distribution ====================
  lane :firebase_distribute do |options|
    desc "Distribute to Firebase App Distribution"
    
    # Build APK
    build(build_type: "Debug", flavor: options[:flavor])
    
    # Distribute
    firebase_app_distribution(
      app: ENV["FIREBASE_APP_ID"],
      firebase_cli_token: ENV["FIREBASE_CLI_TOKEN"],
      groups: options[:groups] || "internal-testers",
      release_notes: generate_release_notes,
      apk_path: lane_context[SharedValues::GRADLE_APK_OUTPUT_PATH]
    )
    
    slack(
      message: "Android build distributed via Firebase!",
      channel: "#mobile-releases"
    )
  end
  
  # ==================== Google Play ====================
  lane :beta do
    desc "Upload to Play Store internal testing"
    
    # Increment version code
    increment_version_code(
      gradle_file_path: "app/build.gradle"
    )
    
    # Build release bundle
    gradle(
      task: "bundle",
      build_type: "Release",
      properties: {
        "android.injected.signing.store.file" => ENV["KEYSTORE_PATH"],
        "android.injected.signing.store.password" => ENV["KEYSTORE_PASSWORD"],
        "android.injected.signing.key.alias" => ENV["KEY_ALIAS"],
        "android.injected.signing.key.password" => ENV["KEY_PASSWORD"]
      }
    )
    
    # Upload to Play Store
    upload_to_play_store(
      package_name: PACKAGE_NAME,
      track: "internal",
      aab: lane_context[SharedValues::GRADLE_AAB_OUTPUT_PATH],
      json_key: ENV["GOOGLE_PLAY_JSON_KEY_PATH"],
      skip_upload_metadata: true,
      skip_upload_images: true,
      skip_upload_screenshots: true
    )
  end
  
  lane :production do |options|
    desc "Deploy to Play Store production"
    
    version_name = options[:version] || prompt(text: "Version name: ")
    
    # Update version
    android_set_version_name(
      version_name: version_name,
      gradle_file_path: "app/build.gradle"
    )
    
    increment_version_code(
      gradle_file_path: "app/build.gradle"
    )
    
    # Build
    gradle(
      task: "bundle",
      build_type: "Release"
    )
    
    # Upload with full metadata
    upload_to_play_store(
      package_name: PACKAGE_NAME,
      track: "production",
      rollout: "0.1",  # Start with 10% rollout
      aab: lane_context[SharedValues::GRADLE_AAB_OUTPUT_PATH],
      json_key: ENV["GOOGLE_PLAY_JSON_KEY_PATH"],
      mapping: "app/build/outputs/mapping/release/mapping.txt",
      release_status: "inProgress"  # Staged rollout
    )
  end

  def generate_release_notes
    changelog_from_git_commits(
      between: [last_git_tag, "HEAD"],
      pretty: "- %s"
    )
  rescue
    "Bug fixes and performance improvements"
  end
end
```

### 4.2 build.gradle สำหรับ CI

```groovy
// app/build.gradle
plugins {
    id 'com.android.application'
    id 'kotlin-android'
    id 'com.google.firebase.crashlytics'
    id 'com.google.gms.google-services'
}

android {
    compileSdk 34
    
    defaultConfig {
        applicationId "com.company.myapp"
        minSdk 24
        targetSdk 34
        versionCode getVersionCode()
        versionName getVersionName()
        
        // สำหรับ testing
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }
    
    // Signing configurations
    signingConfigs {
        debug {
            storeFile file(System.getenv("DEBUG_KEYSTORE_PATH") ?: "debug.keystore")
            storePassword System.getenv("DEBUG_KEYSTORE_PASSWORD") ?: "android"
            keyAlias System.getenv("DEBUG_KEY_ALIAS") ?: "androiddebugkey"
            keyPassword System.getenv("DEBUG_KEY_PASSWORD") ?: "android"
        }
        
        release {
            if (System.getenv("KEYSTORE_PATH")) {
                storeFile file(System.getenv("KEYSTORE_PATH"))
                storePassword System.getenv("KEYSTORE_PASSWORD")
                keyAlias System.getenv("KEY_ALIAS")
                keyPassword System.getenv("KEY_PASSWORD")
            } else {
                // Fallback to debug signing for local development
                storeFile file("debug.keystore")
                storePassword "android"
                keyAlias "androiddebugkey"
                keyPassword "android"
            }
        }
    }
    
    buildTypes {
        debug {
            applicationIdSuffix ".debug"
            versionNameSuffix "-debug"
            signingConfig signingConfigs.debug
            minifyEnabled false
        }
        
        release {
            signingConfig signingConfigs.release
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 
                          'proguard-rules.pro'
            
            // Enable R8 full mode
            crunchPngs true
        }
    }
    
    // Product flavors สำหรับ environments
    flavorDimensions "environment"
    productFlavors {
        dev {
            dimension "environment"
            applicationIdSuffix ".dev"
            buildConfigField "String", "API_URL", '"https://dev-api.company.com"'
            buildConfigField "boolean", "ENABLE_LOGGING", "true"
        }
        
        staging {
            dimension "environment"
            applicationIdSuffix ".staging"
            buildConfigField "String", "API_URL", '"https://staging-api.company.com"'
            buildConfigField "boolean", "ENABLE_LOGGING", "true"
        }
        
        prod {
            dimension "environment"
            buildConfigField "String", "API_URL", '"https://api.company.com"'
            buildConfigField "boolean", "ENABLE_LOGGING", "false"
        }
    }
    
    // Test options
    testOptions {
        unitTests {
            includeAndroidResources = true
            returnDefaultValues = true
        }
        animationsDisabled = true
    }
    
    // Enable View Binding
    buildFeatures {
        viewBinding true
        buildConfig true
    }
    
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_17
        targetCompatibility JavaVersion.VERSION_17
    }
    
    kotlinOptions {
        jvmTarget = '17'
    }
}

// Helper functions สำหรับ version
def getVersionCode() {
    def versionCode = System.getenv("VERSION_CODE")
    return versionCode ? versionCode.toInteger() : 1
}

def getVersionName() {
    return System.getenv("VERSION_NAME") ?: "1.0.0"
}

dependencies {
    // Core
    implementation 'androidx.core:core-ktx:1.12.0'
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.google.android.material:material:1.11.0'
    
    // Testing
    testImplementation 'junit:junit:4.13.2'
    testImplementation 'org.mockito:mockito-kotlin:5.2.1'
    testImplementation 'app.cash.turbine:turbine:1.0.0'
    
    androidTestImplementation 'androidx.test.ext:junit:1.1.5'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.5.1'
    androidTestImplementation 'androidx.compose.ui:ui-test-junit4:1.5.4'
}
```

### 4.3 GitHub Actions สำหรับ Android

```yaml
# .github/workflows/android.yml
name: Android CI/CD Pipeline

on:
  push:
    branches: [main, develop]
    paths: ['Android/**', '.github/workflows/android.yml']
  pull_request:
    branches: [main, develop]
    paths: ['Android/**']

jobs:
  test:
    name: Run Android Tests
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: gradle
      
      - name: Grant execute permission for gradlew
        run: chmod +x Android/gradlew
      
      - name: Cache Gradle
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
      
      - name: Run unit tests
        run: |
          cd Android
          ./gradlew test --no-daemon
      
      - name: Upload test reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: android-test-reports
          path: Android/app/build/reports/tests/
      
      - name: Code coverage
        run: |
          cd Android
          ./gradlew jacocoTestReport --no-daemon
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: Android/app/build/reports/jacoco/jacocoTestReport/jacocoTestReport.xml
          flags: android

  lint:
    name: Android Lint
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Run lint
        run: |
          cd Android
          ./gradlew lintRelease --no-daemon
      
      - name: Upload lint results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: android-lint-results
          path: Android/app/build/reports/lint-results-release.html
      
      - name: Annotate PR with lint warnings
        if: github.event_name == 'pull_request'
        uses: yutailang0119/action-android-lint@v3
        with:
          xml_path: Android/app/build/reports/lint-results-release.xml

  firebase-distribute:
    name: Distribute to Firebase
    runs-on: ubuntu-latest
    needs: [test, lint]
    if: github.ref == 'refs/heads/develop'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true
          working-directory: Android/
      
      - name: Decode Keystore
        env:
          KEYSTORE_BASE64: ${{ secrets.ANDROID_KEYSTORE_BASE64 }}
        run: |
          echo "$KEYSTORE_BASE64" | base64 --decode > $RUNNER_TEMP/release.keystore
      
      - name: Build and distribute
        env:
          KEYSTORE_PATH: ${{ runner.temp }}/release.keystore
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
          FIREBASE_APP_ID: ${{ secrets.FIREBASE_ANDROID_APP_ID }}
          FIREBASE_CLI_TOKEN: ${{ secrets.FIREBASE_CLI_TOKEN }}
        run: |
          cd Android
          bundle exec fastlane firebase_distribute groups:"qa-team"

  play-store-beta:
    name: Upload to Play Store Beta
    runs-on: ubuntu-latest
    needs: [test, lint]
    if: github.ref == 'refs/heads/main'
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Setup JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true
          working-directory: Android/
      
      - name: Decode Keystore
        env:
          KEYSTORE_BASE64: ${{ secrets.ANDROID_KEYSTORE_BASE64 }}
        run: |
          echo "$KEYSTORE_BASE64" | base64 --decode > $RUNNER_TEMP/release.keystore
      
      - name: Decode Google Play JSON Key
        env:
          PLAY_JSON_KEY: ${{ secrets.GOOGLE_PLAY_JSON_KEY }}
        run: |
          echo "$PLAY_JSON_KEY" > $RUNNER_TEMP/play-key.json
      
      - name: Upload to Play Store Internal
        env:
          KEYSTORE_PATH: ${{ runner.temp }}/release.keystore
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
          GOOGLE_PLAY_JSON_KEY_PATH: ${{ runner.temp }}/play-key.json
        run: |
          cd Android
          
          # Set version code from build number
          echo "VERSION_CODE=${{ github.run_number }}" >> $GITHUB_ENV
          echo "VERSION_NAME=$(cat version.txt)" >> $GITHUB_ENV
          
          bundle exec fastlane beta
```

---

## 5. Instrumented Testing บน Real Devices

### 5.1 Firebase Test Lab

```yaml
# .github/workflows/firebase-test.yml
name: Firebase Test Lab

on:
  pull_request:
    branches: [main]

jobs:
  firebase-test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Build test APK
        run: |
          cd Android
          ./gradlew assembleDebug assembleDebugAndroidTest --no-daemon
      
      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SERVICE_ACCOUNT_KEY }}
      
      - name: Setup gcloud
        uses: google-github-actions/setup-gcloud@v2
      
      - name: Run tests on Firebase Test Lab
        run: |
          gcloud firebase test android run \
            --type instrumentation \
            --app Android/app/build/outputs/apk/debug/app-debug.apk \
            --test Android/app/build/outputs/apk/androidTest/debug/app-debug-androidTest.apk \
            --device model=Pixel7,version=33,locale=th,orientation=portrait \
            --device model=Pixel7,version=33,locale=en,orientation=portrait \
            --device model=Pixel4a,version=30,locale=th,orientation=portrait \
            --timeout 5m \
            --results-bucket gs://my-test-results \
            --results-dir firebase-tests/${{ github.run_id }}
      
      - name: Download test results
        if: always()
        run: |
          gsutil -m cp -r \
            gs://my-test-results/firebase-tests/${{ github.run_id }}/* \
            ./test-results/
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: firebase-test-results
          path: test-results/
```

---

## 6. Code Signing Strategy

### 6.1 iOS Certificates Management ด้วย Match

```bash
# สร้าง private repository สำหรับ certificates
# (แนะนำให้เป็น private repo และ encrypt ด้วย password)

# Initialize match
bundle exec fastlane match init

# Generate/sync certificates
bundle exec fastlane match development  # สำหรับ development
bundle exec fastlane match adhoc        # สำหรับ beta distribution
bundle exec fastlane match appstore     # สำหรับ App Store

# Nuke and recreate (เมื่อ certificates expire)
bundle exec fastlane match nuke development
bundle exec fastlane match development --force

# สำหรับ CI: readonly mode เท่านั้น
MATCH_READONLY=true bundle exec fastlane match appstore
```

### 6.2 Android Keystore Management

```bash
# สร้าง keystore ใหม่
keytool -genkey -v \
  -keystore release.keystore \
  -alias myapp \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -dname "CN=My Company, OU=Mobile, O=Company, L=Bangkok, S=Bangkok, C=TH"

# Encode เป็น Base64 สำหรับ GitHub Secrets
base64 -i release.keystore | tr -d '\n'

# เก็บใน GitHub Secrets:
# ANDROID_KEYSTORE_BASE64 = <base64 output>
# KEYSTORE_PASSWORD = <your password>
# KEY_ALIAS = myapp
# KEY_PASSWORD = <your key password>
```

---

## 7. TestFlight และ App Store Submission

### 7.1 App Store Connect API

```ruby
# fastlane/actions/app_store_connect.rb
# ใช้ API Key แทน Apple ID password

# สร้าง API Key ที่ App Store Connect
# (Users and Access > Keys > App Store Connect API)

# Appfile
api_key_path "fastlane/AuthKey_ABCD1234.p8"
# หรือ pass ผ่าน environment variable:
# APP_STORE_CONNECT_API_KEY_PATH, APP_STORE_CONNECT_API_KEY_ID,
# APP_STORE_CONNECT_API_ISSUER_ID

# ใช้ใน Fastfile
lane :upload do
  api_key = app_store_connect_api_key(
    key_id: ENV["ASC_KEY_ID"],
    issuer_id: ENV["ASC_ISSUER_ID"],
    key_content: ENV["ASC_PRIVATE_KEY"],
    is_key_content_base64: true
  )
  
  pilot(
    api_key: api_key,
    skip_waiting_for_build_processing: false
  )
end
```

### 7.2 Automated Screenshot Generation

```swift
// UITests/Screenshots/ScreenshotTests.swift
import XCTest

class ScreenshotTests: XCTestCase {
    
    override func setUpWithError() throws {
        continueAfterFailure = false
        
        let app = XCUIApplication()
        setupSnapshot(app)
        app.launch()
    }
    
    func testTakeScreenshots() throws {
        let app = XCUIApplication()
        
        // Home Screen
        snapshot("01-HomeScreen")
        
        // Navigate to product list
        app.tabBars.buttons["Products"].tap()
        snapshot("02-ProductList")
        
        // Open product detail
        app.collectionViews.cells.firstMatch.tap()
        snapshot("03-ProductDetail")
        
        // Cart
        app.buttons["Add to Cart"].tap()
        app.tabBars.buttons["Cart"].tap()
        snapshot("04-Cart")
        
        // Checkout
        app.buttons["Checkout"].tap()
        snapshot("05-Checkout")
    }
}
```

---

## 8. แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: iOS CI/CD Pipeline

```
Task: สร้าง complete iOS CI/CD pipeline

Requirements:
1. Setup Fastlane ด้วย:
   - test lane (unit + UI tests)
   - beta lane (TestFlight)
   - release lane (App Store)

2. GitHub Actions workflows:
   - PR: run tests only
   - Develop branch: build + Firebase distribute
   - Main branch: TestFlight upload
   - Release tag: App Store submission

3. Code signing ด้วย Match:
   - Development certs
   - Distribution certs
   - Store ใน private repo

4. Automated changelog generation
5. Slack notifications

Challenge:
- ทำให้ build number increment อัตโนมัติ
- Generate screenshots สำหรับ App Store
```

### แบบฝึกหัดที่ 2: Android Pipeline with Firebase

```
Task: สร้าง Android CI/CD ด้วย Firebase App Distribution

Steps:
1. Setup Gradle signing configurations
2. สร้าง Fastlane Android Fastfile
3. Configure Firebase App Distribution
4. Setup GitHub Actions:
   - Unit tests + Lint
   - Build APK (debug) 
   - Firebase distribution ไปยัง QA team
   - Play Store upload (internal track)

Testing:
- Run Firebase Test Lab บน 3 device configurations
- Generate test report
- Fail PR ถ้า test fail
```

### แบบฝึกหัดที่ 3: Cross-Platform React Native

```
สร้าง CI/CD pipeline สำหรับ React Native app:

1. Shared pipeline:
   - Jest unit tests
   - TypeScript type checking
   - ESLint

2. iOS-specific:
   - Pod install
   - Build + TestFlight

3. Android-specific:
   - Gradle build
   - Firebase distribution

Challenge:
- Cache node_modules และ Pods
- Build ทั้ง iOS และ Android parallel
- ส่ง notification เมื่อ both platforms build สำเร็จ
```

### สรุปบทที่ 64

ในบทนี้เราได้เรียนรู้:
- **Fastlane**: Automate iOS และ Android builds, signing, distribution
- **iOS CI/CD**: GitHub Actions + macOS runners สำหรับ iOS builds
- **Xcode Cloud**: Apple's native CI/CD solution
- **Android Gradle**: Build variants, product flavors, signing
- **Firebase Test Lab**: Real device testing ด้วย cloud
- **Code Signing**: Match สำหรับ iOS, Keystore สำหรับ Android
- **TestFlight/Play Store**: Automated submission pipelines

บทถัดไปเราจะเรียนรู้ Frontend CI/CD ด้วย Next.js, Vercel, และ Netlify
