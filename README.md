# Conceal (Forked & Updated)

[![Build Status](https://travis-ci.org/facebook/conceal.svg?branch=master)](https://travis-ci.org/facebook/conceal)

Conceal provides a set of Java APIs to perform cryptography on Android.  
It was designed to be able to encrypt large files on disk in a fast and
memory efficient manner.

The major target for this project is typical Android devices which run old
Android versions, have low memory and slower processors.

Unlike other libraries, which provide a Smorgasbord of encryption algorithms
and options, Conceal prefers to abstract this choice and use sane defaults.  
Thus Conceal is not a general purpose crypto library, however it aims to provide
useful functionality.

***Upgrading version?*** Check the [Upgrade notes](#upgrade-notes) for key compatibility!

***Already using 1.1.x?*** It's strongly advised to upgrade to ```1.1.3``` as the library size is significatively smaller.

---

## ✨ Fork & Build System Updates

This fork modernizes the build system and adds Android 15 compatibility improvements.

### 🔧 Gradle & Build Configuration
- Added **Gradle wrapper** (`gradlew`, `gradlew.bat`, `gradle-wrapper.properties`)  
  → Ensures consistent builds across environments.  
  → Configured to use **Gradle 9.0-milestone-1**.
- Updated **Android Gradle Plugin** → `8.7.2`.
- Updated SDK settings:
    - `compileSdkVersion` → 35
    - `buildToolsVersion` → `35.0.0`
    - `minSdkVersion` → 23
    - `targetSdkVersion` → 35
- Updated `namespace` → `com.facebook.crypto`.
- Bumped `versionName` → `1.1.3`.
- Set Java compatibility → **Java 17**.
- Configured **Maven publishing** with:
    - New version: `1.1.3-16kb-fixed`
    - Updated POM description (16KB alignment for Android 15).
    - Sources + Javadoc JARs included.

### 📦 Native (NDK) Build Updates
- Set `ndkVersion` → `29.0.14033849`.
- Defined ABI filters: `armeabi-v7a`, `arm64-v8a`, `x86`, `x86_64`.
- Added build arguments:
    - `ANDROID_SUPPORT_FLEXIBLE_PAGE_SIZES=ON`
    - `-DANDROID_PAGE_SIZE=16384`
- Refactored `ndkBuildRelease` task:
    - Uses `android.ndkDirectory`
    - Handles Windows path correctly

### ⚙️ Native Code Configuration (`Application.mk`)
- Updated:
    - `APP_ABI` → `armeabi-v7a arm64-v8a x86 x86_64`
    - `APP_PLATFORM` → `android-23`
    - `APP_STL` → `c++_static`
- Enabled **C++17** (`APP_CPPFLAGS += -std=c++17`).
- Added linker flag:
  ```make
  LOCAL_LDFLAGS += "-Wl,-z,max-page-size=16384"
  ```

### 🛠 Code Changes
- Added missing `<string.h>` includes in:
    - `gcm_util.c`
    - `hmac_util.c`

---

## Quick start

#### Setup options

* **Build using buck**
```bash
buck build :conceal_android
```

* **Use prebuilt binaries**: http://facebook.github.io/conceal/documentation/.

* **Use Maven Central**: Available on maven central under **com.facebook.conceal:conceal:1.1.3@aar** as an AAR package.

#### Running Benchmarks
```bash
./benchmarks/run   benchmarks/src/com/facebook/crypto/benchmarks/CipherReadBenchmark.java   -- -Dsize=102400
```

This script runs vogar with caliper benchmarks.  
You can also specify all the options caliper provides.

###### An aside on KitKat
> Conceal predates Jellybean 4.3. On KitKat, Android changed the provider for
> cryptographic algorithms to OpenSSL. The default Cipher stream however still
> does not perform well. When replaced with our Cipher stream
> (see BetterCipherInputStream), the default implementation is competitive against
> Conceal. On older phones, Conceal is faster than the system provided libraries.

#### Running unit tests
```bash
buck test
```

#### Running integration tests
```bash
./instrumentTest/crypto/run
```

Since Conceal uses native libraries, the only way to run a test on the entire
encryption process is using integration tests.

---

## Usage

#### Encryption
```java
// Creates a new Crypto object with default implementations of a key chain
KeyChain keyChain = new SharedPrefsBackedKeyChain(context, CryptoConfig.KEY_256);
Crypto crypto = AndroidConceal.get().createDefaultCrypto(keyChain);

// Check for whether the crypto functionality is available
if (!crypto.isAvailable()) {
  return;
}

OutputStream fileStream = new BufferedOutputStream(
  new FileOutputStream(file));

// Creates an output stream which encrypts the data as
// it is written to it and writes it out to the file.
OutputStream outputStream = crypto.getCipherOutputStream(
  fileStream,
  Entity.create("entity_id"));

// Write plaintext to it.
outputStream.write(plainText);
outputStream.close();
```

#### Decryption
```java
FileInputStream fileStream = new FileInputStream(file);

// Creates an input stream which decrypts the data as
// it is read from it.
InputStream inputStream = crypto.getCipherInputStream(
  fileStream,
  Entity.create("entity_id"));

int read;
byte[] buffer = new byte[1024];

while ((read = inputStream.read(buffer)) != -1) {
  out.write(buffer, 0, read);
}

inputStream.close();
```

If you don't have a lot of data to encrypt, you could
use the convenience functions:

```java
byte[] cipherText = crypto.encrypt(plainText);

byte[] plainText = crypto.decrypt(cipherText);
```

#### Integrity
```java
OutputStream outputStream = crypto.getMacOutputStream(fileStream, entity);
outputStream.write(plainTextBytes);
outputStream.close();

InputStream inputStream = crypto.getMacInputStream(fileStream, entity);

while((read = inputStream.read(buffer)) != -1) {
  out.write(buffer, 0, read);
}
inputStream.close();
```

---

## Upgrade notes

Starting with v1.1 recommended encryption will use a 256-bit key (instead of 128-bit). This means stronger security.  
You should use this default.

If you need to read from an existing file, you still will need 128-bit encryption. You can use the old way of creating `Crypto` objects as it preserves its 128-bit behavior. Although ideally you should re-encrypt that content with a 256-bit key.

Also there's an improved way of creating `Entity` objects which is platform independent. It's strongly recommended for new encrypted items although you need to stick to the old way for already encrypted content.

#### Existing code still with 128-bit keys and old Entity (deprecated)
```java
KeyChain keyChain = new SharedPrefsBackedKeyChain(context);
Crypto crypto = new Crypto(keyChain, library);
Entity entity = new Entity(someStringId);
```

#### New code using 256-keys and Entity.create
```java
KeyChain keyChain = new SharedPrefsBackedKeyChain(context, CryptoConfig.KEY_256);
AndroidConceal.get().createDefaultCrypto(keyChain);
Entity entity = Entity.create(someStringId);
```

---

## Troubleshooting

#### I'm getting NoSuchFieldError on runtime
```
java.lang.NoSuchFieldError: no field with name='mCtxPtr' signature='J' in class Lcom/facebook/crypto/cipher/NativeGCMCipher;
```

This happens because native code needs to refer to Java fields/methods.  
At the same time tools like ProGuard trim off or rename class members in order to get smaller executables.

To avoid this, configure ProGuard with the rules defined in `proguard_annotations.pro`.  
You can use the file as is, or include its content in your own configuration.

---
