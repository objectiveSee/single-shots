### Cool Nerd Icon  
Home · Live coding · Videos · 🇫🇷 / 🇬🇧  

# How to build a live activity with Expo, SwiftUI and React Native  
*Published on February 5th 2025*

## Widgets & native-code integration with React Native

**Tags:** `react-native` `expo` `swiftui` `live-activity` `apple` `ios` `mobile`

As you may know, I built a Pomodoro Timer while live-streaming on Twitch. This was a fun way to learn about modern Expo and share my learnings with my community. On **January 25** I decided to do another round—rewriting the same application with many more features (gotta show the skills!).

Something people kept asking for: widgets, specifically **Live Activities** on iOS.  
This article breaks down how I did it.

I even went a bit viral on Twitter (yeah, still calling it Twitter):

> **We DID IT!**  
> React Native + Expo + Live Activity + Custom module to control the live activity!  
> `pic.twitter.com/2q1LU792gl` — @LLCoolChris_ (Feb 2 2025)

---

## Requirements

I’m assuming you already understand React Native & Expo; I won’t dig into Expo basics. Some Swift helps, but isn’t required.

| Tool | Version (at time of writing) |
|------|------------------------------|
| Xcode | **16.2** (`xcodebuild -version`) |
| CocoaPods | **1.16.2** |
| Node | **20+** |

We’ll start from a blank Expo app. Clone the base tag of the repo:

```bash
# SSH
git clone --depth=1 --branch base git@github.com:Christopher2K/live-activity-example.git

# HTTPS
git clone --depth=1 --branch base https://github.com/Christopher2K/live-activity-example.git

If you’re using HTTPS, do yourself a favor and switch to SSH!

The repo contains an Expo v52 managed-workflow app with Expo Router, TypeScript, and a dev client set-up. Expo will generate an Xcode project for our custom native code—managed workflow goodness without the native headaches.

npm install
npx expo run:ios -d   # choose an iOS simulator

You should see the blank Expo app running.

⸻

What we’re building
	1.	Another iOS target containing the Live Activity (targets are “extensions” like widgets, iMessage, etc.).
	2.	A custom native module so our JS can start/stop the Live Activity.

⸻

Create an iOS target

[Evan Bacon 🥓] made an experimental expo-plugin to create extra targets.

npx create-target

If all goes well you’ll be prompted to add the plugin to app.config.ts:

{
  "plugins": [
    "@bacons/apple-targets"
  ]
}

Run npx create-target again, choose Widget, and follow the prompts. You’ll see a warning about ios.appleTeamId; add it to your config as instructed.

The process creates targets/widget with Swift files. Clean up the ones we don’t need:

rm -rf targets/widget/WidgetControl.swift \
       targets/widget/widgets.swift \
       targets/widget/AppIntent.swift

Edit targets/widget/index.swift and remove references to the deleted files:

@main
struct exportWidgets: WidgetBundle {
  var body: some Widget {
    // Export widgets here
    widget()
    /* widgetControl()  <- remove this line */
    WidgetLiveActivity()
  }
}

Configure the target

Edit targets/widget/expo-target.config.js:

/** @type {import('@bacons/apple-targets/app.plugin').ConfigFunction} */
module.exports = config => ({
  type: "widget",
  icon: "https://github.com/expo.png",
  entitlements: {
    /* Add entitlements */
  },
  frameworks: ["SwiftUI", "ActivityKit"],
});

Enable Live Activities in app.config.ts:

ios: {
  /* existing iOS config */
  infoPlist: {
    NSSupportsLiveActivities: true,
  },
},

Re-generate and build:

rm -rf ios
npx expo prebuild -p ios
npx expo run:ios

The app should build & run smoothly!

⸻

Create a view for the Live Activity

Open the project in Xcode:

xed ios

Define the data model

Create targets/widget/Attributes.swift:

import ActivityKit
import SwiftUI

public struct MyLiveActivityAttributes: ActivityAttributes {
  public struct ContentState: Codable & Hashable {}

  public typealias MyLiveActivityState = ContentState

  public let customString: String
  public let customNumber: Int
}

Build the UI

Replace WidgetLiveActivity.swift with:

import ActivityKit
import WidgetKit
import SwiftUI

struct WidgetLiveActivity: Widget {
  var body: some WidgetConfiguration {
    ActivityConfiguration(for: MyLiveActivityAttributes.self) { context in
      // Lock screen / banner
      VStack {
        Text(context.attributes.customString)
      }
    } dynamicIsland: { context in
      DynamicIsland {
        DynamicIslandExpandedRegion(.leading) { Text("Leading") }
        DynamicIslandExpandedRegion(.trailing) { Text("Trailing") }
        DynamicIslandExpandedRegion(.bottom) { Text("Bottom") }
      } compactLeading: {
        Text("L")
      } compactTrailing: {
        Text("T")
      } minimal: {
        Text("M")
      }
    }
  }
}

#Preview("Lockscreen View", as: .content,
  using: MyLiveActivityAttributes(customString:"Hello World", customNumber:1)
) {
  WidgetLiveActivity()
} contentStates: {
  MyLiveActivityAttributes.MyLiveActivityState()
}


⸻

Create a native module to control the Live Activity

Generate a local Expo module:

npx create-expo-module@latest --local
# → name: activity-controller
# → keep defaults

Cleanup:

rm -rf \
  modules/activity-controller/android \
  modules/activity-controller/src/ActivityControllerModule.web.ts \
  modules/activity-controller/ios/ActivityControllerView.swift \
  modules/activity-controller/src/ActivityControllerView.*

# remove android stanza from expo-module.config.json
touch modules/activity-controller/src/ActivityControllerModule.android.ts

TypeScript interface

modules/activity-controller/src/ActivityController.types.ts

export type LiveActivityParams = { customString: string; customNumber: number };

export type StartLiveActivityFn = (
  params: LiveActivityParams
) => Promise<{ activityId: string }>;

export type StopLiveActivityFn = () => Promise<void>;
export type IsLiveActivityRunningFn = () => boolean;

modules/activity-controller/src/ActivityControllerModule.ts

import { requireNativeModule } from "expo";
import * as types from "./ActivityController.types";

const nativeModule = requireNativeModule("ActivityController");

export const startLiveActivity: types.StartLiveActivityFn = params =>
  nativeModule.startLiveActivity(JSON.stringify(params));

export const stopLiveActivity: types.StopLiveActivityFn = () =>
  nativeModule.stopLiveActivity();

export const isLiveActivityRunning: types.IsLiveActivityRunningFn = () =>
  nativeModule.isLiveActivityRunning();

export const areLiveActivitiesEnabled: boolean =
  nativeModule.areLiveActivitiesEnabled;

Android stub
modules/activity-controller/src/ActivityControllerModule.android.ts

import * as types from "./ActivityController.types";

export const startLiveActivity: types.StartLiveActivityFn = () =>
  Promise.resolve({ activityId: "" });

export const stopLiveActivity: types.StopLiveActivityFn = () => Promise.resolve();

export const isLiveActivityRunning: types.IsLiveActivityRunningFn = () => false;

export const areLiveActivitiesEnabled = false;

Barrel export in modules/activity-controller/index.ts

export * from "./src/ActivityController.types";
export * from "./src/ActivityControllerModule";

Share the attributes struct

Duplicate Attributes.swift for the module:

cp targets/widget/Attributes.swift modules/activity-controller/ios/Attributes.swift

Re-prebuild and reopen Xcode:

rm -rf ios
npx expo prebuild -p ios
xed ios

Native implementation

modules/activity-controller/ios/ActivityControllerModule.swift

import ActivityKit
import SwiftUI
import ExpoModulesCore

// MARK: Exceptions
final class ActivityUnavailableException: GenericException<Void> {
  override var reason: String { "Live activities are not available on this system." }
}
final class ActivityFailedToStartException: GenericException<Void> {
  override var reason: String { "Live activity couldn't be launched." }
}
final class ActivityNotStartedException: GenericException<Void> {
  override var reason: String { "Live activity has not started yet." }
}
final class ActivityAlreadyRunningException: GenericException<Void> {
  override var reason: String { "Live activity is already running." }
}
final class ActivityDataException: GenericException<String> {
  override var reason: String { "The data passed down to the Live Activity is incorrect. \(param)" }
}

// MARK: Types
struct StartActivityArgs: Codable {
  let customString: String
  let customNumber: Int

  static func fromJSON(rawData: String) -> Self? {
    try? JSONDecoder().decode(Self.self, from: Data(rawData.utf8))
  }
}

@available(iOS 16.2, *)
class DefinedActivityWrapper {
  private var activity: Activity<MyLiveActivityAttributes>
  init(_ activity: Activity<MyLiveActivityAttributes>) { self.activity = activity }
  func get() -> Activity<MyLiveActivityAttributes> { activity }
}

struct StartActivityReturnType: Record { @Field var activityId: String }

// MARK: Helpers
func currentActivity() -> Activity<MyLiveActivityAttributes>? {
  guard #available(iOS 16.2, *) else { return nil }
  return Activity<MyLiveActivityAttributes>.activities.first
}

// MARK: Module
public class ActivityControllerModule: Module {
  public func definition() -> ModuleDefinition {
    Name("ActivityController")

    Property("areLiveActivitiesEnabled") {
      if #available(iOS 16.2, *) {
        ActivityAuthorizationInfo().areActivitiesEnabled
      } else { false }
    }

    AsyncFunction("startLiveActivity") { (raw: String) -> StartActivityReturnType in
      guard #available(iOS 16.2, *) else { throw ActivityUnavailableException(()) }
      guard let args = StartActivityArgs.fromJSON(rawData: raw) else { throw ActivityDataException(raw) }
      guard currentActivity() == nil else { throw ActivityAlreadyRunningException(()) }
      guard ActivityAuthorizationInfo().areActivitiesEnabled else { throw ActivityUnavailableException(()) }

      do {
        let attr = MyLiveActivityAttributes(customString: args.customString,
                                            customNumber: args.customNumber)
        let state = MyLiveActivityAttributes.MyLiveActivityState()
        let activity = try Activity.request(attributes: attr,
                                            content: .init(state: state, staleDate: nil))
        return StartActivityReturnType(activityId: activity.id)
      } catch {
        throw ActivityFailedToStartException(())
      }
    }

    AsyncFunction("stopLiveActivity") {
      guard #available(iOS 16.2, *) else { throw ActivityUnavailableException(()) }
      guard let activity = currentActivity() else { throw ActivityNotStartedException(()) }
      Task { await activity.end(nil, dismissalPolicy: .immediate) }
    }

    Function("isLiveActivityRunning") { currentActivity() != nil }
  }
}


⸻

Test it!

Replace app/index.tsx:

import { useState } from "react";
import { StyleSheet, View, Text, TouchableOpacity } from "react-native";
import { startLiveActivity, stopLiveActivity, isLiveActivityRunning } from "#modules/activity-controller";

const styles = StyleSheet.create({
  title: { fontSize: 20, textAlign: "center", fontWeight: "bold", marginTop: 10 },
  container: { flex: 1, justifyContent: "center", alignItems: "center" },
  button: { marginTop: 10, padding: 10, borderRadius: 5, backgroundColor: "#007AFF" },
  buttonText: { fontSize: 16, color: "white" },
});

export default function Index() {
  const [running, setRunning] = useState(isLiveActivityRunning);

  const handleStart = async () => {
    await startLiveActivity({ customString: "Live Activity Testing", customNumber: 123 });
    setRunning(true);
  };
  const handleStop = async () => {
    await stopLiveActivity();
    setRunning(false);
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Live Activity Testing app</Text>

      <TouchableOpacity style={styles.button} onPress={handleStart}>
        <Text style={styles.buttonText}>Start Live Activity</Text>
      </TouchableOpacity>

      <TouchableOpacity style={styles.button} onPress={handleStop}>
        <Text style={styles.buttonText}>Stop Live Activity</Text>
      </TouchableOpacity>

      <Text>{running ? "Running ✅" : "Stopped ⏹️"}</Text>
    </View>
  );
}

Run:

npx expo run:ios

Press Start Live Activity, lock the simulator—you should see it on the lock screen! 🎉

⸻

Wrapping up

This bare-bones example shows how Expo, Expo Modules, expo-apple-targets, and SwiftUI let you ship a feature that React Native doesn’t provide out-of-the-box.

Next time we’ll explore:
	•	bi-directional communication between the widget and TypeScript
	•	code-sharing between the target and the native module

Questions? Reach out on Bluesky or Twitter.

Happy hacking!

All the code is on my GitHub.
© Christopher N. Katoyi Kaba — 2024. All rights reserved.
Subscribe to the RSS feed.

