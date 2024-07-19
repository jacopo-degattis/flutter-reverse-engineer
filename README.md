# Flutter reverse engineering

## Required tools

- Python3
- [Blutter](https://github.com/worawit/blutter)
- [Apkeditor](https://github.com/REAndroid/APKEditor/releases/)
- Apksigner (available from android sdk build tools)
- [Apktool](https://apktool.org/)
- [Frida](https://frida.re/)

## Procedure

### Setup needed tools

First thing first we're going to create a folder named `flutter_rev` where we are going to put all our tools and files we need to work with.

Now we have to clone the `blutter` repository using the following command:

```shell
$ git clone https://github.com/worawit/blutter
```

Next we have to also downloda the `apkeditor` file going on its github page or simply by using the command:

```shell
$ wget https://github.com/REAndroid/APKEditor/releases/download/V1.3.8/APKEditor-1.3.8.jar -O apkeditor.jar
```

In order to install frida, instead, we can simply use the pip CLI tool entering the following command:

```shell
$ pip3 install frida-tools
```

Last but not least we have to make sure to have the android SDK in path in order to use the `apksigner` utility.

### Choose the target apk file

In order to proceed we need to identify an apk file that we want to use as our target.

Obviously, a very important thing to notice is that the apk has to be built with `flutter`.

Now we have to treat a special case: usually apk files built upon the flutter framework when decompiled with `apktool` have a `lib` folder that contains the `libapp.so` file that is the core of our research.

Sometimes, though, may happen that this file can't be found inside the `base.apk` file and we have to make sure that the apk file has not been splitted in different .apk files.

If this is the case then we need to pull the splitted alone files and check for each of them if they contains the `libapp.so` file.

When we find the one we're interested one we can merge the `base.apk` file and the `splitted_file.apk`.

A shorter solution could be directly merging all splitted files and `base.apk` file in order to avoid checking for each of them.

#### Check for splitted apk files

If we have an application installed, if we want to check for splitted or single files we can use the following command using the `adb` command-line tool:

```shell
$ adb shell pm path <APP_PACKAGE_NAME>
```

Result should be similar to something like

```
package:/data/app/<YOUR_APP_PACKAGE_NAME>/base.apk
package:/data/app/<YOUR_APP_PACKAGE_NAME>/split_config.arm64_v8a.apk
package:/data/app/<YOUR_APP_PACKAGE_NAME>/split_config.en.apk
package:/data/app/<YOUR_APP_PACKAGE_NAME>/split_config.it.apk
package:/data/app/<YOUR_APP_PACKAGE_NAME>/split_config.xxhdpi.apk
```

Now let's imagine, for example that `base.apk` file doesn't contain the `libapp.so` file but `split_config.arm64_v8a.apk` does then we can proceed by merging the two files using the command:

#### Merge splitted apks file together

```shell
$ java -jar ./apkeditor.jar m -i input_folder -o output_merged.apk
```

Where the `input_folder` should be a folder that contains all the apk we want to merge together. (You can easily pull them from the phone using `adb pull <PATH_AFTER_PACKAGE:>`).

#### Sign the merged apk file

Now we have to sign this .apk file to be able to install it on our device and we can use the following command:

```shell
$ apksigner sign --ks <MY_KEYSTORE_FILE> --ks-key-alias <MY_KEYSTORE_ALIAS> --out output.apk input.apk
```

#### Install on device

Now we're able to to install the app on our phone with:

```shell
$ adb install merged.apk
```

### Start the reversing process

At this point we can take the merged apk file and start doing some reverse engineering on it.

We will use the `blutter` utility that we cloned previously.

To use blutter we should write the following command:

```shell
$ python3 blutter.py <FOLDER_CONTAINING_THE_LIBAPP.SO_FILE> <OUTPUT_DIRECTORY>
```

After hitting enter we should wait for some time for it to download the dart sdk and decompile and dump all needed informations from the apk file.

This will provide us with all the methods, classes, variables, etc. names of the dart code.

When the process ends we can find a folder with the name we gave it previously and inside we can see different folder and files:

- **asm**: _folder containing the assembly code that represents all the decompiled dart code_
- **ida_script**: _folder containing ida scripts that are useful if you want to use ida to proceed with reverse engineering_
- **blutter_frida.js**: _key file that we will use to hook inside the app and make the desired changes using the `frida` toolkit_.
- **objs.txt**: _file that contains dumps of dart objects_
- **pp.txt**: _file that contains all dumps of strings, methods, objects, etc._

#### Diving the code

At this point we can dive deeper inside the decompiled code and search for the method or variable who want to hook using the `blutter_frida.js` script.

Here's an example of a decompile asm code:

```dart
class _SplashScreenState extends State<dynamic> {
  _ checkDeviceId(/* No info */) async {
    // previous code...
    // 0x4bf0b4: r0 = post()
    //     0x4bf0b4: bl              #0x37b834  ; [package:http/http.dart] ::post
    // 0x4bf0b8: mov             x1, x0
    // 0x4bf0bc: stur            x1, [fp, #-0x78]
    // 0x4bf0c0: r0 = Await()
    //     0x4bf0c0: bl              #0x288a90  ; AwaitStub
    // 0x4bf0c4: str             x0, [SP]
    // 0x4bf0c8: r0 = body()
    //     0x4bf0c8: bl              #0x41adc0  ; [package:http/src/response.dart] Response::body
    // 0x4bf0cc: r16 = Instance_JsonCodec
    //     0x4bf0cc: ldr             x16, [PP, #0xa40]  ; [pp+0xa40] Obj!JsonCodec@695ad1
    // 0x4bf0d0: stp             x0, x16, [SP]
    // 0x4bf0d4: r4 = const [0, 0x2, 0x2, 0x2, null]
    //     0x4bf0d4: ldr             x4, [PP, #0x160]  ; [pp+0x160] List(5) [0, 0x2, 0x2, 0x2, Null]
    // 0x4bf0d8: r0 = decode()
  }
}
```

This part of the code for example make sure to make an HTTP `post` request and then proceed to decode the response using the `decode()` function.

If we want to edit the response that we get inside the `decode()` method we can proceed using the `blutter_frida.js` script.

All we need to do is, first of all, to replace the address we find at this line

```dart
const fn_addr = <YOUR_ADDRESS_HERE>;
```

with the address of the method we want to hook into.

For example given the class above we can hook into the `decode()` function to see which value is passed to it, so we can replace

```dart
const fn_addr = <YOUR_ADDRESS_HERE>;
```

with

```dart
const fn_addr = 0x4bf0d8;
```

Now we can launch the script using the frida CLI toolkit launching the command:

```shell
$ frida -U -l <LOCATION_OF_BLUTTER_FRIDA.JS> -f <NAME_OF_YOUR_APP_PACKAGE>
```

And what we'll see is the arguments that are passed to the `decode()` function that most of the time will be a JSON formatted string like the following one:

```json
"{\"name\": \"John\", \"username\": \"johnny\"}"
```

#### Overwriting values

Now, the good part. We saw that we are able to read the parameter passed to the `decode()` function but the most cool part is that we can actually change it using the `blutter_frida.js` script.

So in the case of JSON, as it is represented as a string, we have to define a function that helps us to override that address, the function is the following:

```js
function setDartString(ptr, cls) {
  const len = ptr.add(cls.lenOffset).readU32() >> 1;
  var compromisedValue = '{"name":"Paul","username":"Paull}';
  ptr.add(cls.dataOffset).writeUtf8String(compromisedLogin);
}
```

N.B: A really important note to consider is that when we overwrite a value inside the memory we have to always overwrite the string using a string with equal length. In case we have a string with less characters we can use some kind of 'padding' adding white spaces.

Now we can call this function inside the `onEnter` of the script like the following:

```js
let ptr = tptr.sub(1);
setDartString(ptr, cls);
```

And the print again the json that at this point should be modified with proper values:

```js
// Print compromised payload
console.log("Compromised JSON");

var [tptr, cls, values] = getTaggedObjectValue(objPtr);

console.log(
  `${cls.name}@${tptr.toString().slice(2)} =`,
  JSON.stringify(values, null, 2)
);
```

Now executing the script with this command

```shell
$ frida -U -l <LOCATION_OF_BLUTTER_FRIDA.JS> -f <NAME_OF_YOUR_APP_PACKAGE>
```

We should see that our variable has changed.

## Conclusions

As we have seen this is a long process and not always that easy to achieve.
In fact we also have to consider the possibility that the source code can be `obfuscated` and in that case it's really problematic to reverse engineer the code.

If everything goes the right way now we can simply `inject` a permanent script inside the apk and have a fully functional mod.

## Examples

In this guide I used a mod I wrote for the "alexbooks" application but I also
added in this repository another mod for the journal app "internazionale", here's the differences:

- [Alexbooks](./patches/alexbooks_mod.js): mod to provided premium features for free
- [Internazionale](./patches/internazionale_mod.js): mod to read all article for free, even the paid ones. Also block new updates and show popup on app launch

## Sources

- https://www.youtube.com/watch?v=RtKOe8HQy8Q
