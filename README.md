# RemotePreferences

A drop-in solution for inter-app access to `SharedPreferences`.


## Installation

1\. Add the dependency to your `build.gradle` file:

```
repositories {
    jcenter()
}

dependencies {
    compile 'com.crossbowffs.remotepreferences:remotepreferences:0.6'
}
```

2\. Subclass `RemotePreferenceProvider` and implement a 0-argument
constructor which calls the super constructor with an authority
(e.g. `"com.example.app.preferences"`) and a map of
preference files (with modes: after unlock `Boolean.FALSE` and before unlock `Boolean.TRUE`) to expose:

```Java
public class MyPreferenceProvider extends RemotePreferenceProvider {
    public MyPreferenceProvider() {
        super("com.example.app.preferences", map("main_prefs", Boolean.TRUE));
    }
}

private static Map<String, Boolean> map(Object... values) {
        if (values.length%2==1)
            throw new IllegalArgumentException("Map must have even number of values");
        Map<String, Boolean> hashMap = new HashMap<>(values.length/2);
        String key;
        Boolean value;
        for(int i=0; i<values.length; i=i+2) {
            try {
                key = (String) values[i];
            } catch (ClassCastException e) {
                continue;
            }
            try {
                value = (Boolean) values[i+1];
            } catch (ClassCastException e) {
                continue;
            }
            hashMap.put(key, value);
        }
        return hashMap;
    }
```

3\. Add the corresponding entry to `AndroidManifest.xml`, with
`android:authorities` equal to the authority you picked in the
last step, and `android:exported` set to `true`:

```XML
<provider
    android:name=".MyPreferenceProvider"
    android:authorities="com.example.app.preferences"
    android:exported="true"/>
```

4\. You're all set! To access your preferences, create a new
instance of `RemotePreferences` with the same authority and the
name of the preference file:

```Java
SharedPreferences prefs = new RemotePreferences(context, "com.example.app.preferences", "main_prefs");
int value = prefs.getInt("my_int_pref", 0);
```

**WARNING**: **DO NOT** use `RemotePreferences` from within
`IXposedHookZygoteInit.initZygote`, since app providers have not been
initialized at this point. Instead, defer preference loading to
`IXposedHookLoadPackage.handleLoadPackage`.

Note that you should still use `context.getSharedPreferences("main_prefs", MODE_PRIVATE)`
if your code is executing within the app that owns the preferences. Only use
`RemotePreferences` when accessing preferences from the context of another app.

Also note that your preference keys cannot be `null` or `""` (empty string).

Additionally, if you are trying to use any of these preferences before the user unlocks the phone,
use `context = context.createDeviceProtectedStorageContext()`. This is an Android limitation (feature?)

## Security

By default, all preferences have global read/write access. If this is what
you want, then no additional configuration is required. However, chances are
you'll want to prevent 3rd party apps from reading or writing your
preferences. There are two ways to accomplish this:

1. Use the Android permissions system built into `ContentProvider`
2. Override the `checkAccess` method in `RemotePreferenceProvider`

Option 1 is the simplest to implement - just add `android:readPermission`
and/or `android:writePermission` to your preference provider in
`AndroidManifest.xml`. Unfortunately, this does not work very well if
you are hooking apps that you do not control (e.g. Xposed), since you
cannot modify their permissions.

Option 2 requires a bit of code, but is extremely powerful since you
can control exactly which preferences can be accessed. To do this,
override the `checkAccess` method in your preference provider class:

```Java
@Override
protected boolean checkAccess(String prefName, String prefKey, boolean write) {
    // Only allow read access
    if (write) {
        return false;
    }

    // Only allow access to certain preference keys
    if (!"my_pref_key".equals(prefKey)) {
        return false;
    }

    // Only allow access from certain apps
    if (!"com.example.otherapp".equals(getCallingPackage())) {
        return false;
    }

    return true;
}
```

Warning: when checking an operation such as `getAll()` or `clear()`,
`prefKey` will be an empty string. If you are blacklisting certain
keys, make sure to also blacklist the `""` key as well!


## Strict mode

To maintain API compatibility with `SharedPreferences`, by default any errors
encountered while accessing the preference provider will be ignored, resulting
in default values being returned from the getter methods and `apply()` silently
failing (we advise using `commit()` and checking the return value, at least).
This can be caused by bugs in your code, or the user disabling your app/provider
component. To detect and handle this scenario, you may opt-in to *strict mode*
by passing an extra parameter to the `RemotePreferences` constructor:

```Java
SharedPreferences prefs = new RemotePreferences(context, authority, prefName, true);
```

Now, if the preference provider cannot be accessed, a
`RemotePreferenceAccessException` will be thrown. You can handle this by
wrapping your preference accesses in a try-catch block:

```Java
try {
    int value = prefs.getInt("my_int_pref", 0);
    prefs.edit().putInt("my_int_pref", value + 1).apply();
} catch (RemotePreferenceAccessException e) {
    // Handle the error
}
```


## Why would I need this?

This library was developed to simplify Xposed module preference access.
`XSharedPreferences` [has been known to silently fail on some devices](https://github.com/rovo89/XposedBridge/issues/74),
and does not support remote write access or value changed listeners.
Thus, RemotePreferences was born.

Of course, feel free to use this library anywhere you like; it's not
limited to Xposed at all! :-)


## How does it work?

To achieve true inter-process `SharedPreferences` access, all requests
are proxied through a `ContentProvider`. Preference change callbacks are
implemented using `ContentObserver`.

This solution does **not** use `MODE_WORLD_WRITEABLE` (which was
deprecated in Android 4.2) or any other file permission hacks.


## License

Distributed under the [MIT License](http://opensource.org/licenses/MIT).


## Changelog

0.6
- Improved error checking
- Fixed case where strict mode was not applying when editing multiple preferences
- Added more documentation for library internals
- Updated project to modern Android Studio layout

0.5

- Ensure edits are atomic - either all or no edits succeed when committing
- Minor performance improvement when adding/removing multiple keys

0.4

- Fixed `IllegalArgumentException` being thrown instead of `RemotePreferenceAccessException`

0.3

- Values can now be `null` again
- Improved error checking if you are using the ContentProvider interface directly

0.2

- Fixed catastrophic security bug allowing anyone to write to preferences
- Added strict mode to distinguish between "cannot access provider" vs. "key doesn't exist"
- Keys can no longer be `null` or `""`, values can no longer be `null`

0.1

- Initial release.
