Got it — since you’re **not using AppCompat**, you can still do the splash screen with a plain `Activity` and a theme based on `android:Theme.Material.Light.NoActionBar` (or just `android:Theme.Material.NoActionBar` if you want dark).

Here’s how to do it without AppCompat:

---

### 1. Define Splash Theme in `styles.xml`

```xml
<style name="SplashTheme" parent="android:Theme.Material.Light.NoActionBar">
    <item name="android:windowBackground">@color/black</item>
</style>
```

Make sure you have black defined in `colors.xml`:

```xml
<color name="black">#000000</color>
```

---

### 2. Create Splash Activity extending `android.app.Activity`

```java
public class SplashActivity extends Activity {
    private final Handler handler = new Handler(Looper.getMainLooper());

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // No setContentView needed; windowBackground handles splash

        handler.postDelayed(() -> {
            startActivity(new Intent(SplashActivity.this, MainActivity.class));
            finish();
        }, 1000); // 1 second delay
    }
}
```

---

### 3. Update `AndroidManifest.xml`

```xml
<activity
    android:name=".SplashActivity"
    android:theme="@style/SplashTheme"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>

<activity android:name=".MainActivity" />
```

---

### Explanation

* Since no AppCompat, `SplashActivity` extends plain `Activity`.
* The splash screen background is handled by the theme’s `android:windowBackground`.
* You delay 1 second via a Handler, then launch `MainActivity`.

---

If you want, I can help you tailor this to your project setup!
