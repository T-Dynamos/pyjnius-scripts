# pyjnius-scripts

## Get App in full screen with status bar and navigation bar transparent
![Screenshot_20230307_100007](https://user-images.githubusercontent.com/68729523/223321096-da8e35db-cffa-4fb2-b75a-ae96bf1e75bc.png)

*Want somthing like this?*

Import this on top

```python
from android.runnable import run_on_ui_thread
from jnius import autoclass
```
Add this function in your code and call it in `build` function.

```python
    @run_on_ui_thread
    def setup_android(self):
        activity = autoclass("org.kivy.android.PythonActivity").mActivity
        WindowCompat = autoclass("androidx.core.view.WindowCompat")
        ViewCompat = autoclass("androidx.core.view.ViewCompat")
        Color = autoclass("android.graphics.Color")
        window = activity.getWindow()
        windowInsetsController = ViewCompat.getWindowInsetsController(
            window.getDecorView()
        )
        windowInsetsController.setAppearanceLightNavigationBars(self.BarMode)
        windowInsetsController.setAppearanceLightStatusBars(self.BarMode)
        WindowCompat.setDecorFitsSystemWindows(window, False)
        window.setNavigationBarColor(Color.TRANSPARENT)
        window.setStatusBarColor(Color.TRANSPARENT)
```
*If you have custom _android.entrypoint_ please replace it with `org.kivy.android.PythonActivity`*

Add this in _buildozer.spec_:
```python
android.apptheme = "@style/AppTheme"
```
Now create a file naming `styles.xml`:
```xml
<resources>
    <style name="AppTheme" parent="android:Theme.Material.Wallpaper.NoTitleBar">
        <item name="android:windowDrawsSystemBarBackgrounds">true</item>
        <item name="android:enforceNavigationBarContrast">false</item>
    </style>
</resources>
```
Now when you will compile your app, it will fail.

Then move styles.xml to `.buildozer/android/platform/build-armeabi-v7a/dists/<>/src/main/res/values/styles.xml`(replace <> with you package.name)

Now try compile it, then your app will be in full screen with transparent status and navigation bar

## Get all installed apps with icon and starting intent

This also requires a permission in android 11+

_buildozer.spec_
```
android.permissions = QUERY_ALL_PACKAGES
```

Don't forget to import this on top

```python
from android.runnable import run_on_ui_thread
from jnius import autoclass
```

set `ANDROID_PATH` in starting of your app class to `/data/data/<>/shared_prefs` (replace <> with package name)

```python
    @run_on_ui_thread
    def get_all_apps(self) -> dict:
        Bitmap = autoclass("android.graphics.Bitmap")
        Canvas = autoclass("android.graphics.Canvas")
        Intent = autoclass("android.content.Intent")
        BitmapConfig = autoclass("android.graphics.Bitmap$Config")
        File = autoclass("java.io.File")
        FileOutputStream = autoclass("java.io.FileOutputStream")
        CompressFormat = autoclass("android.graphics.Bitmap$CompressFormat")
        activity = autoclass("org.kivy.android.PythonActivity").mActivity
        intent = Intent(Intent.ACTION_MAIN)
        PackageManager = activity.getPackageManager()
        intent.addCategory(Intent.CATEGORY_LAUNCHER)

        InstalledApps = PackageManager.queryIntentActivities(intent, 0)
        self.AllInstalledApps = {}

        if os.path.isdir(os.path.join(self.ANDROID_PATH, "icons")) != True:
            os.mkdir(os.path.join(self.ANDROID_PATH, "icons"))

        for app in InstalledApps:
            icon_file = os.path.join(
                self.ANDROID_PATH, "icons/{}.png".format(app.activityInfo.packageName)
            )
            drawable = app.activityInfo.loadIcon(PackageManager)
            bitmap = Bitmap.createBitmap(
                drawable.getIntrinsicWidth(),
                drawable.getIntrinsicHeight(),
                BitmapConfig.ARGB_8888,
            )
            canvas = Canvas(bitmap)
            drawable.setBounds(0, 0, canvas.getWidth(), canvas.getHeight())
            drawable.draw(canvas)
            imageFile = File(
                os.path.join(self.ANDROID_PATH, "icons/"),
                "{}.png".format(app.activityInfo.packageName),
            )

            file_buffer = FileOutputStream(imageFile)
            bitmap.compress(CompressFormat.PNG, 100, file_buffer)
            file_buffer.close()

            self.AllInstalledApps[app.loadLabel(PackageManager)] = [
                app.activityInfo.packageName,
                icon_file,
                app.activityInfo.packageName + "/" + app.activityInfo.name,
            ]

        with open(os.path.join(self.ANDROID_PATH, "apps_installed.json"), "w") as file:
            json.dump(self.AllInstalledApps, file)
            file.close()
        print(self.AllInstalledApps)
        return self.AllInstalledApps

```
This will save icons to icons folder _in shared_prefs_ and print something like:
```
03-07 09:53:34.590  3054  3054 I python  : {'Camera': ['com.android.camera2', '/data/data/com.tdynamos.snowflakes/shared_prefs/icons/com.android.camera2.png', 'com.android.camera2/com.android.camera.CameraLauncher'], 'Contacts': ['com.android.contacts', '/data/data/com.tdynamos.snowflakes/shared_prefs/icons/com.android.contacts.png', 'com.android.contacts/com.android.contacts.activities.PeopleActivity'], 'Clock': ['com.android.deskclock', '/data/data/com.tdynamos.snowflakes/shared_prefs/icons/com.android.deskclock.png', 'com.android.deskclock/com.android.deskclock.DeskClock'], 'Gallery': ['com.android.gallery3d', '/data/data/com.tdynamos.snowflakes/shared_prefs/icons/com.android.gallery3d.png', 'com.android.gallery3d/com.android.gallery3d.app.GalleryActivity'], 
```
Now you can start you app with that last arg using:
```python
    @run_on_ui_thread
    def launch_app(self, root):
        strings = root.split("/")
        Intent = autoclass("android.content.Intent")
        ComponentName = autoclass("android.content.ComponentName")
        intent = Intent(Intent.ACTION_MAIN)
        activity = autoclass("org.kivy.android.PythonActivity").mActivity
        intent.setClassName(strings[0], strings[1])
        intent.setComponent(ComponentName(strings[0], strings[1]))
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        activity.startActivity(intent)
```
where root is arg[-1] like `com.android.camera2/com.android.camera.CameraLauncher`
