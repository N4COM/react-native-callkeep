# AppDelegate Setup Guide for RNCallKeep

The `RCTRootView` error usually means you're using a different React Native setup. Here are solutions for different scenarios:

## Solution 1: Modern React Native (0.68+) with RCTAppDelegate

If you're using the new architecture or modern React Native:

```objective-c
// AppDelegate.h
#import <UIKit/UIKit.h>
#import <RCTAppDelegate.h>
#import "RNCallKeep.h"

@interface AppDelegate : RCTAppDelegate
@end

// AppDelegate.m
#import "AppDelegate.h"

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    // ✅ Call setup BEFORE super (which creates the bridge)
    [RNCallKeep setup:@{
        @"appName": @"Your App Name",
        @"maximumCallGroups": @3,
        @"maximumCallsPerCallGroup": @1,
        @"supportsVideo": @NO,
    }];
    
    // Now call super (which will create bridge and initialize modules)
    return [super application:application didFinishLaunchingWithOptions:launchOptions];
}

@end
```

---

## Solution 2: Standard React Native (0.60-0.67)

If you have a standard AppDelegate setup:

```objective-c
// AppDelegate.m
#import "AppDelegate.h"
#import <React/RCTBundleURLProvider.h>
#import "RNCallKeep.h"

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    // ✅ Call setup FIRST, before creating bridge
    [RNCallKeep setup:@{
        @"appName": @"Your App Name",
        @"maximumCallGroups": @3,
        @"maximumCallsPerCallGroup": @1,
        @"supportsVideo": @NO,
    }];
    
    // Your existing bridge creation code
    RCTBridge *bridge = [[RCTBridge alloc] initWithDelegate:self launchOptions:launchOptions];
    
    // Your existing root view setup (whatever you're using)
    // ... rest of your code
    
    return YES;
}

@end
```

---

## Solution 3: Expo / Expo Bare Workflow

If you're using Expo:

```objective-c
// AppDelegate.m
#import "AppDelegate.h"
#import <ExpoModulesCore/ExpoModulesCore.h>
#import "RNCallKeep.h"

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    // ✅ Call setup FIRST
    [RNCallKeep setup:@{
        @"appName": @"Your App Name",
        @"maximumCallGroups": @3,
        @"maximumCallsPerCallGroup": @1,
        @"supportsVideo": @NO,
    }];
    
    // Your existing Expo setup
    self.moduleName = @"main";
    // ... rest of Expo initialization
    
    return [super application:application didFinishLaunchingWithOptions:launchOptions];
}

@end
```

---

## Solution 4: Minimal Setup (No RCTRootView needed)

**You don't need RCTRootView!** Just call setup early:

```objective-c
// AppDelegate.m
#import "AppDelegate.h"
#import "RNCallKeep.h"

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    // ✅ This is ALL you need! No RCTRootView required.
    [RNCallKeep setup:@{
        @"appName": @"Your App Name",
        @"maximumCallGroups": @3,
        @"maximumCallsPerCallGroup": @1,
        @"supportsVideo": @NO,
    }];
    
    // Your existing app initialization code
    // (whatever you're currently using - doesn't matter)
    
    return YES;
}

@end
```

---

## Solution 5: If You're Using React Native Navigation or Other Routers

```objective-c
// AppDelegate.m
#import "AppDelegate.h"
#import "RNCallKeep.h"

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    // ✅ Call setup FIRST
    [RNCallKeep setup:@{
        @"appName": @"Your App Name",
        @"maximumCallGroups": @3,
        @"maximumCallsPerCallGroup": @1,
        @"supportsVideo": @NO,
    }];
    
    // Your existing navigation setup
    // React Native Navigation, React Navigation, etc.
    // ... your existing code
    
    return YES;
}

@end
```

---

## Important Notes

### 1. Import Statement
Make sure you import RNCallKeep:

```objective-c
#import "RNCallKeep.h"
// OR if using pods:
#import <RNCallKeep/RNCallKeep.h>
```

### 2. Setup Options
The setup dictionary should match what you'd pass from JavaScript:

```objective-c
[RNCallKeep setup:@{
    @"appName": @"Your App Name",                    // Required
    @"maximumCallGroups": @3,                        // Optional
    @"maximumCallsPerCallGroup": @1,                 // Optional
    @"supportsVideo": @NO,                           // Optional
    @"handleType": @"number",                        // Optional: @"number", @"email", @"generic"
    @"includesCallsInRecents": @YES,                 // Optional (iOS 11+)
    @"ringtoneSound": @"ringtone.mp3",               // Optional
    @"imageName": @"callkeep_icon",                  // Optional
}];
```

### 3. Timing is Critical
**Call `[RNCallKeep setup:...]` BEFORE:**
- Creating RCTBridge
- Calling `[super application:didFinishLaunchingWithOptions:]`
- Any other React Native initialization

**The goal:** Initialize RNCallKeep before the bridge initializes modules, so when `init` is called, setup has already run.

---

## Troubleshooting

### Error: "RNCallKeep.h file not found"
**Solution:** Make sure RNCallKeep is properly linked:
```bash
cd ios && pod install
```

### Error: "RCTRootView not found"
**Solution:** You don't need RCTRootView! Just call `[RNCallKeep setup:...]` without it.

### Error: "No visible @interface for 'RNCallKeep' declares the selector 'setup:'"
**Solution:** Make sure you're importing the header:
```objective-c
#import "RNCallKeep.h"
```

### Still getting errors?
**Check:**
1. Is RNCallKeep properly installed? (`pod install`)
2. Is the import path correct?
3. Are you calling setup in the right place (before bridge creation)?

---

## Complete Example (Minimal)

Here's a complete minimal example that works with any React Native setup:

```objective-c
// AppDelegate.h
#import <UIKit/UIKit.h>

@interface AppDelegate : UIResponder <UIApplicationDelegate>
@property (nonatomic, strong) UIWindow *window;
@end

// AppDelegate.m
#import "AppDelegate.h"
#import "RNCallKeep.h"

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    // ✅ Initialize RNCallKeep FIRST
    [RNCallKeep setup:@{
        @"appName": @"My App",
    }];
    
    // Your existing app initialization
    // (bridge, root view, etc. - whatever you're using)
    
    return YES;
}

@end
```

**That's it!** You don't need RCTRootView or any specific React Native setup. Just call `[RNCallKeep setup:...]` early in `didFinishLaunchingWithOptions`.

