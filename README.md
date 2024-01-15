# What is uhuruphotos-android?
You may see it as a FOSS gallery manager and also a client for [LibrePhotos](https://github.com/LibrePhotos/librephotos).

See [official repo](https://github.com/savvasdalkitsis/uhuruphotos-android) for details.

# What is this repo for?
This is a **non-official** repo, not related with uhuruphotos-android.

Also note that I **don't** know about android developing (at least for now),
don't take my instructions here too seriously (although I'll try to write it better).

I created this repo for notes and backup about building and translation (localization) thingy.

I've also [forked](https://github.com/clsty/uhuruphotos-android) the official repo when I tested building.
You may find the result apk file [there](https://github.com/clsty/uhuruphotos-android/releases).

# How to translate
The official provides a link to [weblate](https://hosted.weblate.org/engage/uhuruphotos/), where you can contribute translation online.

Also, the localization file is actually at `foundation/strings/api/src/main/res/values*` inside the git repo.

You may download those files packed as zip from weblate once you've done translations, or edit the file directly.

## Sync entries
Ideally, the strings of the original file and the translated file should be automatically synchronized, but in practice, there may be differences.

Here's a stupid way to synchronize them mannually.

Firstly, cp them to a temp dir:
```bash
cd foundation/strings/api/src/main/res/values
t=/tmp/uhurut;mkdir $t
cp ./strings.xml $t/strings-en.xml
cp ../values-zh-rCN/strings.xml $t/
cd $t
```
Now if you run `ls`, there should be `strings-en.xml` `strings.xml`.

Then, open them with vim (or neovim) splitted horizontally:
```bash
vim -O *.xml
```
In every window (you may use `Ctrl+W` to switch window for normal mode), use the command to sort all lines:
```vimcommand
:%sort
```
and then:
```vimcommand
:%s/<string name="\([^"]*\)">[^<]*<\/string>/<string name="\1"><\/string>/g
```
This will delete the content between `<string name="NAME">` and `</string>`.

Now, remove every lines (using dd) **except** for `plurals` and `string` ones (examples below):
```xml
<plurals name="days">
<string name="about"></string>
```
That's done! Now you can tell the differences of the entries easily.
You can also save and exit and use `diff *.xml` to show them even more clearly.


# How to build
For system on the build machine, take Arch Linux as an example.

Get the repo:
```bash
git clone https://github.com/savvasdalkitsis/uhuruphotos-android
```

Install dependencies:
```bash
sudo pacman -S jdk-openjdk gradle
yay -S android-sdk{,-build-tools}
```
Add the following to your `.bashrc` or `.zshrc`:
```bash
export ANDROID_HOME=/usr/lib/android-sdk
export PATH=$PATH:$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools
```

Then, check the file `build.gradle` under the git repo.
Search for `android` block, for example:
```gradle
    android {
        compileSdk 34

        defaultConfig {
            minSdk 24
            targetSdk 34
        }
...
```
In this example, we take the `34` here, and therefore we do:
```bash
sdkmanager --list|grep 34|grep "build-tools\|platforms"
```
Example output:
```plain
build-tools;34.0.0           | 34.0.0      | Android SDK Build-Tools 34                                          
build-tools;34.0.0-rc1       | 34.0.0 rc1  | Android SDK Build-Tools 34-rc1                                      
build-tools;34.0.0-rc2       | 34.0.0 rc2  | Android SDK Build-Tools 34-rc2                                      
build-tools;34.0.0-rc3       | 34.0.0 rc3  | Android SDK Build-Tools 34-rc3                                      
platforms;android-34         | 2           | Android SDK Platform 34                                             
platforms;android-34-ext10   | 1           | Android SDK Platform 34-ext10                                       
platforms;android-34-ext8    | 1           | Android SDK Platform 34-ext8   
```
So we need to install:
```bash
sudo sdkmanager "build-tools;34.0.0-rc3" "platforms;android-34"
```

Now, let's build the project (before that you may want to overwrite the localization file as we mentioned above).

Change dir (cd) to the repo before executing the gradle wrapper inside:
```bash
./gradlew assembleDebug --stacktrace
```
And that's done!
If it went all right, then you'll find your apk file at `app/build/outputs/apk/debug/app-debug.apk`.

However, there could be one or more errors preventing a successful build.
I just encountered that myself (see [issue](https://github.com/savvasdalkitsis/uhuruphotos-android/issues/515)).

Below is how you may deal with those errors.

## Refer to GitHub workflow file
An error I encountered was:
```bash
> Task :app:processDebugGoogleServices FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:processDebugGoogleServices'.
> File google-services.json is missing. The Google Services Plugin cannot function without it.
```
After I refer to the [workflow file](https://github.com/savvasdalkitsis/uhuruphotos-android/blob/main/.github/workflows/main.yml):
```yml
    ...
    - name: Create local properties
      run: touch local.properties
    - name: Copy mock google services
      run: cp mock-google-services.json app/google-services.json
    ...
```
I ran `cp local.defaults.properties local.properties` and `cp mock-google-services.json app/google-services.json` under the git repo.
Then the error disapeared.

## Clean
> NOTE: This appeared to be of no effect, so this operation may be unnecessary.

Use `./gradlew clean` to clean cache.

## Update packages
> NOTE: This appeared to be of no effect, so this operation may be unnecessary.

I noticed that in [Pull Requests](https://github.com/savvasdalkitsis/uhuruphotos-android/pulls) of the official repo, there're some PRs created by a bot named `renovate`, all of which modifying version info in the file `gradle/libs.versions.toml`.

So, I mannually applied the commits into that file.

Also, I searched checked the file `gradle/wrapper/gradle-wrapper.properties` and edit the `gradle-8.4-bin.zip` to `gradle-8.5-bin.zip`。

## Make use of flag `--stacktrace`
I actually didn't use that flag at first; however, after the flag was applied, the real problem behind finally showed up.

```plain
...
Caused by: java.io.FileNotFoundException: /xxx/uhuruphotos-android/feature/feed/domain/implementation/build/intermediates/javac/debug/classes/com/savvasdalkitsis/uhuruphotos/feature/feed/domain/implementation/broadcast/Hilt_CancelFeedDetailsDownloadWorkBroadcastReceiver.class (No such file or directory)
...
```
And then, it seemed that `/xxx/uhuruphotos-android/feature/feed/domain/implementation/build/intermediates/javac/debug/` had no `classes` folder under it but a `compileDebugJavaWithJavac` ;
However, the `/xxx/uhuruphotos-android/feature/feed/domain/implementation/build/intermediates/javac/debug/compileDebugJavaWithJavac/classes` exist, and the file
```plain
/xxx/uhuruphotos-android/feature/feed/domain/implementation/build/intermediates/javac/debug/compileDebugJavaWithJavac/classes/com/savvasdalkitsis/uhuruphotos/feature/feed/domain/implementation/broadcast/Hilt_CancelFeedDetailsDownloadWorkBroadcastReceiver.class
```
also exist.

Apprently, the following command may solve the problem:
```
cd /xxx/uhuruphotos-android/feature/feed/domain/implementation/build/intermediates/javac/debug
cp -r compileDebugJavaWithJavac/classes classes
```
And that's it! As I built again, it worked!

# Extra notes
To setup a LibrePhotos server, if you use docker, it may not create an admin account for you automatically.
If this happened, you may do:
```bash
c(){docker exec -ti backend python manage.py $@ ; }
c createsuperuser
```
Also `c --help` to see a list of available commands.
