# [fit] ReactiveCocoa
# [fit] in
# [fit] Practice

#### Mark Corbyn (@markcorbyn) — Jeames Bone (@jeamesbone)
#### YOW Connected

^ Last ~6 months team adopted ReactiveCocoa

^ Intro RAC as a framework for modelling problems as streams of values over time

---

# :gift:

^ We'd like to share with you 3 interesting examples of interesting ways we've been able to apply RAC in our app

---

# A Quick Introduction

- Signals represent streams of values over time

```objc
[textField.rac_textSignal subsribeNext:^(NSString *text) {
  NSLog(@"hey, the text changed! %@", text);
}];
// ["r", "re", "rea", "reac", "react"]
```

^ This talk is not meant to teach you the basics, but we'll give you a quick refresher on how it works.

---

- You can use operators to transform them

```objc
RACSignal *numberSignal = [textSignal map:^(NSString *text) {
  return @(text.integerValue);
}];
// [1, 2, 3, 4, 5, 6]

RACSignal *evenNumberSignal = [numberSignal filter:^(NSNumber *number) {
  return (number / 2 == 0)
}]
// [2, 4, 6]
```

---

- And to combine them

```objc
RACSignal *firstSignal; // [1, 3, 5, 7]
RACSignal *secondSignal; // [2, 4, 6, 8]

RACSignal *mergedSignal = [firstSignal merge:secondSignal];
// [1, 2, 3, 4, 5, 6, 7, 8]

RACSignal *concatSignal = [firstSignal concat:secondSignal];
// [1, 3, 5, 7, 2, 4, 6, 8]

RACSignal *combineLatest = [firstSignal combineLatestWith:secondSignal];
// [(1, 2), (3, 4), (5, 6), (7, 8)]
```

---

## Why Signals?

- They’ll solve all your problems
- Magic and stuff
- $$$

---

# How far can you go?
^ ReactiveCocoa is a very powerful tool for modeling systems in your app, and we'd like to show you some more exciting examples.

^ All examples will be taken from real code my team's worked on

^ Maybe you didn't realise you could use ReactiveCocoa for this?

---

### 1. A reactive container view controller
### 2. Reactive Notifications
### 3. Functional data processing

---

# Something simple

### Authentication

```objc
/// Stores authentication credentials and tells us if we're authenticated.
@protocol AuthStore <NSObject>

@property (nonatomic, readonly, getter=isAuthenticated) BOOL authenticated;

- (void)storeAccessCredentials:(AccessCredentials *)accessCredentials;
- (nullable AccessCredentials *)retrieveAccessCredentials;
- (void)removeAccessCredentials;

@end
```

---

### Observe authentication status changes

```objc
// Boolean signal sends the current value on subscription, and a new value
// whenever we log in or out.
RACSignal *authenticatedSignal = RACObserve(authStore, authenticated);
```

^ RACObserve can be used to create a signal that fires every time a property changes

---

### Select the right view controller

```objc
RACSignal *authenticatedSignal = RACObserve(authStore, authenticated);

// Pick a view controller
RACSignal *viewControllerSignal = [authenticatedSignal map:^(NSNumber *isAuthenticated) {
  if (isAuthenticated.boolValue) {
    return [DashboardViewController new];
  } else {
    return [LoginViewController new];
  }
}];
```

---

### Displaying it on screen

#### How do we get values out of a signal? We have to subscribe, like a callback.

```objc
- (void)viewDidLoad {
  [super viewDidLoad];

  // Whenever we get a new view controller, PUSH IT
  [[viewControllerSignal deliverOnMainThread]
    subscribeNext:^(UIViewController *viewController) {
      [self showViewController:viewController sender:self];
    }];
}
```

^ Problems with this approach
^ If we are in a navigation controller this will keep pushing a new view controller onto the stack.

---

### Ok, so maybe that’s pushing it

#### Let’s make a custom view controller container!

^ We'll make a custom container that will display the latest view controller sent on a signal.

---

### Switching to the latest view controller

```objc
/// Manages a signal of view controllers, always displaying the most recent one
/// as its content.
@interface SwitchingController : UIViewController

- (instanctype)initWithViewControllers:(RACSignal *)viewControllers;

@property (nonatomic, readonly) UIViewController *currentViewController;

@end
```

---

```objc
- (void)viewDidLoad {
  [super viewDidLoad];

  [[self.viewControllers deliverOnMainThread] subscribeNext:^(UIViewController *viewController) {
    [self transitionFrom:self.currentViewController to:viewController];
  }];
}

- (void)transitionFrom:(UIViewController *)fromViewController to:(UIViewController *)toViewController {
  if (!fromViewController) {
    [self addInitialViewController:toViewController];
    return;
  }

  [self addChildViewController:nextViewController];
  nextViewController.view.frame = self.view.bounds;
  [self.view addSubview:nextViewController.view];
  [previousViewController willMoveToParentViewController:nil];

  [self transitionFromViewController:fromViewController
                    toViewController:toViewController
                            duration:0.5
                             options:UIViewAnimationOptionTransitionCrossDissolve
                          animations:nil
                          completion:^(BOOL finished) {
                            [toViewController didMoveToParentViewController:self];
                            [fromViewController removeFromParentViewController];
                            self.currentViewController = toViewController;
                          }];
}
```

^ Most of this is just container view controller boilerplate

---

### Tying a 🎀 on it

---

### Tying a 🎀 on it

- What happens if the view controller changes rapidly (faster than the animation?)

---

# v1: Simple Throttling

```objc
- (void)viewDidLoad {
  [super viewDidLoad];

  [[[self.viewControllers
    throttle:0.5]
    deliverOnMainThread]
    subscribeNext:^(UIViewController *viewController) {
      [self transitionFrom:self.currentViewController to:viewController];
    }];
}
```

^ First pass: use -throttle: to prevent multiple view controllers during an animation

---

# v2: Only throttle while animating

## 1) keep track of whether we're animating

```objc
@interface SwitchingController ()
@property (getter=isAnimating) BOOL animating;
@end
```

---

## 1) keep track of whether we're animating

```objc
@interface SwitchingController ()
@property (nonatomic, getter=isAnimating) BOOL animating;
@end

// ...

- (void)transitionFrom:(UIViewController *)fromViewController to:(UIViewController *)toViewController {
  // code

  self.animating = YES;

  [self transitionFromViewController:fromViewController
                    toViewController:toViewController
                            duration:0.5
                             options:UIViewAnimationOptionTransitionCrossDissolve
                          animations:nil
                          completion:^(BOOL finished) {
                            // more code
                            self.animating = NO;
                          }];
}
```

---

## 2) throttle with a huge timeout while animating

```objc
  NSDate *forever = [NSDate distantFuture];

  [[[self.viewControllers
    throttle:forever valuesPassingTest:^(id _) {
      return self.isAnimating;
    }]
    deliverOnMainThread]
    subscribeNext:^(UIViewController *viewController) {
      [self transitionFrom:self.currentViewController to:viewController];
    }];
```

---

### 1. ~~A reactive container view controller~~
### 2. Reactive Notifications
### 3. Functional data processing

---

# Reactive Notifications

---

## Push notifications (old school style)

```objc
  - (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo {
    // Do something horrible™ in here
  }
```

---

## A better option:

```objc
  typedef NS_ENUM(NSUInteger, NotificationType) {
    NotificationTypeA,
    NotificationTypeB,
    NotificationTypeC,
  };

  @protocol NotificationProvider

  - (RACSignal *)notificationSignalForNotificationType:(NotificationType)type;

  @end
```

---

## We want this:

```objc
  @property id<NotificationProvider> notificationProvider;

  - (void)viewDidLoad {
    [[[self.notificationProvider
      notificationSignalForNotificationType:NotificationTypeA]
      map:^(Notification *notification) {
        return notification.model;
      }]
      subscribeNext^(AwesomeModel *model) {
        [self updateInterfaceWithModel:model];
      }]
  }
```

---

![](http://vignette1.wikia.nocookie.net/katyperry/images/4/45/Katyperry_hotncold.jpg/revision/latest?cb=20140211091238)

---

## Hot Signals
- Events happen regardless of any observers.
- Stream of *events* happening in the world.
- e.g. button presses, notifications

---

## Cold Signals
- Subscribing starts the stream of events.
- Stream of *results* caused by some side effects.
- e.g. network calls, database transactions

---

## A new friend!

```objc
  - (RACSignal *)rac_signalForSelector:(SEL)selector;
  - (RACSignal *)rac_signalForSelector:(SEL)selector fromProtocol:(Protocol *)protocol;
```

Lifts a selector into the world of signals.

The returned signal will fire an event every time the method is called.

^ This is a hot signal
^ Sends a tuple whenever the method is called
^ A tuple is like an array with a fixed size. Objective-C does not have an in-built tuple class so RAC provides one. In this case it will have 2 values: the application and the notification info

---

## Lets do it

```objc
  - (RACSignal *)notificationSignalForNotificationType:(NotificationType)type {
    return [[[self
      rac_signalForSelector:@selector(application:didReceiveRemoteNotification:)]
      map:^(RACTuple *arguments) {
        return arguments.second;
      }]
      flattenMap:^(NSDictionary *userInfo) {
        // Parse our user info dictionary into a model object
        return [self parseNotification:userInfo];
      }]
      filter:^(Notification *notification) {
        notification.type = type;
      }];
  }
```

---

## A wild local notification appears!

---

### Now we have 2 sources of notifications
- We don't want to duplicate our current notification handling.
- Local and remote notifications should have the same effects.

---

```objc
  RACSignal *remoteNotificationInfo = [[self
    rac_signalForSelector:@selector(application:didReceiveRemoteNotification:)]
    map:^(RACTuple *arguments) {
      return arguments.second
    }];

  RACSignal *localNotificationInfo = [[self
    rac_signalForSelector:@selector(application:didReceiveLocalNotification:)]
    map:^(RACTuple *arguments) {
      UILocalNotification *notification = arguments.second;
      return notification.userInfo;
    }];

  self.notificationSignal =
      [[remoteNotificationInfo merge:localNotificationInfo]
      flattenMap:^(NSDictionary *userInfo) {
        // Parse our user info dictionary into a model object
        return [self parseNotification:userInfo];
      }];
```

---

## Problem?
- We only get notifications sent *after* we subscribe.
- We can't easily update app state or UI that is created after the notification is sent.

---

## Solution?
## `replayLast`

- Whenever you subscribe to the signal, it will immediately send you the most recent value from the stream.

---

```objc
  self.notificationSignal = [[[[self
      rac_signalForSelector:@selector(application:didReceiveRemoteNotification:)]
      map:^(RACTuple *arguments) {
        return arguments.second;
      }]
      flattenMap:^(NSDictionary *userInfo) {
        // Parse our user info dictionary into a model object
        return [self parseNotification:userInfo];
      }]
      replayLast];



  - (RACSignal *)notificationSignalForNotificationType:(NotificationType)type {
    return [self.notificationSignal
      filter:^(Notification *notification) {
        return notification.type = type;
      }]
  }
```

---

### 1. ~~A reactive container view controller~~
### 2. ~~Reactive Notifications~~
### 3. Functional Data Processing


---

## Describing an algorithm with functions

### Example: Finding the best voucher to cover a purchase

- If there are any vouchers with higher value than the purchase, use the lowest valued of those
- Otherwise, use the highest valued voucher available

---

# Imperative approach

---

```objc
- (id<Voucher>)voucherForPurchaseAmount:(NSDecimalNumber *)purchaseAmount {

  NSArray *vouchers = [[self voucherLibrary] vouchers];

  NSArray *sortedVouchers = 
  	[vouchers sortedArrayUsingComparator:^NSComparisonResult(id<Voucher> voucher1, id<Voucher> voucher2) {
		return [voucher1 compare:voucher2];
  	}];

  id<Voucher> bestVoucher = nil;
  for (id<Voucher> voucher in sortedVouchers) {
    NSDecimalNumber *voucherAmount = voucher.amount;
    if (!voucherAmount) continue;

    if ([voucherAmount isLessThan:purchaseAmount]) {
      bestVoucher = voucher;
    } else if ([voucherAmount isGreaterThanOrEqualTo:purchaseAmount]) {
      bestVoucher = voucher;
      break;
    }
  }

  return bestVoucher;
}
```

---

```objc
- (NSComparisonResult)compare:(id<Voucher>)anotherVoucher

	NSDecimalNumber *voucher1Amount = self.amount;
	NSDecimalNumber *voucher2Amount = anotherVoucher.amount;

	if (voucher1Amount == nil && voucher2Amount == nil) {
	  return NSOrderedSame;
	} else if (voucher1Amount == nil) {
	  return NSOrderedDescending;
	} else if (voucher2Amount == nil) {
	  return NSOrderedAscending;
	} else {
	  return [voucher1Amount compare:voucher2Amount];
	}
}
```

---

# Separate the nil filter?

```objc
NSMutableArray *vouchersWithValue = [NSMutableArray array];
for (id<Voucher> voucher in sortedVouchers) {
	if (voucher.amount) {
	  [vouchersWithValue addObject:voucher];
	}
}

id<Voucher> bestVoucher = nil;
for (id<Voucher> voucher in vouchersWithValue) {
  NSDecimalNumber *voucherAmount = voucher.amount;
  if ([voucherAmount isLessThen:purchaseAmount]) {
    bestVoucher = voucher;
  } else if ([voucherAmount isGreaterThanOrEqualTo:purchaseAmount]) {
    bestVoucher = voucher;
    break;
  }
}
```

---

# Functional

---

```objc
- (id<Voucher>)voucherForPurchaseAmount:(NSDecimalNumber *)purchaseAmount {
[[[[[[[[self voucherLibrary]
    vouchers]
    sortedArrayUsingComparator:^NSComparisonResult(id<Voucher> voucher1, 
    											   id<Voucher> voucher2) {
      return [voucher1 compare:voucher2];
    }]
    rac_sequence]
    filter:^BOOL(id<Voucher> voucher) {
      return voucher.amount != nil;
    }]
    yow_takeUptoBlock:^BOOL(id<Voucher> voucher) {
      return [voucher.amount isGreaterThanOrEqualTo:purchaseAmount];
    }]
    array]
    lastObject];
```

---

### 1. ~~A reactive container view controller~~
### 2. ~~Reactive Notifications~~
### 3. ~~Functional Data Processing~~

---

## Resources

- [reactivecocoa.io](http://reactivecocoa.io)
- [ReactiveCocoa on GitHub](https://github.com/ReactiveCocoa/ReactiveCocoa)

- [ReactiveCocoa - The Definitive Introduction (Ray Wenderlich)](http://www.raywenderlich.com/62699/reactivecocoa-tutorial-pt1)
- [A First Look at ReactiveCocoa 3.0 (Scott Logic)](http://blog.scottlogic.com/2015/04/24/first-look-reactive-cocoa-3.html)

^ Philosophy page on reactivecocoa.io is awesome for explaining why we want to use FRP
^ Very well documented source code
^ These tutorials are a great introduction if you haven't seen this stuff before
^ For a swift introduction, the Scott Logic article is great (especially if you've used the objective-c framework)

---

## Next Steps

- Try it!
- Don't be scared to go too far

^ Try using reactivecocoa in your own projects
^ Don't be scared to experiment, we use it very heavily in our current project and we've found that it is
  incredibly flexible and can be applied to almost any problem.
