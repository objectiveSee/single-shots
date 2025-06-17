# Live Activities Implementation Guide

A comprehensive guide to implementing iOS Live Activities in React Native/Expo apps, featuring a countdown timer example that can be adapted for any use case.

## Table of Contents

1. [Prerequisites and Setup](#prerequisites-and-setup)
2. [Project Configuration](#project-configuration)
3. [iOS Target Creation](#ios-target-creation)
4. [Native Expo Module Setup](#native-expo-module-setup)
5. [Swift Implementation](#swift-implementation)
6. [TypeScript Integration](#typescript-integration)
7. [React Integration](#react-integration)
8. [Architecture Insights](#architecture-insights)
9. [Testing and Troubleshooting](#testing-and-troubleshooting)
10. [Reference Materials](#reference-materials)

## Prerequisites and Setup

### Required Tools and Versions

| Tool | Version | Command to Check |
|------|---------|------------------|
| Xcode | 16.2+ | `xcodebuild -version` |
| CocoaPods | 1.16.2+ | `pod --version` |
| Node.js | 20+ | `node --version` |
| Expo CLI | Latest | `npx expo --version` |
| Bun | Latest | `bun --version` |

### Environment Configuration

Ensure you have:
- Apple Developer Account with Team ID
- iOS device or simulator running iOS 16.2+
- Expo development build configured
- Valid bundle identifier and provisioning profiles

### Dependencies

Add these to your [`package.json`](package.json):

```json
{
  "dependencies": {
    "expo": "~52.0.0",
    "jotai": "^2.0.0"
  },
  "devDependencies": {
    "@bacons/apple-targets": "latest"
  }
}
```

## Project Configuration

### 1. Update app.json

Configure your [`app.json`](app.json) with Live Activities support:

```json
{
  "expo": {
    "ios": {
      "bundleIdentifier": "your.bundle.identifier",
      "appleTeamId": "YOUR_TEAM_ID",
      "infoPlist": {
        "NSSupportsLiveActivities": true
      }
    },
    "plugins": [
      "@bacons/apple-targets"
    ]
  }
}
```

### 2. Update app.config.ts

Ensure your [`app.config.ts`](app.config.ts) includes the apple-targets plugin:

```typescript
import { ConfigContext, ExpoConfig } from "@expo/config"

module.exports = ({ config }: ConfigContext): Partial<ExpoConfig> => {
  const existingPlugins = config.plugins ?? []

  return {
    ...config,
    plugins: [...existingPlugins, "@bacons/apple-targets"],
  }
}
```

## iOS Target Creation

### 1. Install @bacons/apple-targets

```bash
bun add -D @bacons/apple-targets
```

### 2. Create Widget Target

```bash
npx create-target
```

Follow the prompts:
- Choose "Widget" type
- Name it "widget"
- Configure as needed

### 3. Clean Up Generated Files

Remove unnecessary files:

```bash
rm -rf targets/widget/WidgetControl.swift \
       targets/widget/widgets.swift \
       targets/widget/AppIntent.swift
```

### 4. Configure Target

Create [`targets/widget/expo-target.config.js`](targets/widget/expo-target.config.js):

```javascript
/** @type {import('@bacons/apple-targets/app.plugin').ConfigFunction} */
module.exports = () => ({
  type: "widget",
  icon: "../../assets/images/logo.png",
  entitlements: {},
  frameworks: ["SwiftUI", "ActivityKit"],
  deploymentTarget: "17.4",
})
```

### 5. Create Attributes Structure

Create [`targets/widget/Attributes.swift`](targets/widget/Attributes.swift):

```swift
import ActivityKit
import Foundation

struct Attributes: ActivityAttributes {
  public struct ContentState: Codable, Hashable {
    var endDate: Date
    var title: String
    var color: String?
    var subtitle: String?
    var alertMessage: String?
  }
}
```

### 6. Create Live Activity Widget

Create [`targets/widget/WidgetLiveActivity.swift`](targets/widget/WidgetLiveActivity.swift):

```swift
import ActivityKit
import SwiftUI
import WidgetKit

struct WidgetLiveActivity: Widget {
  var body: some WidgetConfiguration {
    ActivityConfiguration(for: Attributes.self) { context in
      CountdownView(contentState: context.state).padding(8)
    } dynamicIsland: { context in
      DynamicIsland {
        DynamicIslandExpandedRegion(.leading) {
          CountdownView(contentState: context.state)
        }
      }
      compactLeading: { EmptyView() }
      compactTrailing: {
        CountdownView(contentState: context.state, style: .compact)
      } minimal: { EmptyView() }
    }
  }
}
```

### 7. Create Countdown View

Create [`targets/widget/CountdownView.swift`](targets/widget/CountdownView.swift):

```swift
import Combine
import SwiftUI

enum CountdownStyle {
  case large
  case compact

  var fontSize: CGFloat {
    switch self {
    case .large: 32
    case .compact: 14
    }
  }

  var showProgress: Bool {
    self == .large
  }

  var spacing: CGFloat {
    self == .compact ? 0 : 8
  }
}

private struct FormattedCountdownView: View {
  let endDate: Date
  @State private var now: Date = .init()
  private let timer = Timer.publish(every: 1, on: .main, in: .common).autoconnect()

  var body: some View {
    Text(formattedTime)
      .font(.system(size: 16, design: .monospaced))
      .foregroundColor(.white)
      .onReceive(timer) { _ in
        now = Date()
      }
  }

  private var formattedTime: String {
    let remaining = max(Int(endDate.timeIntervalSince(now)), 0)
    let days = remaining / 86400
    let hours = (remaining % 86400) / 3600
    let minutes = (remaining % 3600) / 60
    let seconds = remaining % 60

    if days > 0 {
      return "\(days)d \(hours)h"
    } else if hours > 0 {
      return "\(hours)h \(minutes)m"
    } else {
      return "\(minutes)m \(seconds)s"
    }
  }
}

struct CountdownView: View {
  let contentState: Attributes.ContentState
  var style: CountdownStyle = .large

  var body: some View {
    VStack(spacing: style.spacing) {
      switch style {
      case .large:
        VStack(spacing: 0) {
          HStack(alignment: .top) {
            VStack(alignment: .leading, spacing: 2) {
              Text(contentState.title)
                .font(.custom("SF Pro Text", size: 16).weight(.bold))
                .foregroundColor(.white)
              if let subtitle = contentState.subtitle {
                Text(subtitle)
                  .font(.custom("SF Pro Text", size: 16).weight(.bold))
                  .foregroundColor(.gray)
              }
            }
            Spacer()
            FormattedCountdownView(endDate: contentState.endDate)
              .font(.custom("SF Pro Text", size: 16).weight(.medium))
              .foregroundColor(.white)
              .frame(minWidth: 60, alignment: .trailing)
          }
          .padding(.horizontal, 24)
          .padding(.bottom, 16)
          if let alertMessage = contentState.alertMessage {
            Text(alertMessage)
              .font(.custom("SF Pro Display", size: 13).weight(.regular))
              .foregroundColor(.white)
              .multilineTextAlignment(.leading)
              .frame(maxWidth: .infinity, alignment: .leading)
              .padding(.horizontal, 24)
              .padding(.vertical, 16)
          }
        }
      case .compact:
        HStack(spacing: 8) {
          Text(contentState.title)
            .font(.custom("SF Pro Text", size: 14).weight(.medium))
            .foregroundColor(.white)
            .lineLimit(1)
          Spacer()
          FormattedCountdownView(endDate: contentState.endDate)
            .font(.custom("SF Pro Text", size: 14).weight(.medium))
            .foregroundColor(.white)
        }
        .padding(.horizontal, 16)
      }
    }
  }
}
```

## Native Expo Module Setup

### 1. Create Local Expo Module

```bash
npx create-expo-module@latest --local
# Name: live-activity-controller
# Keep defaults for other options
```

### 2. Clean Up Generated Files

```bash
rm -rf \
  modules/live-activity-controller/android \
  modules/live-activity-controller/src/LiveActivityControllerModule.web.ts \
  modules/live-activity-controller/ios/LiveActivityControllerView.swift \
  modules/live-activity-controller/src/LiveActivityControllerView.*
```

### 3. Configure Module

Update [`modules/live-activity-controller/expo-module.config.json`](modules/live-activity-controller/expo-module.config.json):

```json
{
  "platforms": [
    "apple"
  ],
  "apple": {
    "modules": [
      "LiveActivityControllerModule"
    ]
  }
}
```

### 4. Create TypeScript Types

Create [`modules/live-activity-controller/src/LiveActivityController.types.ts`](modules/live-activity-controller/src/LiveActivityController.types.ts):

```typescript
export type LiveActivityParams = {
  endDate: number
  title: string
  color?: string
  subtitle?: string
  alertMessage?: string
}

export type StartLiveActivityFn = (params: LiveActivityParams) => Promise<{ activityId: string }>
export type UpdateLiveActivityFn = (params: LiveActivityParams) => Promise<void>
export type StopLiveActivityFn = () => Promise<void>
export type IsLiveActivityRunningFn = () => boolean
```

### 5. Create TypeScript Module

Create [`modules/live-activity-controller/src/LiveActivityControllerModule.ts`](modules/live-activity-controller/src/LiveActivityControllerModule.ts):

```typescript
import { requireNativeModule } from "expo"

import * as types from "./LiveActivityController.types"

const nativeModule = requireNativeModule("LiveActivityController")

export const startLiveActivity: types.StartLiveActivityFn = async (params) => {
  const stringParams = JSON.stringify(params)
  return nativeModule.startLiveActivity(stringParams)
}

export const updateLiveActivity: types.UpdateLiveActivityFn = async (params) => {
  const stringParams = JSON.stringify(params)
  return nativeModule.updateLiveActivity(stringParams)
}

export const stopLiveActivity: types.StopLiveActivityFn = async () => {
  return nativeModule.stopLiveActivity()
}

export const isLiveActivityRunning: types.IsLiveActivityRunningFn = () => {
  return nativeModule.isActivityRunning()
}

export const areLiveActivitiesEnabled: boolean = nativeModule.areLiveActivitiesEnabled
```

### 6. Create Android Stub

Create [`modules/live-activity-controller/src/LiveActivityControllerModule.android.ts`](modules/live-activity-controller/src/LiveActivityControllerModule.android.ts):

```typescript
import * as types from "./LiveActivityController.types"

export const startLiveActivity: types.StartLiveActivityFn = async () => {
  return Promise.resolve({ activityId: "" })
}

export const updateLiveActivity: types.UpdateLiveActivityFn = async () => {
  return
}

export const stopLiveActivity: types.StopLiveActivityFn = async () => {
  return
}

export const isLiveActivityRunning: types.IsLiveActivityRunningFn = () => {
  return false
}

export const areLiveActivitiesEnabled: boolean = false
```

### 7. Create Barrel Export

Create [`modules/live-activity-controller/index.ts`](modules/live-activity-controller/index.ts):

```typescript
export * from "./src/LiveActivityController.types"
export * from "./src/LiveActivityControllerModule"
```

## Swift Implementation

### 1. Copy Attributes to Module

```bash
cp targets/widget/Attributes.swift modules/live-activity-controller/ios/Attributes.swift
```

### 2. Create Swift Module

Create [`modules/live-activity-controller/ios/LiveActivityControllerModule.swift`](modules/live-activity-controller/ios/LiveActivityControllerModule.swift):

```swift
import ActivityKit
import ExpoModulesCore
import SwiftUI

// MARK: Exceptions

final class ActivityUnavailableException: GenericException<Void> {
  override var reason: String {
    "Live activities are not available on this system."
  }
}

final class ActivityFailedToStartException: GenericException<Void> {
  override var reason: String {
    "Live activity couldn't be launched."
  }
}

final class ActivityNotStartedException: GenericException<Void> {
  override var reason: String {
    "Live activity has not started yet."
  }
}

final class ActivityAlreadyRunningException: GenericException<Void> {
  override var reason: String {
    "Live activity is already running."
  }
}

final class ActivityDataException: GenericException<String> {
  override var reason: String {
    "The data passed down to the Live Activity is incorrect. \(param)"
  }
}

// MARK: Types

struct StartActivityArgs: Codable {
  let endDate: Double
  let title: String
  let color: String?
  let subtitle: String?
  let alertMessage: String?

  public static func fromJSON(rawData: String) -> Self? {
    let decoder = JSONDecoder()
    return try? decoder.decode(self, from: Data(rawData.utf8))
  }
}

protocol ActivityWrapper {}

@available(iOS 16.2, *)
class DefinedActivityWrapper: ActivityWrapper {
  private var activity: Activity<Attributes>

  init(_ activity: Activity<Attributes>) {
    self.activity = activity
  }

  public func setActivity(activity: Activity<Attributes>) {
    self.activity = activity
  }

  public func getActivity() -> Activity<Attributes> {
    activity
  }
}

struct FallbackActivityWrapper: ActivityWrapper {}

struct StartActivityReturnType: Record {
  @Field
  var activityId: String
}

// MARK: Helper functions

func getCurrentActivity() -> ActivityWrapper? {
  guard #available(iOS 16.2, *) else {
    return nil
  }

  if let activity = Activity<Attributes>.activities.first {
    return DefinedActivityWrapper(activity)
  } else {
    return nil
  }
}

func isActivityRunning() -> Bool {
  getCurrentActivity() != nil
}

// MARK: Module definition

public class LiveActivityControllerModule: Module {
  private var activityWrapper: ActivityWrapper?

  public func definition() -> ModuleDefinition {
    Name("LiveActivityController")

    Property("areLiveActivitiesEnabled") {
      if #available(iOS 16.2, *) {
        return ActivityAuthorizationInfo().areActivitiesEnabled
      }
      return false
    }

    AsyncFunction("startLiveActivity") {
      (rawData: String,
       _: Promise) in
      guard #available(iOS 16.2, *) else {
        throw ActivityUnavailableException(())
      }

      guard let args = StartActivityArgs.fromJSON(rawData: rawData) else {
        throw ActivityDataException(rawData)
      }

      guard isActivityRunning() == false else {
        throw ActivityAlreadyRunningException(())
      }

      let info = ActivityAuthorizationInfo()
      guard info.areActivitiesEnabled else {
        throw ActivityUnavailableException(())
      }

      do {
        let activityAttrs = Attributes()
        let endDate = Date(timeIntervalSince1970: args.endDate / 1000)

        let activityState = Attributes.ContentState(endDate: endDate, title: args.title,
                                                    color: args.color, subtitle: args.subtitle, alertMessage: args.alertMessage)

        let activity = try Activity.request(attributes: activityAttrs,
                                            content: .init(state: activityState, staleDate: endDate))

        log.debug("Started \(activity.id)")

        return StartActivityReturnType(activityId: Field(wrappedValue: activity.id))
      } catch {
        log.error("Failed to start live activity: \(error.localizedDescription)")
        throw ActivityFailedToStartException(())
      }
    }

    AsyncFunction("updateLiveActivity") { (
      rawData: String,
      promise: Promise
    ) in
      guard #available(iOS 16.2, *) else {
        throw ActivityUnavailableException(())
      }

      guard let activity = (getCurrentActivity() as? DefinedActivityWrapper)?.getActivity() else {
        throw ActivityNotStartedException(())
      }

      guard let args = StartActivityArgs.fromJSON(rawData: rawData) else {
        throw ActivityDataException(rawData)
      }

      let contentState: Attributes.ContentState

      let endDate = Date(timeIntervalSince1970: args.endDate / 1000)

      contentState = Attributes.ContentState(endDate: endDate, title: args.title,
                                             color: args.color, subtitle: args.subtitle, alertMessage: args.alertMessage)

      Task {
        await activity.update(ActivityContent<Attributes.ContentState>(state: contentState,
                                                                       staleDate: endDate))

        log.debug("Updated activity \(activity.id)")
        return promise.resolve()
      }
    }

    AsyncFunction("stopLiveActivity") { (promise: Promise) in
      guard #available(iOS 16.2, *) else {
        throw ActivityUnavailableException(())
      }

      guard let activity = (getCurrentActivity() as? DefinedActivityWrapper)?.getActivity() else {
        throw ActivityNotStartedException(())
      }

      log.debug("Stopping activity \(activity.id)")

      Task {
        await activity.end(nil, dismissalPolicy: .immediate)
        log.debug("Stopped activity \(activity.id)")
        return promise.resolve()
      }
    }

    Function("isActivityRunning") { () -> Bool in
      return isActivityRunning()
    }
  }
}
```

## TypeScript Integration

### 1. Create Live Activity Hook

Create [`app/hooks/useLiveActivity.ts`](app/hooks/useLiveActivity.ts):

```typescript
import { atom, useAtom } from "jotai"
import { useCallback, useEffect } from "react"

import type { LiveActivityParams } from "../../modules/live-activity-controller/src/LiveActivityController.types"
import {
  isLiveActivityRunning,
  startLiveActivity,
  stopLiveActivity,
  updateLiveActivity,
} from "../../modules/live-activity-controller/src/LiveActivityControllerModule"

type LiveActivityState = {
  isRunning: boolean
  lastMessage: LiveActivityParams | null
}

const liveActivityAtom = atom<LiveActivityState>({
  isRunning: false,
  lastMessage: null,
})

export function useLiveActivity() {
  const [state, setState] = useAtom(liveActivityAtom)

  // Sync state with native on mount
  useEffect(() => {
    const running = isLiveActivityRunning()
    setState((prev) => ({
      ...prev,
      isRunning: running,
      lastMessage: running ? prev.lastMessage : null,
    }))
  }, [setState])

  const start = useCallback(
    async (params: LiveActivityParams) => {
      await startLiveActivity(params)
      setState({ isRunning: true, lastMessage: params })
    },
    [setState],
  )

  const update = useCallback(
    async (params: LiveActivityParams) => {
      await updateLiveActivity(params)
      setState((prev) => ({ ...prev, lastMessage: params }))
    },
    [setState],
  )

  const stop = useCallback(async () => {
    await stopLiveActivity()
    setState({ isRunning: false, lastMessage: null })
  }, [setState])

  return {
    isRunning: state.isRunning,
    lastMessage: state.lastMessage,
    start,
    update,
    stop,
  }
}
```

### 2. Create Helper Functions

Create [`app/utils/liveActivityHelpers.ts`](app/utils/liveActivityHelpers.ts):

```typescript
// Generic data type for items with countdown timers
export interface CountdownItem {
  id: string
  endDate: string
  title: string
  subtitle?: string
  status: 'active' | 'ended' | 'paused'
}

export function findItemEndingSoonest(items: CountdownItem[]): CountdownItem | null {
  if (!items || items.length === 0) return null

  const activeItems = items.filter((item) => {
    if (!item?.endDate || !item?.status) return false

    const endTime = new Date(item.endDate).getTime()
    const now = Date.now()

    return endTime > now && item.status === 'active'
  })

  if (activeItems.length === 0) return null

  return activeItems.reduce((soonest, current) => {
    const soonestEndTime = new Date(soonest.endDate!).getTime()
    const currentEndTime = new Date(current.endDate!).getTime()

    return currentEndTime < soonestEndTime ? current : soonest
  })
}

export function checkIfItemEndedRecently(item: CountdownItem, withinMinutes: number = 10): boolean {
  if (!item?.endDate) return false

  const endTime = new Date(item.endDate).getTime()
  const now = Date.now()
  const withinMs = withinMinutes * 60 * 1000

  return now > endTime && now - endTime <= withinMs
}

export function shouldStartLiveActivity(
  currentItem: CountdownItem | null,
  previousItem: CountdownItem | null,
  preferences: { enableLiveActivities: boolean },
): boolean {
  if (!preferences.enableLiveActivities) {
    return false
  }

  if (!currentItem) return false

  if (!previousItem) return true

  return currentItem.id !== previousItem.id
}

export function calculateTimeUntilEnd(item: CountdownItem): number {
  if (!item?.endDate) return 0

  const endTime = new Date(item.endDate).getTime()
  const now = Date.now()

  return Math.max(0, endTime - now)
}

export function formatTimeRemaining(milliseconds: number): string {
  if (milliseconds <= 0) return "Ended"

  const totalSeconds = Math.floor(milliseconds / 1000)
  const hours = Math.floor(totalSeconds / 3600)
  const minutes = Math.floor((totalSeconds % 3600) / 60)
  const seconds = totalSeconds % 60

  if (hours > 0) {
    return `${hours}h ${minutes}m`
  } else if (minutes > 0) {
    return `${minutes}m ${seconds}s`
  } else {
    return `${seconds}s`
  }
}

export function createLiveActivityParams(item: CountdownItem) {
  return {
    endDate: new Date(item.endDate!).getTime(),
    title: item.title || "Countdown Timer",
    subtitle: item.subtitle || "Time remaining",
    color: "#007AFF",
  }
}
```

### 3. Create Preferences Hook

Create [`app/hooks/useLiveActivityPreferences.ts`](app/hooks/useLiveActivityPreferences.ts):

```typescript
import { useAtom } from "jotai"

import { LiveActivityPreferences, liveActivityPreferencesAtom } from "@/atoms/appAtoms"

export function useLiveActivityPreferences() {
  const [preferences, setPreferences] = useAtom(liveActivityPreferencesAtom)

  const updatePreference = (key: keyof LiveActivityPreferences, value: boolean) => {
    setPreferences((prev) => ({
      ...prev,
      [key]: value,
    }))
  }

  const setEnableLiveActivities = (enabled: boolean) => {
    updatePreference("enableLiveActivities", enabled)
  }

  return {
    preferences: preferences || { enableLiveActivities: true },
    setEnableLiveActivities,
    updatePreference,
  }
}
```

## React Integration

### 1. Create State Atoms

Create [`app/atoms/appAtoms.ts`](app/atoms/appAtoms.ts):

```typescript
import { atom } from "jotai"

import type { CountdownItem } from "../utils/liveActivityHelpers"
import { atomWithMMKV } from "../services/state/storage"

export interface AppState {
  items: CountdownItem[]
  lastUpdated: number
  isLoading: boolean
  hasInitiallyLoaded: boolean
}

export interface LiveActivityPreferences {
  enableLiveActivities: boolean
}

export const appStateAtom = atomWithMMKV<AppState>("app-state", {
  items: [],
  lastUpdated: Date.now(),
  isLoading: false,
  hasInitiallyLoaded: false,
})

export const liveActivityPreferencesAtom = atomWithMMKV<LiveActivityPreferences>(
  "live-activity-preferences",
  {
    enableLiveActivities: true,
  },
)

export const activeLiveActivityItemIdAtom = atom<string | null>(null)
```

### 2. Create Provider Component

Create [`app/providers/AppProvider.tsx`](app/providers/AppProvider.tsx):

```typescript
import { useAtom } from "jotai"
import { type ReactNode, useEffect, useMemo, useRef } from "react"

import { activeLiveActivityItemIdAtom, appStateAtom } from "../atoms/appAtoms"
import { useLiveActivity } from "../hooks/useLiveActivity"
import { useLiveActivityPreferences } from "../hooks/useLiveActivityPreferences"
import type { CountdownItem } from "../utils/liveActivityHelpers"
import {
  calculateTimeUntilEnd,
  createLiveActivityParams,
  findItemEndingSoonest,
  shouldStartLiveActivity,
} from "../utils/liveActivityHelpers"

interface AppProviderProps {
  children: ReactNode
  items: CountdownItem[] // Pass your app's data here
  loading?: boolean
  error?: Error | null
}

export function AppProvider({ children, items, loading = false, error = null }: AppProviderProps) {
  const [, setAppState] = useAtom(appStateAtom)
  const [activeLiveActivityItemId, setActiveLiveActivityItemId] = useAtom(
    activeLiveActivityItemIdAtom,
  )
  const { isRunning, start, update, stop } = useLiveActivity()
  const { preferences } = useLiveActivityPreferences()
  const previousItemRef = useRef<CountdownItem | null>(null)
  const updateCountRef = useRef(0)
  const previousPreferencesRef = useRef(preferences)

  const filteredItems = useMemo(() => {
    const oneDayAgo = Date.now() - 24 * 60 * 60 * 1000
    const filtered = items.filter((item) => {
      if (!item?.endDate) return true

      const endTime = new Date(item.endDate).getTime()
      return endTime > oneDayAgo
    })

    return filtered
  }, [items])

  useEffect(() => {
    setAppState((prev) => ({
      items: loading ? prev.items : filteredItems,
      lastUpdated: loading ? prev.lastUpdated : Date.now(),
      isLoading: loading,
      hasInitiallyLoaded: prev.hasInitiallyLoaded || (!loading && !error),
    }))
  }, [filteredItems, loading, error, setAppState])

  // Handle preference changes
  useEffect(() => {
    if (!preferences.enableLiveActivities && isRunning) {
      console.log("🔴 Live Activity: Stopping due to user preference disabled")
      setActiveLiveActivityItemId(null)
      previousItemRef.current = null
      updateCountRef.current = 0

      const stopLiveActivity = async () => {
        try {
          await stop()
          console.log("✅ Live Activity: Successfully stopped")
        } catch (error) {
          console.warn("Failed to stop live activity:", error)
        }
      }
      stopLiveActivity()
    }

    previousPreferencesRef.current = preferences
  }, [preferences.enableLiveActivities, isRunning, stop, setActiveLiveActivityItemId, preferences])

  // Handle starting/switching live activities
  useEffect(() => {
    if (!preferences.enableLiveActivities) {
      console.log("🔴 Live Activity: Skipping due to user preference disabled")
      return
    }

    const currentItem = findItemEndingSoonest(filteredItems)
    const previousItem = previousItemRef.current

    console.log("🔍 Live Activity: Checking items", {
      currentItemId: currentItem?.id,
      currentItemTitle: currentItem?.title,
      previousItemId: previousItem?.id,
      itemCount: filteredItems.length,
      isRunning,
    })

    if (shouldStartLiveActivity(currentItem, previousItem, preferences)) {
      if (currentItem) {
        if (isRunning && previousItem && previousItem.id !== currentItem.id) {
          console.log("🔄 Live Activity: Switching from previous item", {
            from: previousItem.title,
            to: currentItem.title,
          })
          const stopPrevious = async () => {
            try {
              await stop()
              console.log("✅ Live Activity: Previous item stopped")
            } catch (error) {
              console.warn("Failed to stop previous live activity:", error)
            }
          }
          stopPrevious()
        }

        const params = createLiveActivityParams(currentItem)
        console.log("🟢 Live Activity: Starting for item", {
          itemId: currentItem.id,
          title: currentItem.title,
          endDate: currentItem.endDate,
          params,
        })
        start(params).catch((error) => {
          console.warn("Failed to start live activity:", error)
        })
        setActiveLiveActivityItemId(currentItem.id)
        previousItemRef.current = currentItem
        updateCountRef.current = 0
      }
    } else if (!currentItem && isRunning) {
      console.log("🔴 Live Activity: Stopping - no eligible items found")
      const stopNoItems = async () => {
        try {
          await stop()
          setActiveLiveActivityItemId(null)
          previousItemRef.current = null
          updateCountRef.current = 0
          console.log("✅ Live Activity: Successfully stopped - no items")
        } catch (error) {
          console.warn("Failed to stop live activity:", error)
        }
      }
      stopNoItems()
    }
  }, [filteredItems, preferences, start, stop, isRunning, setActiveLiveActivityItemId])

  // Handle updating live activities
  useEffect(() => {
    if (!isRunning || !preferences.enableLiveActivities) {
      return
    }

    const currentItem = findItemEndingSoonest(filteredItems)
    if (!currentItem) {
      console.log("🔍 Live Activity: No current item found for update")
      return
    }

    if (activeLiveActivityItemId && currentItem.id !== activeLiveActivityItemId) {
      console.log("🔍 Live Activity: Skipping update - different item active", {
        activeItemId: activeLiveActivityItemId,
        currentItemId: currentItem.id,
      })
      return
    }

    const timeRemaining = calculateTimeUntilEnd(currentItem)

    if (timeRemaining <= 0) {
      console.log("🔴 Live Activity: Stopping - item ended", {
        itemId: currentItem.id,
        title: currentItem.title,
      })
      const stopEnded = async () => {
        try {
          await stop()
          setActiveLiveActivityItemId(null)
          previousItemRef.current = null
          updateCountRef.current = 0
          console.log("✅ Live Activity: Successfully stopped - item ended")
        } catch (error) {
          console.warn("Failed to stop live activity:", error)
        }
      }
      stopEnded()
      return
    }

    const params = createLiveActivityParams(currentItem)
    console.log("🔄 Live Activity: Updating data", {
      itemId: currentItem.id,
      title: currentItem.title,
      timeRemaining: Math.floor(timeRemaining / 1000),
      params,
    })
    update(params).catch((error) => {
      console.warn("Failed to update live activity:", error)
    })
  }, [
    filteredItems,
    isRunning,
    preferences.enableLiveActivities,
    activeLiveActivityItemId,
    update,
    stop,
    setActiveLiveActivityItemId,
  ])

  return <>{children}</>
}
```

### 3. Add Settings Integration

Add Live Activity settings to your settings screen:

```typescript
// In your SettingsScreen component
import { Platform } from "react-native"
import { Switch } from "@/components/Toggle/Switch"
import { useLiveActivityPreferences } from "@/hooks/useLiveActivityPreferences"

export const SettingsScreen = () => {
  const { preferences, setEnableLiveActivities } = useLiveActivityPreferences()

  return (
    <Screen>
      {Platform.OS === "ios" && (
        <SettingsGroup title="Live Activities">
          <SettingsCell
            leftIcon={<Icon icon="live-tv" size={24} />}
            text="Enable Live Activities"
            rightContent={
              <Switch
                value={preferences.enableLiveActivities}
                onValueChange={setEnableLiveActivities}
              />
            }
            isLast={true}
          />
        </SettingsGroup>
      )}
    </Screen>
  )
}
```

## Architecture Insights

### 1. Date Handling Best Practices

**Problem**: The original implementation had inconsistent time handling between JavaScript and Swift.

**Solution**: Pass timestamps as milliseconds and handle conversion in Swift:

```typescript
// JavaScript - Pass timestamp in milliseconds
export function createLiveActivityParams(item: CountdownItem) {
  return {
    endDate: new Date(item.endDate!).getTime(), // milliseconds
    title: item.title || "Countdown Timer",
    subtitle: item.subtitle || "Time remaining",
    color: "#007AFF",
  }
}
```

```swift
// Swift - Convert to Date object
let endDate = Date(timeIntervalSince1970: args.endDate / 1000)
```

### 2. Type Safety Considerations

- Use strict TypeScript types for all Live Activity parameters
- Implement proper error handling with custom exception classes
- Validate data at the Swift boundary using `Codable` protocols

### 3. State Management Architecture

The implementation uses a layered approach:

1. **Native Layer**: Swift module handles ActivityKit operations
2. **Bridge Layer**: Expo module provides JavaScript interface
3. **Hook Layer**: React hooks manage state and lifecycle
4. **Provider Layer**: Context providers orchestrate business logic
5. **Component Layer**: UI components consume the functionality

### 4. Cross-Platform Compatibility

- iOS: Full Live Activity implementation
- Android: Stub implementation that gracefully degrades
- Web: Not applicable (Live Activities are iOS-only)

### 5. Performance Optimizations

- Use `useCallback` and `useMemo` to prevent unnecessary re-renders
- Implement proper cleanup in `useEffect` hooks
- Batch state updates where possible
- Use refs for values that don't need to trigger re-renders

## Testing and Troubleshooting

### 1. Build and Test Setup

```bash
# Clean and rebuild
rm -rf ios
npx expo prebuild -p ios
npx expo run:ios

# Test on device (Live Activities don't work in simulator)
npx expo run:ios --device
```

### 2. Common Issues and Solutions

#### Issue: "Live activities are not available on this system"

**Causes:**
- Running on iOS < 16.2
- Live Activities disabled in device settings
- Missing `NSSupportsLiveActivities` in Info.plist

**Solutions:**
- Test on iOS 16.2+ device
- Check Settings > Face ID & Passcode > Live Activities
- Verify `app.json` configuration

#### Issue: "Live activity couldn't be launched"

**Causes:**
- Invalid data passed to ActivityKit
- Missing required permissions
- ActivityKit rate limiting

**Solutions:**
- Validate JSON data structure
- Check device Live Activity permissions
- Implement retry logic with exponential backoff

#### Issue: Live Activity not updating

**Causes:**
- Stale date in the past
- Invalid update data
- Activity already ended

**Solutions:**
- Ensure `staleDate` is in the future
- Validate update parameters
- Check activity state before updating

### 3. Debugging Tips

#### Enable Detailed Logging

```swift
// In LiveActivityControllerModule.swift
log.debug("Started \(activity.id)")
log.debug("Updated activity \(activity.id)")
log.debug("Stopped activity \(activity.id)")
```

#### Monitor Activity State

```typescript
// In your React components
console.log("🔍 Live Activity: Checking watchlist", {
  currentSaleId: currentSale?.id,
  isRunning,
  preferences: preferences.enableLiveActivities,
})
```

#### Test Activity Lifecycle

```typescript
// Simple test component
export function LiveActivityTest() {
  const { isRunning, start, stop } = useLiveActivity()

  const testStart = async () => {
    await start({
      endDate: Date.now() + 60000, // 1 minute from now
      title: "Test Activity",
      subtitle: "Testing Live Activities",
      color: "#C76542",
    })
  }

  return (
    <View>
      <Text>Status: {isRunning ? "Running" : "Stopped"}</Text>
      <Button title="Start Test" onPress={testStart} />
      <Button title="Stop Test" onPress={stop} />
    </View>
  )
}
```

### 4. Platform-Specific Considerations

#### iOS Requirements
- iOS 16.2+ required
- Physical device needed for testing
- Live Activities must be enabled in Settings
- App must be in foreground to start activities

#### Dynamic Island Support
- iPhone 14 Pro and later have Dynamic Island
- Compact and minimal views are required
- Test on both Dynamic Island and non-Dynamic Island devices

#### Activity Limits
- Maximum 8 hours duration
- System may end activities early under memory pressure
- Rate limiting applies to activity creation

## Build and Deployment

### 1. Prebuild Configuration

```bash
# Clean previous builds
rm -rf ios

# Generate iOS project with targets
npx expo prebuild -p ios

# Open in Xcode for final configuration
xed ios
```

### 2. Xcode Configuration

1. **Verify Target Configuration**:
   - Check that widget target is properly configured
   - Ensure deployment target is iOS 16.2+
   - Verify bundle identifiers are correct

2. **Check Entitlements**:
   - Main app should have Live Activities capability
   - Widget extension should have appropriate entitlements

3. **Test Build**:
   - Build and run on physical device
   - Test Live Activity creation and updates
   - Verify Dynamic Island behavior

### 3. EAS Build Configuration

Update your `eas.json`:

```json
{
  "build": {
    "development": {
      "ios": {
        "simulator": false,
        "device": true
      }
    },
    "production": {
      "ios": {
        "simulator": false
      }
    }
  }
}
```

## Reference Materials

### 1. External Resources

- **Christopher2K's Live Activity Example**: [GitHub Repository](https://github.com/Christopher2K/live-activity-example) - Complete working example with countdown timer implementation
- **Original Tutorial**: [How to build a live activity with Expo, SwiftUI and React Native](https://christopher.engineering/en/blog/live-activity-with-react-native/) - Detailed blog post by Christopher N. Katoyi Kaba (contents copied to [`guide.md`](guide.md))
- **Apple Documentation**: [ActivityKit Framework](https://developer.apple.com/documentation/activitykit)
- **Expo Apple Targets**: [@bacons/apple-targets](https://github.com/EvanBacon/expo-apple-targets)

### 2. Key Files in This Implementation

- [`modules/live-activity-controller/ios/LiveActivityControllerModule.swift`](modules/live-activity-controller/ios/LiveActivityControllerModule.swift) - Main Swift implementation
- [`targets/widget/WidgetLiveActivity.swift`](targets/widget/WidgetLiveActivity.swift) - Live Activity UI
- [`targets/widget/CountdownView.swift`](targets/widget/CountdownView.swift) - Custom countdown component
- [`app/providers/WatchlistProvider.tsx`](app/providers/WatchlistProvider.tsx) - Business logic orchestration
- [`app/hooks/useLiveActivity.ts`](app/hooks/useLiveActivity.ts) - React hook interface

### 3. Architecture Documents

- [`docs/live-activity-architecture-refactor.md`](docs/live-activity-architecture-refactor.md) - Lessons learned and architectural improvements

### 4. Best Practices Summary

1. **Always pass dates as timestamps** to avoid timezone issues
2. **Implement proper error handling** at all layers
3. **Use TypeScript strictly** for type safety
4. **Test on physical devices** - simulators don't support Live Activities
5. **Handle activity lifecycle properly** - start, update, and stop appropriately
6. **Implement graceful degradation** for unsupported platforms
7. **Monitor activity state** and handle edge cases
8. **Use proper state management** to avoid memory leaks
9. **Implement user preferences** for Live Activity control
10. **Follow Apple's Human Interface Guidelines** for Live Activity design

## Conclusion

This guide provides a complete implementation of iOS Live Activities in React Native/Expo apps. The architecture is designed to be:

- **Type-safe**: Full TypeScript coverage with proper error handling
- **Cross-platform**: Graceful degradation on non-iOS platforms
- **Maintainable**: Clean separation of concerns and modular design
- **Production-ready**: Proper state management and lifecycle handling
- **User-friendly**: Settings integration and preference management

The implementation uses a countdown timer example that can be easily adapted for any use case - whether it's tracking events, timers, deadlines, or any time-sensitive content in your app.

For questions or issues, refer to the troubleshooting section or consult the reference materials provided.