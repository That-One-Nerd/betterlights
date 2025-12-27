# Better Light Strips

Contents:
- [About](#better-light-strips)
    - [What this library does](#what-this-library-does)
    - [What this library does NOT do](#what-this-library-doesnt-do-yet)
- [Installation](#installation)
- [Usage](#usage)
    - [Use Case](#use-case)
    - [Configuration](#configuration)
        - [Strip Segments](#strip-segments)
        - [States](#states)
    - [Using States](#using-states)
    - [Custom Patterns](#custom-patterns)

---

This is a WPILib "library" that aims to add customization power to robot light strips.

Writing your own animation code is annoying, and not something you want to spend time doing during a season. If you are, you're mismanaging your time. This library makes things faster.

Work-in-progress. Feel free to use it.

**This library is not particularly easy to use.** While it's powerful, I'll admit it's probably a bit convoluted at first glance.

**This library ALSO is not reliably maintained.** Fork or use at your own risk.

## What this library does.

- Allows you to create your own classes to represent animated patterns, or use some predefined ones.
- Implements a state machine with a priority system.
- Animates patterns on its own clock.
- Dynamically transitions between patterns (you can customize these, too).

## What this library doesn't do YET.

- Support multiple LED strips (planning to add that later).

# Installation

No vendordep yet. Maybe in the future.

- **Recommended Installation**: Clone via `git submodule`. Requires a git repository.
    - `cd ./src/main/java`
    - `git submodule add https://github.com/That-One-Nerd/betterlights`
- Alternatively, you could manually create a folder called `src/main/java/betterlights` and paste the source code from the latest release there.

That's it. The Java package is `betterlights`.

# Usage

## Use Case

Normal WPILib convention is to create a subsystem dedicated to light strips. This is not how this library is intended to be used, and is in fact the problem this library was created to fix.

Rather, all configuration is intended to be done up-front in the `Robot.robotInit()` method or similar. Light segment patterns are managed and animated by the `LightScheduler`, and new states are requested directly, where each subsystem creates its own `LightStatusRequest` object.

You can decide to use this library in the more traditional way, but most subsystem code wrapped around this library would likely be redundant. Feel free to use it in your own way.

## Configuration

The first thing you need to do is configure your light scheduler. I recommend doing this in `Robot.robotInit()` or a dedicated LED subsystem. The `LightScheduler.configure()` method returns a configuration class. You can either save this to a variable, or run methods directly. This class follows the convention of a builder, so all methods return the original instance, for ease of use.

You can set the log level for the light scheduler here. The level you input represents the minimum level to print. 0 represents debug messages, 1 represent information, 2 for warnings, and 3 for errors.

All logs are prefixed with `[LIGHTS]`.

```java
// In Robot...
void robotInit() {
    LightSchedulerConfig lightConfig = LightScheduler.configure();
    lightConfig.withLogLevel(2); // Only show warnings or worse.
}
```

### Strip Segments

Now you should define your light strips. This library is prepared to support multiple *individual* light strips, but at the moment **it does not**. What you **can** do is split a single light strip into multiple segments. Here's a good example: maybe your light strip has one half for the left side of the robot and one half for the right side.

You can define human-readable names for your segments. The first parameter is the name, then the physical port of the LED strip in your RoboRIO. The second and third parameters are the beginning and end indices of the segment, inclusive.

```java
// In Robot.robotInit()...

// Take this example strip. It's set up such that 20 LEDs are on the left
// side of the robot, then there's a 10 LED gap, and then 20 more LEDs are
// on the right. The strip itself is plugged into port 3.
lightConfig.withNamedLightSegment("leftside", 3, 0, 19);   // LEDs #0-#19
lightConfig.withNamedLightSegment("rightside", 3, 30, 49); // Skipping our block of 10.
```

### States

Then we define our states. You can have global states that apply to all segments, as well as specific strip states. A good example would be a decoration strip and a debug strip. Things such as the disabled animation could be a global state that affects both strips, but otherwise the decoration strip and the debug strip would be controlled independently and show different values.

There are a few default patterns in this library. They are represented as classes and follow the "builder" structure. Let's create a few states.

The name of a state can technically be any object, but strings, integers, or enums are preferred.

```java
// In Robot.robotInit()...

// This state applies to the left-side specifically.
lightConfig.withNamedState("left", "default-state", 10, new SolidLightPattern(Color.kBlue));

// This state applies to the right-side specifically. This one is animated.
lightConfig.withNamedState("right", "default-state", 10,
    new BounceLightPattern(Color.kGreen)
        .withMoveSpeed(0.75)
        .withLength(4)
        .withWaveBounce()
        .withFade()
);

// However, there is a state that applies to all segments when active.
// Unless you want to do specific things with specific segments, this
// is the preferred way to define a state. This state has a higher
// priority than the other two.
lightConfig.withStateAll("extra-state", 20, new SolidLightPattern(Color.kRed));
```

States with higher priority are shown when available. However, not all states will always be active (see [Using States](#using-states)).

In addition, you can also set a catch-all state, something that shows when no states are active or an invalid state is active.

```java
// In Robot.robotInit()...
lightConfig.withUnknownBehavior(new RandomLightPattern());
```

That's all for configuration. You don't need to set the configuration variable, as it's passed by-reference.

When you're ready to start, call `LightScheduler.start()` either in your `robotInit()` method or in an LED subsystem. Note that changing the configuration while the scheduler is running will result in undefined behavior (most likely it just won't do anything). If you need to change things, call `LightScheduler.end()`.

```java
// In Robot.robotInit()...
LightScheduler.start();
```

That's it! Check the logs if you have any warnings or errors.

## Using States

To use a state, you must request it. You can request a particular state for one or more segments with `LightScheduler.requestState()`. The state is applied to all segments that defined a state with the name given.

The function returns a `LightStatusRequest` object. It represents the request itself in the scheduler. There are two supported ways to handle a request object. You can either treat it as a toggle, or treat it as a disposable object.

Here is an example. We have a state that toggles ON and OFF every 100 ticks. There are two methods, an `animInit` method and an `animTick()` method.

```java
// This system uses the request as a disposable object.
int tick = 0;
LightStatusRequest request = null;

void animInit() {}
void animTick() {
    tick++;
    if (tick == 100) {
        if (request == null) {
            // Enable the state.
            request = LightScheduler.requestState("state");

            // The state is automatically enabled when requested.
            // Nothing else needs to be done.
        } else {
            // Disable the state.
            request.dispose();
            
            // The object is no longer usable once disposed.
            // It has been removed from the queue.
        }
        tick = 0; // Reset counter and wait another 100 ticks.
    }
}
```

This system either initializes or disposes the request object every 100 ticks, effectively turning the state on and off.

Here's the other supported system, where we directly enable and disable the state.

```java
// This system uses the request object as a togglable object.
int tick = 0;
LightStatusRequest request;

void animInit() {
    request = LightScheduler.requestState("state");

    // Create the object. It's worth noting that the state still
    // starts enabled. If you would like it to start disabled,
    // see below.
}
void animTick() {
    tick++;
    if (tick == 100) {
        if (request.isEnabled()) {
            // Disable the state. This does NOT dispose of this
            // request object, we can keep using it.
            request.disable();
        } else {
            // Re-enable the state.
            request.enable();
        }
        tick = 0; // Reset counter and wait another 100 ticks.
    }
}
```

The second object is my personal preference, but the scheduler is made to handle both.

If you wish for the request to be disposed of when `disable()` is called, set `request.temporary = true`. This will *immediately* dispose of the object when it becomes disabled, and the object will no longer be usable.

This is not recommended, but you can also directly change the state requested by the object. Simply set `request.state = "new-state"`.

## Custom Patterns

Creating your own animations is simple with this library. All you need to do is create a class that extends the `LightPattern` base class. This class implements the standard `LEDPattern` interface, so it *is* backwards compatible, though time will not be automatically incremented in that case.

The only method you need to define is `void applyTo(LEDReader, LEDWriter)` where you set the LEDs directly. To get the current time value, call `getTick()`. It will automatically be incremented. You can also define whether to use the absolute time since the start of the program as your tick value, or the relative time since the pattern has started. Implement the method `boolean useAbsoluteTicks()`. By default, *relative* time is used.

Here is an example pattern that sets LEDs to two different colors.

```java
public class ExamplePattern extends LightPattern {
    @Override
    public void applyTo(LEDReader reader, LEDWriter writer) {
        int tick = getTick();
        int ledCount = reader.getLength();

        if (tick % 2 == 0) {
            // Set each light to RED on EVEN ticks.
            for (int i = 0; i < ledCount; i++) {
                writer.setLED(i, Color.kRed);
            }
        } else {
            // Set each light to GREEN on ODD ticks.
            for (int i = 0; i < ledCount; i++) {
                writer.setLED(i, Color.kGreen);
            }
        }
    }

    // Optional. Defaults to FALSE.
    @Override
    public boolean useAbsoluteTicks() { return true; }
}
```

There are some additional optional methods.
- The `onEnabled()` method is invoked when the pattern is first enabled *before* the first tick occurs.
- The `onDisabled()` method is invoked when the pattern is first disabled.
- The `isComplete()` method is not fully utilized at the moment. It is meant to represent when a transitional pattern has completed.

Some helper methods have been defined for you, namely the `colorLerp(Color, Color, float)` which will interpolate between colors.

An optional fourth gamma parameter is accepted. An ideal gamma value makes interpolated colors more vibrant. On displays (in simulation) a gamma value of 2.2 is recommended, while on LED strips it is typically closer to 1. See [this short video](https://youtu.be/LKnqECcg6Gw) for more information about why it matters.
