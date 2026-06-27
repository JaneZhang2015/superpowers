# WeChat Travel Check-in Mini Program Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-city or single-scenic-area WeChat mini program where users can view preset attractions, unlock check-ins only when physically nearby, save travel records, and track collection progress.

**Architecture:** Use a native WeChat mini program client plus WeChat Cloud functions. Keep page rendering thin by moving distance, status, progress, and check-in rules into small service modules that can be unit-tested with Jest. Persist authoritative check-ins in cloud storage while caching progress locally for responsive page updates.

**Tech Stack:** Native WeChat Mini Program (JavaScript, WXML, WXSS, JSON), Jest, WeChat Cloud Functions, WeChat Cloud Database

**Spec:** `docs/superpowers/specs/2026-04-14-wechat-travel-checkin-mini-program-design.md`

---

## File Structure

| File | Responsibility | Action |
|---|---|---|
| `travel-checkin-mini-program/package.json` | Local scripts and Jest dependencies | Create |
| `travel-checkin-mini-program/project.config.json` | WeChat DevTools project config | Create |
| `travel-checkin-mini-program/app.js` | Cloud initialization | Create |
| `travel-checkin-mini-program/app.json` | Global pages + tab bar | Create |
| `travel-checkin-mini-program/app.wxss` | Shared styles | Create |
| `travel-checkin-mini-program/sitemap.json` | WeChat sitemap config | Create |
| `travel-checkin-mini-program/shared/spots.js` | Shared preset scenic spots for client + cloud | Create |
| `travel-checkin-mini-program/shared/geo.js` | Shared distance helper for client + cloud | Create |
| `travel-checkin-mini-program/miniprogram/data/spots.js` | Preset scenic spots for the MVP scope | Create |
| `travel-checkin-mini-program/miniprogram/utils/geo.js` | Distance and status helpers | Create |
| `travel-checkin-mini-program/miniprogram/services/spot-service.js` | Catalog/home data shaping | Create |
| `travel-checkin-mini-program/miniprogram/services/checkin-service.js` | Check-in eligibility, local cache, cloud submission | Create |
| `travel-checkin-mini-program/miniprogram/services/profile-service.js` | Progress and badge view models | Create |
| `travel-checkin-mini-program/miniprogram/pages/home/index.{js,wxml,wxss,json}` | Home page | Create |
| `travel-checkin-mini-program/miniprogram/pages/catalog/index.{js,wxml,wxss,json}` | Spot catalog page | Create |
| `travel-checkin-mini-program/miniprogram/pages/spot/index.{js,wxml,wxss,json}` | Spot detail + check-in page | Create |
| `travel-checkin-mini-program/miniprogram/pages/profile/index.{js,wxml,wxss,json}` | Profile/progress page | Create |
| `travel-checkin-mini-program/cloudfunctions/checkin/package.json` | Cloud function dependencies | Create |
| `travel-checkin-mini-program/cloudfunctions/checkin/index.js` | Authoritative check-in API | Create |
| `travel-checkin-mini-program/tests/geo.test.js` | Distance/status tests | Create |
| `travel-checkin-mini-program/tests/spot-service.test.js` | Home/catalog service tests | Create |
| `travel-checkin-mini-program/tests/checkin-service.test.js` | Check-in service tests | Create |
| `travel-checkin-mini-program/tests/profile-service.test.js` | Profile/progress tests | Create |
| `travel-checkin-mini-program/tests/cloud-checkin.test.js` | Cloud function rule tests | Create |

---

### Task 1: Bootstrap the mini program workspace

**Files:**
- Create: `travel-checkin-mini-program/package.json`
- Create: `travel-checkin-mini-program/project.config.json`
- Create: `travel-checkin-mini-program/app.js`
- Create: `travel-checkin-mini-program/app.json`
- Create: `travel-checkin-mini-program/app.wxss`
- Create: `travel-checkin-mini-program/sitemap.json`

- [ ] **Step 1: Create `package.json` and project scripts**

```json
{
  "name": "travel-checkin-mini-program",
  "private": true,
  "scripts": {
    "test": "jest --runInBand"
  },
  "devDependencies": {
    "jest": "^29.7.0",
    "wx-server-sdk": "^3.0.1"
  }
}
```

- [ ] **Step 2: Create WeChat project config files**

`travel-checkin-mini-program/project.config.json`

```json
{
  "appid": "wx1234567890abcd",
  "projectname": "travel-checkin-mini-program",
  "miniprogramRoot": "miniprogram/",
  "cloudfunctionRoot": "cloudfunctions/",
  "compileType": "miniprogram",
  "setting": {
    "es6": true,
    "enhance": true,
    "postcss": true,
    "minified": true,
    "urlCheck": false
  },
  "simulatorType": "wechat",
  "libVersion": "trial"
}
```

`travel-checkin-mini-program/sitemap.json`

```json
{
  "desc": "travel checkin sitemap",
  "rules": [
    {
      "action": "allow",
      "page": "*"
    }
  ]
}
```

- [ ] **Step 3: Create the global app shell**

`travel-checkin-mini-program/app.js`

```javascript
App({
  onLaunch() {
    if (wx.cloud) {
      wx.cloud.init({
        traceUser: true
      });
    }
  }
});
```

`travel-checkin-mini-program/app.json`

```json
{
  "pages": [
    "pages/home/index",
    "pages/catalog/index",
    "pages/spot/index",
    "pages/profile/index"
  ],
  "window": {
    "navigationBarTitleText": "旅行打卡",
    "navigationBarBackgroundColor": "#ffffff",
    "navigationBarTextStyle": "black",
    "backgroundColor": "#f6f7fb",
    "backgroundTextStyle": "light"
  },
  "tabBar": {
    "color": "#7a7f8a",
    "selectedColor": "#1677ff",
    "backgroundColor": "#ffffff",
    "list": [
      {
        "pagePath": "pages/home/index",
        "text": "首页"
      },
      {
        "pagePath": "pages/catalog/index",
        "text": "图鉴"
      },
      {
        "pagePath": "pages/profile/index",
        "text": "我的"
      }
    ]
  },
  "style": "v2",
  "lazyCodeLoading": "requiredComponents"
}
```

`travel-checkin-mini-program/app.wxss`

```css
page {
  background: #f6f7fb;
  color: #1f2329;
  font-size: 28rpx;
}

.container {
  padding: 32rpx;
}

.card {
  background: #ffffff;
  border-radius: 24rpx;
  padding: 24rpx;
  box-shadow: 0 12rpx 30rpx rgba(31, 35, 41, 0.06);
}

.section-title {
  font-size: 32rpx;
  font-weight: 600;
  margin-bottom: 20rpx;
}

.primary-button {
  background: #1677ff;
  color: #ffffff;
  border-radius: 999rpx;
  padding: 20rpx 0;
  text-align: center;
}

.tag {
  display: inline-block;
  padding: 8rpx 20rpx;
  border-radius: 999rpx;
  font-size: 22rpx;
}

.tag-pending {
  background: #eef2ff;
  color: #4f46e5;
}

.tag-available {
  background: #e8fff1;
  color: #0f9d58;
}

.tag-complete {
  background: #fff4e5;
  color: #c97a00;
}
```

- [ ] **Step 4: Install dependencies**

Run: `cd travel-checkin-mini-program && npm install`
Expected: `added 2xx packages` and exit code `0`

- [ ] **Step 5: Verify the Jest command is wired**

Run: `cd travel-checkin-mini-program && npm test`
Expected: FAIL with `No tests found`

- [ ] **Step 6: Commit**

```bash
git add travel-checkin-mini-program/package.json travel-checkin-mini-program/project.config.json travel-checkin-mini-program/app.js travel-checkin-mini-program/app.json travel-checkin-mini-program/app.wxss travel-checkin-mini-program/sitemap.json

git commit -m "chore: bootstrap travel checkin mini program"
```

---

### Task 2: Add scenic spot seed data and geo rules

**Files:**
- Create: `travel-checkin-mini-program/shared/spots.js`
- Create: `travel-checkin-mini-program/shared/geo.js`
- Create: `travel-checkin-mini-program/miniprogram/data/spots.js`
- Create: `travel-checkin-mini-program/miniprogram/utils/geo.js`
- Create: `travel-checkin-mini-program/tests/geo.test.js`

- [ ] **Step 1: Write the failing geo test**

`travel-checkin-mini-program/tests/geo.test.js`

```javascript
const { calculateDistanceMeters, getSpotStatus } = require('../miniprogram/utils/geo');

describe('geo helpers', () => {
  test('calculateDistanceMeters returns 0 for identical coordinates', () => {
    expect(calculateDistanceMeters(30.2591, 120.1303, 30.2591, 120.1303)).toBe(0);
  });

  test('getSpotStatus returns available when user is inside radius', () => {
    expect(getSpotStatus({ radiusMeters: 200 }, 180, false)).toBe('available');
  });

  test('getSpotStatus returns checked when already completed', () => {
    expect(getSpotStatus({ radiusMeters: 200 }, 50, true)).toBe('checked');
  });

  test('getSpotStatus returns pending when user is outside radius', () => {
    expect(getSpotStatus({ radiusMeters: 200 }, 260, false)).toBe('pending');
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd travel-checkin-mini-program && npm test -- geo.test.js`
Expected: FAIL with `Cannot find module '../miniprogram/utils/geo'`

- [ ] **Step 3: Create the shared scenic spot seed data**

`travel-checkin-mini-program/shared/spots.js`

```javascript
module.exports = [
  {
    id: 'west-lake-broken-bridge',
    name: '断桥残雪',
    scopeType: 'city',
    scopeId: 'hangzhou-west-lake',
    lat: 30.260289,
    lng: 120.151899,
    coverImage: 'https://images.unsplash.com/photo-1506744038136-46273834b3fb?auto=format&fit=crop&w=800&q=80',
    intro: '西湖北线经典景点，适合作为首个解锁点。',
    radiusMeters: 220,
    badgeName: '西湖初探'
  },
  {
    id: 'west-lake-leifeng',
    name: '雷峰塔',
    scopeType: 'city',
    scopeId: 'hangzhou-west-lake',
    lat: 30.231085,
    lng: 120.148585,
    coverImage: 'https://images.unsplash.com/photo-1472396961693-142e6e269027?auto=format&fit=crop&w=800&q=80',
    intro: '西湖地标性高点，适合作为进度中段目标。',
    radiusMeters: 260,
    badgeName: '古迹探索者'
  },
  {
    id: 'west-lake-sudi',
    name: '苏堤春晓',
    scopeType: 'city',
    scopeId: 'hangzhou-west-lake',
    lat: 30.248607,
    lng: 120.137416,
    coverImage: 'https://images.unsplash.com/photo-1500530855697-b586d89ba3ee?auto=format&fit=crop&w=800&q=80',
    intro: '适合步行打卡的西湖经典线路点位。',
    radiusMeters: 240,
    badgeName: '步行收藏家'
  }
];
```

- [ ] **Step 4: Implement the shared geo helper**

`travel-checkin-mini-program/shared/geo.js`

```javascript
function toRadians(value) {
  return (value * Math.PI) / 180;
}

function calculateDistanceMeters(lat1, lng1, lat2, lng2) {
  if (lat1 === lat2 && lng1 === lng2) {
    return 0;
  }

  const earthRadius = 6371000;
  const dLat = toRadians(lat2 - lat1);
  const dLng = toRadians(lng2 - lng1);
  const startLat = toRadians(lat1);
  const endLat = toRadians(lat2);

  const a =
    Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.cos(startLat) * Math.cos(endLat) * Math.sin(dLng / 2) * Math.sin(dLng / 2);

  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  return Math.round(earthRadius * c);
}

function getSpotStatus(spot, distanceMeters, alreadyChecked) {
  if (alreadyChecked) {
    return 'checked';
  }

  return distanceMeters <= spot.radiusMeters ? 'available' : 'pending';
}

module.exports = {
  calculateDistanceMeters,
  getSpotStatus
};
```

- [ ] **Step 5: Re-export the shared modules for mini program imports**

`travel-checkin-mini-program/miniprogram/data/spots.js`

```javascript
module.exports = require('../../shared/spots');
```

`travel-checkin-mini-program/miniprogram/utils/geo.js`

```javascript
module.exports = require('../../shared/geo');
```

- [ ] **Step 6: Run the test to verify it passes**

Run: `cd travel-checkin-mini-program && npm test -- geo.test.js`
Expected: PASS with `4 passed`

- [ ] **Step 7: Commit**

```bash
git add travel-checkin-mini-program/shared/spots.js travel-checkin-mini-program/shared/geo.js travel-checkin-mini-program/miniprogram/data/spots.js travel-checkin-mini-program/miniprogram/utils/geo.js travel-checkin-mini-program/tests/geo.test.js

git commit -m "feat: add scenic spot seed data and geo rules"
```

---

### Task 3: Build catalog and home data services

**Files:**
- Create: `travel-checkin-mini-program/miniprogram/services/spot-service.js`
- Create: `travel-checkin-mini-program/tests/spot-service.test.js`

- [ ] **Step 1: Write the failing spot service test**

`travel-checkin-mini-program/tests/spot-service.test.js`

```javascript
const spots = require('../miniprogram/data/spots');
const {
  buildCatalogItems,
  buildHomeSummary
} = require('../miniprogram/services/spot-service');

describe('spot service', () => {
  const location = { lat: 30.2602, lng: 120.1518 };
  const checkedIds = [];

  test('buildCatalogItems sorts the nearest spot first', () => {
    const items = buildCatalogItems(spots, location, checkedIds);
    expect(items[0].id).toBe('west-lake-broken-bridge');
  });

  test('buildCatalogItems marks nearby spots as available', () => {
    const items = buildCatalogItems(spots, location, checkedIds);
    expect(items[0].status).toBe('available');
  });

  test('buildHomeSummary returns progress counts and nearest list', () => {
    const summary = buildHomeSummary(spots, location, ['west-lake-sudi']);
    expect(summary.checkedCount).toBe(1);
    expect(summary.totalCount).toBe(3);
    expect(summary.nearest.length).toBe(3);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd travel-checkin-mini-program && npm test -- spot-service.test.js`
Expected: FAIL with `Cannot find module '../miniprogram/services/spot-service'`

- [ ] **Step 3: Implement the spot service**

`travel-checkin-mini-program/miniprogram/services/spot-service.js`

```javascript
const { calculateDistanceMeters, getSpotStatus } = require('../utils/geo');

function buildCatalogItems(spots, location, checkedIds) {
  const checkedSet = new Set(checkedIds);

  return spots
    .map((spot) => {
      const distanceMeters = location
        ? calculateDistanceMeters(location.lat, location.lng, spot.lat, spot.lng)
        : Number.MAX_SAFE_INTEGER;

      return {
        ...spot,
        distanceMeters,
        status: location
          ? getSpotStatus(spot, distanceMeters, checkedSet.has(spot.id))
          : checkedSet.has(spot.id)
            ? 'checked'
            : 'pending'
      };
    })
    .sort((left, right) => left.distanceMeters - right.distanceMeters);
}

function buildHomeSummary(spots, location, checkedIds) {
  const items = buildCatalogItems(spots, location, checkedIds);

  return {
    scopeLabel: '杭州西湖',
    checkedCount: checkedIds.length,
    totalCount: spots.length,
    completionRate: Math.round((checkedIds.length / spots.length) * 100),
    nearest: items.slice(0, 3)
  };
}

module.exports = {
  buildCatalogItems,
  buildHomeSummary
};
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `cd travel-checkin-mini-program && npm test -- spot-service.test.js`
Expected: PASS with `3 passed`

- [ ] **Step 5: Commit**

```bash
git add travel-checkin-mini-program/miniprogram/services/spot-service.js travel-checkin-mini-program/tests/spot-service.test.js

git commit -m "feat: add catalog and home view services"
```

---

### Task 4: Implement the home and catalog pages

**Files:**
- Create: `travel-checkin-mini-program/miniprogram/pages/home/index.js`
- Create: `travel-checkin-mini-program/miniprogram/pages/home/index.json`
- Create: `travel-checkin-mini-program/miniprogram/pages/home/index.wxml`
- Create: `travel-checkin-mini-program/miniprogram/pages/home/index.wxss`
- Create: `travel-checkin-mini-program/miniprogram/pages/catalog/index.js`
- Create: `travel-checkin-mini-program/miniprogram/pages/catalog/index.json`
- Create: `travel-checkin-mini-program/miniprogram/pages/catalog/index.wxml`
- Create: `travel-checkin-mini-program/miniprogram/pages/catalog/index.wxss`

- [ ] **Step 1: Add reusable location lookup helper to `home/index.js`**

`travel-checkin-mini-program/miniprogram/pages/home/index.js`

```javascript
const spots = require('../../data/spots');
const { buildHomeSummary } = require('../../services/spot-service');

function getCurrentLocation() {
  return new Promise((resolve, reject) => {
    wx.getLocation({
      type: 'gcj02',
      success: resolve,
      fail: reject
    });
  });
}

Page({
  data: {
    loading: true,
    summary: null,
    locationError: ''
  },

  onShow() {
    this.loadPage();
  },

  async loadPage() {
    const checkedIds = wx.getStorageSync('checkedSpotIds') || [];

    try {
      const location = await getCurrentLocation();
      const summary = buildHomeSummary(spots, { lat: location.latitude, lng: location.longitude }, checkedIds);
      this.setData({ summary, loading: false, locationError: '' });
    } catch (error) {
      const summary = buildHomeSummary(spots, null, checkedIds);
      this.setData({
        summary,
        loading: false,
        locationError: '定位未开启，先浏览图鉴，到了现场再打卡。'
      });
    }
  },

  openSpot(event) {
    const { id } = event.currentTarget.dataset;
    wx.navigateTo({ url: `/pages/spot/index?id=${id}` });
  },

  openCatalog() {
    wx.switchTab({ url: '/pages/catalog/index' });
  }
});
```

- [ ] **Step 2: Add home page view files**

`travel-checkin-mini-program/miniprogram/pages/home/index.json`

```json
{
  "navigationBarTitleText": "旅行打卡首页"
}
```

`travel-checkin-mini-program/miniprogram/pages/home/index.wxml`

```xml
<view class="container">
  <view class="card hero" wx:if="{{summary}}">
    <view class="hero-title">{{summary.scopeLabel}}</view>
    <view class="hero-progress">已打卡 {{summary.checkedCount}} / {{summary.totalCount}}</view>
    <view class="hero-rate">完成度 {{summary.completionRate}}%</view>
    <view class="hero-tip" wx:if="{{locationError}}">{{locationError}}</view>
    <view class="primary-button" bindtap="openCatalog">查看图鉴</view>
  </view>

  <view class="section-title">离你最近的景点</view>
  <view class="card spot-card" wx:for="{{summary.nearest}}" wx:key="id" data-id="{{item.id}}" bindtap="openSpot">
    <view class="spot-name">{{item.name}}</view>
    <view class="spot-meta">{{item.distanceMeters === 9007199254740991 ? '待定位' : item.distanceMeters + 'm'}} · {{item.intro}}</view>
    <view class="tag {{item.status === 'checked' ? 'tag-complete' : item.status === 'available' ? 'tag-available' : 'tag-pending'}}">
      {{item.status === 'checked' ? '已打卡' : item.status === 'available' ? '附近可打卡' : '未解锁'}}
    </view>
  </view>
</view>
```

`travel-checkin-mini-program/miniprogram/pages/home/index.wxss`

```css
.hero {
  margin-bottom: 32rpx;
}

.hero-title {
  font-size: 40rpx;
  font-weight: 700;
}

.hero-progress,
.hero-rate,
.hero-tip {
  margin-top: 12rpx;
  color: #5b6472;
}

.hero-tip {
  color: #d46b08;
}

.spot-card {
  margin-bottom: 20rpx;
}

.spot-name {
  font-size: 30rpx;
  font-weight: 600;
  margin-bottom: 8rpx;
}

.spot-meta {
  color: #5b6472;
  margin-bottom: 16rpx;
}
```

- [ ] **Step 3: Add catalog page files**

`travel-checkin-mini-program/miniprogram/pages/catalog/index.js`

```javascript
const spots = require('../../data/spots');
const { buildCatalogItems } = require('../../services/spot-service');

function getCurrentLocation() {
  return new Promise((resolve, reject) => {
    wx.getLocation({
      type: 'gcj02',
      success: resolve,
      fail: reject
    });
  });
}

Page({
  data: {
    activeFilter: 'all',
    items: []
  },

  onShow() {
    this.loadCatalog();
  },

  async loadCatalog() {
    const checkedIds = wx.getStorageSync('checkedSpotIds') || [];

    try {
      const location = await getCurrentLocation();
      const items = buildCatalogItems(spots, { lat: location.latitude, lng: location.longitude }, checkedIds);
      this.setData({ items });
    } catch (error) {
      const items = buildCatalogItems(spots, null, checkedIds);
      this.setData({ items });
    }
  },

  setFilter(event) {
    this.setData({ activeFilter: event.currentTarget.dataset.filter });
  },

  openSpot(event) {
    wx.navigateTo({ url: `/pages/spot/index?id=${event.currentTarget.dataset.id}` });
  }
});
```

`travel-checkin-mini-program/miniprogram/pages/catalog/index.json`

```json
{
  "navigationBarTitleText": "景点图鉴"
}
```

`travel-checkin-mini-program/miniprogram/pages/catalog/index.wxml`

```xml
<view class="container">
  <view class="filters">
    <view class="filter {{activeFilter === 'all' ? 'filter-active' : ''}}" data-filter="all" bindtap="setFilter">全部</view>
    <view class="filter {{activeFilter === 'available' ? 'filter-active' : ''}}" data-filter="available" bindtap="setFilter">附近可打卡</view>
    <view class="filter {{activeFilter === 'checked' ? 'filter-active' : ''}}" data-filter="checked" bindtap="setFilter">已打卡</view>
  </view>

  <view
    class="card spot-card"
    wx:for="{{items}}"
    wx:key="id"
    wx:if="{{activeFilter === 'all' || activeFilter === item.status}}"
    data-id="{{item.id}}"
    bindtap="openSpot"
  >
    <view class="spot-row">
      <image class="cover" src="{{item.coverImage}}" mode="aspectFill"></image>
      <view class="spot-body">
        <view class="spot-name">{{item.name}}</view>
        <view class="spot-meta">{{item.distanceMeters === 9007199254740991 ? '待定位' : item.distanceMeters + 'm'}}</view>
        <view class="tag {{item.status === 'checked' ? 'tag-complete' : item.status === 'available' ? 'tag-available' : 'tag-pending'}}">
          {{item.status === 'checked' ? '已打卡' : item.status === 'available' ? '附近可打卡' : '未解锁'}}
        </view>
      </view>
    </view>
  </view>
</view>
```

`travel-checkin-mini-program/miniprogram/pages/catalog/index.wxss`

```css
.filters {
  display: flex;
  gap: 16rpx;
  margin-bottom: 24rpx;
}

.filter {
  background: #ffffff;
  border-radius: 999rpx;
  padding: 14rpx 24rpx;
  color: #5b6472;
}

.filter-active {
  background: #1677ff;
  color: #ffffff;
}

.spot-card {
  margin-bottom: 20rpx;
}

.spot-row {
  display: flex;
  gap: 20rpx;
  align-items: center;
}

.cover {
  width: 160rpx;
  height: 160rpx;
  border-radius: 20rpx;
  background: #eef2f7;
}

.spot-body {
  flex: 1;
}
```

- [ ] **Step 4: Verify in WeChat DevTools**

Open `travel-checkin-mini-program` in WeChat DevTools.
Expected:
- Tab bar shows `首页`, `图鉴`, `我的`
- Home page renders a hero card and nearest scenic spot list
- Catalog page shows all three scenic spots with filter chips

- [ ] **Step 5: Commit**

```bash
git add travel-checkin-mini-program/miniprogram/pages/home/index.js travel-checkin-mini-program/miniprogram/pages/home/index.json travel-checkin-mini-program/miniprogram/pages/home/index.wxml travel-checkin-mini-program/miniprogram/pages/home/index.wxss travel-checkin-mini-program/miniprogram/pages/catalog/index.js travel-checkin-mini-program/miniprogram/pages/catalog/index.json travel-checkin-mini-program/miniprogram/pages/catalog/index.wxml travel-checkin-mini-program/miniprogram/pages/catalog/index.wxss

git commit -m "feat: add home and catalog pages"
```

---

### Task 5: Implement check-in rules and the scenic spot detail page

**Files:**
- Create: `travel-checkin-mini-program/miniprogram/services/checkin-service.js`
- Create: `travel-checkin-mini-program/miniprogram/pages/spot/index.js`
- Create: `travel-checkin-mini-program/miniprogram/pages/spot/index.json`
- Create: `travel-checkin-mini-program/miniprogram/pages/spot/index.wxml`
- Create: `travel-checkin-mini-program/miniprogram/pages/spot/index.wxss`
- Create: `travel-checkin-mini-program/tests/checkin-service.test.js`

- [ ] **Step 1: Write the failing check-in service test**

`travel-checkin-mini-program/tests/checkin-service.test.js`

```javascript
const {
  canCheckIn,
  mergeCheckedIds,
  buildCheckinPayload
} = require('../miniprogram/services/checkin-service');

describe('checkin service', () => {
  const spot = {
    id: 'west-lake-broken-bridge',
    radiusMeters: 220,
    lat: 30.260289,
    lng: 120.151899
  };

  test('canCheckIn allows users inside radius', () => {
    const result = canCheckIn(spot, { lat: 30.2603, lng: 120.1519 }, []);
    expect(result.ok).toBe(true);
  });

  test('canCheckIn blocks duplicate check-ins', () => {
    const result = canCheckIn(spot, { lat: 30.2603, lng: 120.1519 }, ['west-lake-broken-bridge']);
    expect(result.reason).toBe('already-checked');
  });

  test('mergeCheckedIds keeps ids unique', () => {
    expect(mergeCheckedIds(['a'], 'a')).toEqual(['a']);
  });

  test('buildCheckinPayload returns cloud payload fields', () => {
    const payload = buildCheckinPayload(spot, { lat: 30.2603, lng: 120.1519 }, ['photo-a'], '到了');
    expect(payload.spotId).toBe('west-lake-broken-bridge');
    expect(payload.photoList).toEqual(['photo-a']);
    expect(payload.note).toBe('到了');
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd travel-checkin-mini-program && npm test -- checkin-service.test.js`
Expected: FAIL with `Cannot find module '../miniprogram/services/checkin-service'`

- [ ] **Step 3: Implement `checkin-service.js`**

`travel-checkin-mini-program/miniprogram/services/checkin-service.js`

```javascript
const { calculateDistanceMeters } = require('../utils/geo');

function canCheckIn(spot, location, checkedIds) {
  if (checkedIds.includes(spot.id)) {
    return { ok: false, reason: 'already-checked', distanceMeters: 0 };
  }

  const distanceMeters = calculateDistanceMeters(location.lat, location.lng, spot.lat, spot.lng);

  if (distanceMeters > spot.radiusMeters) {
    return { ok: false, reason: 'out-of-range', distanceMeters };
  }

  return { ok: true, reason: 'ok', distanceMeters };
}

function mergeCheckedIds(currentIds, nextId) {
  return Array.from(new Set([...currentIds, nextId]));
}

function buildCheckinPayload(spot, location, photoList, note) {
  return {
    spotId: spot.id,
    spotName: spot.name,
    lat: location.lat,
    lng: location.lng,
    photoList,
    note
  };
}

async function submitCheckin(payload) {
  return wx.cloud.callFunction({
    name: 'checkin',
    data: payload
  });
}

module.exports = {
  canCheckIn,
  mergeCheckedIds,
  buildCheckinPayload,
  submitCheckin
};
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `cd travel-checkin-mini-program && npm test -- checkin-service.test.js`
Expected: PASS with `4 passed`

- [ ] **Step 5: Add the scenic spot detail page**

`travel-checkin-mini-program/miniprogram/pages/spot/index.js`

```javascript
const spots = require('../../data/spots');
const {
  canCheckIn,
  mergeCheckedIds,
  buildCheckinPayload,
  submitCheckin
} = require('../../services/checkin-service');

function getCurrentLocation() {
  return new Promise((resolve, reject) => {
    wx.getLocation({
      type: 'gcj02',
      success: resolve,
      fail: reject
    });
  });
}

Page({
  data: {
    spot: null,
    locationError: '',
    statusText: '定位中...',
    note: ''
  },

  async onLoad(query) {
    const spot = spots.find((item) => item.id === query.id);
    this.setData({ spot });
    await this.refreshStatus();
  },

  async refreshStatus() {
    const checkedIds = wx.getStorageSync('checkedSpotIds') || [];

    try {
      const location = await getCurrentLocation();
      const result = canCheckIn(this.data.spot, { lat: location.latitude, lng: location.longitude }, checkedIds);
      this.currentLocation = { lat: location.latitude, lng: location.longitude };
      this.setData({
        locationError: '',
        statusText: result.ok
          ? '你已到达打卡范围，可以立即打卡'
          : result.reason === 'already-checked'
            ? '这个景点已经点亮过了'
            : `距离目标还有 ${result.distanceMeters - this.data.spot.radiusMeters} 米`
      });
    } catch (error) {
      this.setData({ locationError: '请先打开定位权限后再尝试打卡。', statusText: '暂时无法确认你是否在景点附近' });
    }
  },

  bindNoteInput(event) {
    this.setData({ note: event.detail.value });
  },

  async handleCheckin() {
    if (!this.currentLocation) {
      wx.showToast({ title: '请先开启定位', icon: 'none' });
      return;
    }

    const checkedIds = wx.getStorageSync('checkedSpotIds') || [];
    const result = canCheckIn(this.data.spot, this.currentLocation, checkedIds);

    if (!result.ok) {
      wx.showToast({ title: result.reason === 'already-checked' ? '已打卡过' : '还没到景点附近', icon: 'none' });
      return;
    }

    const payload = buildCheckinPayload(this.data.spot, this.currentLocation, [], this.data.note);
    await submitCheckin(payload);

    wx.setStorageSync('checkedSpotIds', mergeCheckedIds(checkedIds, this.data.spot.id));
    wx.showToast({ title: '打卡成功', icon: 'success' });
    await this.refreshStatus();
  }
});
```

`travel-checkin-mini-program/miniprogram/pages/spot/index.json`

```json
{
  "navigationBarTitleText": "景点详情"
}
```

`travel-checkin-mini-program/miniprogram/pages/spot/index.wxml`

```xml
<view class="container" wx:if="{{spot}}">
  <image class="hero-image" src="{{spot.coverImage}}" mode="aspectFill"></image>
  <view class="card detail-card">
    <view class="section-title">{{spot.name}}</view>
    <view class="intro">{{spot.intro}}</view>
    <view class="distance-tip">{{statusText}}</view>
    <view class="error-tip" wx:if="{{locationError}}">{{locationError}}</view>
    <textarea class="note-box" placeholder="写一句旅行心情（可选）" bindinput="bindNoteInput" value="{{note}}"></textarea>
    <view class="primary-button" bindtap="handleCheckin">立即打卡</view>
  </view>
</view>
```

`travel-checkin-mini-program/miniprogram/pages/spot/index.wxss`

```css
.hero-image {
  width: 100%;
  height: 360rpx;
}

.detail-card {
  margin-top: -40rpx;
  position: relative;
}

.intro,
.distance-tip,
.error-tip {
  color: #5b6472;
  margin-bottom: 16rpx;
}

.error-tip {
  color: #cf1322;
}

.note-box {
  width: 100%;
  min-height: 200rpx;
  background: #f6f7fb;
  border-radius: 20rpx;
  padding: 20rpx;
  box-sizing: border-box;
  margin-bottom: 24rpx;
}
```

- [ ] **Step 6: Verify in WeChat DevTools**

Expected:
- Opening a spot shows cover image, intro, status text, and a note box
- Out-of-range users see remaining distance text
- In-range users can tap `立即打卡` and get a success toast

- [ ] **Step 7: Commit**

```bash
git add travel-checkin-mini-program/miniprogram/services/checkin-service.js travel-checkin-mini-program/miniprogram/pages/spot/index.js travel-checkin-mini-program/miniprogram/pages/spot/index.json travel-checkin-mini-program/miniprogram/pages/spot/index.wxml travel-checkin-mini-program/miniprogram/pages/spot/index.wxss travel-checkin-mini-program/tests/checkin-service.test.js

git commit -m "feat: add spot detail checkin flow"
```

---

### Task 6: Add the cloud check-in function

**Files:**
- Create: `travel-checkin-mini-program/cloudfunctions/checkin/package.json`
- Create: `travel-checkin-mini-program/cloudfunctions/checkin/index.js`
- Create: `travel-checkin-mini-program/tests/cloud-checkin.test.js`

- [ ] **Step 1: Write the failing cloud function test**

`travel-checkin-mini-program/tests/cloud-checkin.test.js`

```javascript
const { validateCheckin } = require('../cloudfunctions/checkin/index');

describe('cloud checkin', () => {
  const spot = {
    id: 'west-lake-broken-bridge',
    lat: 30.260289,
    lng: 120.151899,
    radiusMeters: 220
  };

  test('validateCheckin accepts locations inside radius', () => {
    const result = validateCheckin(spot, { lat: 30.2603, lng: 120.1519 }, false);
    expect(result.ok).toBe(true);
  });

  test('validateCheckin rejects duplicate records', () => {
    const result = validateCheckin(spot, { lat: 30.2603, lng: 120.1519 }, true);
    expect(result.reason).toBe('already-checked');
  });

  test('validateCheckin rejects out-of-range requests', () => {
    const result = validateCheckin(spot, { lat: 30.2803, lng: 120.1819 }, false);
    expect(result.reason).toBe('out-of-range');
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd travel-checkin-mini-program && npm test -- cloud-checkin.test.js`
Expected: FAIL with `Cannot find module '../cloudfunctions/checkin/index'`

- [ ] **Step 3: Implement the cloud function**

`travel-checkin-mini-program/cloudfunctions/checkin/package.json`

```json
{
  "name": "checkin",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "wx-server-sdk": "latest"
  }
}
```

`travel-checkin-mini-program/cloudfunctions/checkin/index.js`

```javascript
const cloud = require('wx-server-sdk');
const spots = require('../../shared/spots');
const { calculateDistanceMeters } = require('../../shared/geo');

cloud.init({ env: cloud.DYNAMIC_CURRENT_ENV });
const db = cloud.database();

function validateCheckin(spot, location, alreadyChecked) {
  if (alreadyChecked) {
    return { ok: false, reason: 'already-checked' };
  }

  const distanceMeters = calculateDistanceMeters(location.lat, location.lng, spot.lat, spot.lng);
  if (distanceMeters > spot.radiusMeters) {
    return { ok: false, reason: 'out-of-range', distanceMeters };
  }

  return { ok: true, reason: 'ok', distanceMeters };
}

exports.validateCheckin = validateCheckin;

exports.main = async (event) => {
  const wxContext = cloud.getWXContext();
  const spot = spots.find((item) => item.id === event.spotId);

  if (!spot) {
    throw new Error('spot-not-found');
  }

  const existing = await db
    .collection('checkins')
    .where({ openId: wxContext.OPENID, spotId: event.spotId })
    .get();

  const validation = validateCheckin(
    spot,
    { lat: event.lat, lng: event.lng },
    existing.data.length > 0
  );

  if (!validation.ok) {
    throw new Error(validation.reason);
  }

  const record = {
    openId: wxContext.OPENID,
    spotId: event.spotId,
    spotName: event.spotName,
    lat: event.lat,
    lng: event.lng,
    note: event.note || '',
    photoList: event.photoList || [],
    checkinTime: Date.now()
  };

  await db.collection('checkins').add({ data: record });
  return { ok: true, record };
};
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `cd travel-checkin-mini-program && npm test -- cloud-checkin.test.js`
Expected: PASS with `3 passed`

- [ ] **Step 5: Deploy and verify in WeChat DevTools**

Expected:
- `cloudfunctions/checkin` deploys successfully
- Calling the detail page check-in button writes one document to the `checkins` collection
- Duplicate taps do not create a second document

- [ ] **Step 6: Commit**

```bash
git add travel-checkin-mini-program/cloudfunctions/checkin/package.json travel-checkin-mini-program/cloudfunctions/checkin/index.js travel-checkin-mini-program/tests/cloud-checkin.test.js

git commit -m "feat: add cloud-backed checkin function"
```

---

### Task 7: Add profile progress and finish MVP verification

**Files:**
- Create: `travel-checkin-mini-program/miniprogram/services/profile-service.js`
- Create: `travel-checkin-mini-program/miniprogram/pages/profile/index.js`
- Create: `travel-checkin-mini-program/miniprogram/pages/profile/index.json`
- Create: `travel-checkin-mini-program/miniprogram/pages/profile/index.wxml`
- Create: `travel-checkin-mini-program/miniprogram/pages/profile/index.wxss`
- Create: `travel-checkin-mini-program/tests/profile-service.test.js`

- [ ] **Step 1: Write the failing profile service test**

`travel-checkin-mini-program/tests/profile-service.test.js`

```javascript
const spots = require('../miniprogram/data/spots');
const { buildProfileViewModel } = require('../miniprogram/services/profile-service');

describe('profile service', () => {
  test('buildProfileViewModel returns progress summary and badge list', () => {
    const viewModel = buildProfileViewModel(spots, [
      { spotId: 'west-lake-broken-bridge', note: '第一站', checkinTime: 1713072000000 }
    ]);

    expect(viewModel.checkedCount).toBe(1);
    expect(viewModel.totalCount).toBe(3);
    expect(viewModel.badges).toContain('西湖初探');
    expect(viewModel.timeline[0].title).toBe('断桥残雪');
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd travel-checkin-mini-program && npm test -- profile-service.test.js`
Expected: FAIL with `Cannot find module '../miniprogram/services/profile-service'`

- [ ] **Step 3: Implement the profile service**

`travel-checkin-mini-program/miniprogram/services/profile-service.js`

```javascript
function buildProfileViewModel(spots, records) {
  const byId = new Map(spots.map((spot) => [spot.id, spot]));
  const checkedIds = records.map((record) => record.spotId);
  const badges = spots
    .filter((spot) => checkedIds.includes(spot.id))
    .map((spot) => spot.badgeName);

  return {
    checkedCount: checkedIds.length,
    totalCount: spots.length,
    badges,
    timeline: records
      .slice()
      .sort((left, right) => right.checkinTime - left.checkinTime)
      .map((record) => ({
        title: byId.get(record.spotId).name,
        note: record.note,
        timeText: new Date(record.checkinTime).toLocaleDateString('zh-CN')
      }))
  };
}

module.exports = {
  buildProfileViewModel
};
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `cd travel-checkin-mini-program && npm test -- profile-service.test.js`
Expected: PASS with `1 passed`

- [ ] **Step 5: Add the profile page**

`travel-checkin-mini-program/miniprogram/pages/profile/index.js`

```javascript
const spots = require('../../data/spots');
const { buildProfileViewModel } = require('../../services/profile-service');

Page({
  data: {
    profile: null
  },

  onShow() {
    const records = wx.getStorageSync('checkinRecords') || [];
    this.setData({
      profile: buildProfileViewModel(spots, records)
    });
  }
});
```

`travel-checkin-mini-program/miniprogram/pages/profile/index.json`

```json
{
  "navigationBarTitleText": "我的打卡"
}
```

`travel-checkin-mini-program/miniprogram/pages/profile/index.wxml`

```xml
<view class="container" wx:if="{{profile}}">
  <view class="card summary-card">
    <view class="summary-title">旅行进度</view>
    <view class="summary-value">{{profile.checkedCount}} / {{profile.totalCount}}</view>
  </view>

  <view class="section-title">已获得徽章</view>
  <view class="card badge-card">
    <view wx:if="{{profile.badges.length === 0}}">还没有徽章，先去打卡第一个景点。</view>
    <view class="badge" wx:for="{{profile.badges}}" wx:key="*this">{{item}}</view>
  </view>

  <view class="section-title">打卡时间线</view>
  <view class="card timeline-card" wx:for="{{profile.timeline}}" wx:key="title">
    <view class="timeline-title">{{item.title}}</view>
    <view class="timeline-note">{{item.note}}</view>
    <view class="timeline-time">{{item.timeText}}</view>
  </view>
</view>
```

`travel-checkin-mini-program/miniprogram/pages/profile/index.wxss`

```css
.summary-card,
.badge-card,
.timeline-card {
  margin-bottom: 20rpx;
}

.summary-title,
.timeline-title {
  font-size: 30rpx;
  font-weight: 600;
}

.summary-value,
.timeline-note,
.timeline-time {
  color: #5b6472;
  margin-top: 12rpx;
}

.badge {
  display: inline-block;
  margin-right: 16rpx;
  margin-bottom: 16rpx;
  padding: 12rpx 20rpx;
  border-radius: 999rpx;
  background: #fff4e5;
  color: #c97a00;
}
```

- [ ] **Step 6: Patch the detail page to cache records after success**

Update `travel-checkin-mini-program/miniprogram/pages/spot/index.js` inside `handleCheckin()` after `await submitCheckin(payload);`:

```javascript
const records = wx.getStorageSync('checkinRecords') || [];
wx.setStorageSync('checkinRecords', [
  {
    spotId: this.data.spot.id,
    note: this.data.note,
    checkinTime: Date.now()
  },
  ...records
]);
```

- [ ] **Step 7: Run the full automated suite**

Run: `cd travel-checkin-mini-program && npm test`
Expected: PASS with `5 test suites passed`

- [ ] **Step 8: Run the MVP acceptance sweep in WeChat DevTools**

Expected:
- User can browse spots without location permission
- Home and catalog update status when location is granted
- Out-of-range check-in is blocked
- In-range check-in succeeds once
- Profile page shows progress, badges, and timeline entry

- [ ] **Step 9: Commit**

```bash
git add travel-checkin-mini-program/miniprogram/services/profile-service.js travel-checkin-mini-program/miniprogram/pages/profile/index.js travel-checkin-mini-program/miniprogram/pages/profile/index.json travel-checkin-mini-program/miniprogram/pages/profile/index.wxml travel-checkin-mini-program/miniprogram/pages/profile/index.wxss travel-checkin-mini-program/miniprogram/pages/spot/index.js travel-checkin-mini-program/tests/profile-service.test.js

git commit -m "feat: add profile progress for travel checkins"
```
