# Race Condition Diagnostic Checklist

Compare these factors between your two apps to identify why one crashes and the other doesn't:

## 1. React Native Version
**Check:** `package.json` - `react-native` version
- Different RN versions initialize modules at different times
- Older versions might initialize modules synchronously
- Newer versions might use lazy initialization

**How to check:**
```bash
cd app1 && cat package.json | grep "react-native"
cd app2 && cat package.json | grep "react-native"
```

---

## 2. Bridge Initialization Method
**Check:** `AppDelegate.m` - How `RCTBridge` is created

**Option A: Synchronous (faster init)**
```objective-c
RCTBridge *bridge = [[RCTBridge alloc] initWithDelegate:self launchOptions:launchOptions];
```

**Option B: Asynchronous (slower init)**
```objective-c
RCTBridge *bridge = [[RCTBridge alloc] initWithDelegate:self launchOptions:launchOptions];
// Bridge initializes modules asynchronously
```

**Option C: Lazy initialization**
```objective-c
// Bridge created later, not in didFinishLaunchingWithOptions
```

**Impact:** Synchronous bridge creation = modules initialized earlier = less likely to crash

---

## 3. When RNCallKeep.setup() is Called
**Check:** `AppDelegate.m` - Is `[RNCallKeep setup:...]` called?

**Location A: Before bridge creation (BETTER)**
```objective-c
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:... {
    [RNCallKeep setup:@{...}];  // ← Called FIRST
    RCTBridge *bridge = [[RCTBridge alloc] initWithDelegate:self launchOptions:launchOptions];
}
```

**Location B: After bridge creation (WORSE)**
```objective-c
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:... {
    RCTBridge *bridge = [[RCTBridge alloc] initWithDelegate:self launchOptions:launchOptions];
    [RNCallKeep setup:@{...}];  // ← Called AFTER
}
```

**Location C: Not in AppDelegate (WORST)**
```objective-c
// Setup only called from JavaScript (too late!)
```

**Impact:** Setup in AppDelegate before bridge = provider ready earlier, but still doesn't fix _delayedEvents

---

## 4. JavaScript Bundle Size
**Check:** Bundle size and load time

**How to check:**
```bash
# Check bundle size
ls -lh app1/ios/main.jsbundle
ls -lh app2/ios/main.jsbundle

# Or check if using Metro bundler (dev mode)
# Larger bundles = longer load time = more time for race condition
```

**Impact:** Larger bundle = longer JS load = more time for PushKit to arrive before init()

---

## 5. Other Native Modules Initialization Order
**Check:** Number and type of native modules

**How to check:**
```bash
# Count native modules
grep -r "RCT_EXPORT_MODULE" app1/ios/
grep -r "RCT_EXPORT_MODULE" app2/ios/
```

**Impact:** More modules = longer initialization = more time for race condition
- Modules with `requiresMainQueueSetup = YES` run on main thread
- Heavy modules delay initialization
- Modules that do work in `init` delay other modules

---

## 6. PushKit Handler Location
**Check:** Where is PushKit handler implemented?

**Location A: AppDelegate (BETTER - earlier)**
```objective-c
// In AppDelegate.m
- (void)pushRegistry:...didReceiveIncomingPushWithPayload:... {
    [RNCallKeep reportNewIncomingCall:...];
}
```

**Location B: Separate class (WORSE - might be called later)**
```objective-c
// In separate PushKitManager class
// Might have additional setup/initialization
```

**Impact:** Handler in AppDelegate = called immediately = less time for bridge to initialize

---

## 7. App State When PushKit Arrives
**Check:** Is app actually killed or just backgrounded?

**Test:**
1. Force kill app (swipe away from app switcher)
2. Send PushKit notification
3. Check logs: Does `init` get called before `reportNewIncomingCall`?

**How to check:**
Add logging:
```objective-c
// In RNCallKeep.m init
NSLog(@"[RNCallKeep][init] CALLED - Timestamp: %f", [[NSDate date] timeIntervalSince1970]);

// In reportNewIncomingCall
NSLog(@"[RNCallKeep][reportNewIncomingCall] CALLED - Timestamp: %f", [[NSDate date] timeIntervalSince1970]);
```

**Impact:** 
- App backgrounded = bridge might still be alive = init already called = no crash
- App killed = bridge dead = init not called yet = crash

---

## 8. requiresMainQueueSetup
**Check:** Does RNCallKeep have `requiresMainQueueSetup`?

**Current code:**
```objective-c
+ (BOOL)requiresMainQueueSetup {
    return YES;  // Line 1034-1037
}
```

**Impact:** 
- `YES` = init runs on main thread = might block = affects timing
- `NO` = init runs on background thread = different timing

**Note:** Both apps should have same value if using same RNCallKeep version

---

## 9. JavaScript Import Timing
**Check:** When does JavaScript import RNCallKeep?

**Location A: Early in app (index.js or App.js)**
```javascript
// index.js
import RNCallKeep from 'react-native-callkeep';
RNCallKeep.setup({...});
```

**Location B: Lazy loaded (in a screen/component)**
```javascript
// SomeScreen.js (loaded later)
import RNCallKeep from 'react-native-callkeep';
```

**Impact:** Early import = module initialized earlier = less likely to crash

---

## 10. React Native Architecture
**Check:** Are both apps using same architecture?

**New Architecture (Fabric/TurboModules):**
- Different initialization timing
- Modules initialized differently

**How to check:**
```bash
# Check if using new architecture
grep -r "RCT_NEW_ARCH_ENABLED" app1/
grep -r "RCT_NEW_ARCH_ENABLED" app2/
```

---

## 11. Debug vs Release Build
**Check:** Are you testing in same build mode?

**Debug mode:**
- Slower initialization
- More logging
- Different timing

**Release mode:**
- Faster initialization
- Less overhead
- Different timing

**Impact:** Debug = slower = more time for race condition

---

## 12. Device Performance
**Check:** Are you testing on same device?

**Older devices:**
- Slower initialization
- More time for race condition

**Newer devices:**
- Faster initialization
- Less time for race condition

---

## 13. iOS Version
**Check:** iOS version differences

**How to check:**
- Device Settings → General → About → Software Version
- Or in Xcode console logs

**Impact:** Different iOS versions might handle PushKit/bridge initialization differently

---

## 14. Additional Initialization Code
**Check:** Does one app have more initialization code in AppDelegate?

**Things that delay bridge creation:**
- Heavy setup code before bridge creation
- Network requests
- File I/O
- Other native module setup

**How to check:**
Compare `AppDelegate.m` `didFinishLaunchingWithOptions` between apps

---

## 15. Module Registration Order
**Check:** Is RNCallKeep registered in same position in module list?

**How to check:**
Look for module registration/packages:
- iOS: Check if RNCallKeep is in Podfile
- Check module initialization order in logs

**Impact:** Modules initialized in order - if RNCallKeep is later in list = initialized later

---

## DIAGNOSTIC SCRIPT

Add this to both apps' `AppDelegate.m` to log timing:

```objective-c
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    NSLog(@"[TIMING] didFinishLaunchingWithOptions START: %f", [[NSDate date] timeIntervalSince1970]);
    
    // Your setup code here
    [RNCallKeep setup:@{...}];
    NSLog(@"[TIMING] RNCallKeep setup called: %f", [[NSDate date] timeIntervalSince1970]);
    
    RCTBridge *bridge = [[RCTBridge alloc] initWithDelegate:self launchOptions:launchOptions];
    NSLog(@"[TIMING] RCTBridge created: %f", [[NSDate date] timeIntervalSince1970]);
    
    // ... rest of setup
    
    NSLog(@"[TIMING] didFinishLaunchingWithOptions END: %f", [[NSDate date] timeIntervalSince1970]);
    return YES;
}
```

And in `RNCallKeep.m`:

```objective-c
- (instancetype)init {
    NSLog(@"[TIMING] RNCallKeep init START: %f", [[NSDate date] timeIntervalSince1970]);
    // ... existing code
    NSLog(@"[TIMING] RNCallKeep init END: %f", [[NSDate date] timeIntervalSince1970]);
    return self;
}

+ (void)reportNewIncomingCall:... {
    NSLog(@"[TIMING] reportNewIncomingCall START: %f", [[NSDate date] timeIntervalSince1970]);
    // ... existing code
}
```

Compare the timestamps between the two apps when PushKit arrives!

---

## MOST LIKELY CULPRITS (in order of probability):

1. **JavaScript bundle size** - Larger bundle = longer load = more time for race condition
2. **Bridge initialization timing** - Synchronous vs asynchronous
3. **App state** - One app might be backgrounded (bridge alive) vs killed (bridge dead)
4. **React Native version** - Different initialization behavior
5. **Other native modules** - More modules = longer initialization
6. **Setup location** - Whether setup is called in AppDelegate or JavaScript

---

## QUICK TEST

To quickly identify the difference:

1. Add the timing logs above
2. Force kill both apps
3. Send PushKit notification to both
4. Compare logs:
   - Which app calls `init` first?
   - What's the time difference between `reportNewIncomingCall` and `init`?
   - Is one app's bridge initialized faster?

This will tell you exactly what's different!

