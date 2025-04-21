# Android setup
ANDROID_NDK_ROOT ?=
# macOS code signing setup
CODESIGN ?= codesign
MACOS_CODESIGN_SIGNATURE ?=
# Download GitHub Action releases
GITHUB_CLI_BIN ?= gh
GITHUB_REPO ?= gilzoide/unity-sqlite-net
RUN_ID ?=

# 암호화 라이브러리 설정
CFLAGS += -DSQLITE_HAS_CODEC -DSQLCIPHER_CRYPTO_OPENSSL -Isqlite-amalgamation #-Iopenssl

# 기본값 설정 (Android가 아닌 경우에만 사용)
ifneq ($(findstring android,$(MAKECMDGOALS)),android)
  LDFLAGS += -lcrypto
endif

ifeq ($(DEBUG),1)
	CFLAGS += -O0 -g -DDEBUG
else
	CFLAGS += -O3 -DNDEBUG
endif

# BUILD_DIRS에서 iOS 시뮬레이터 제거
BUILD_DIRS = \
    lib/windows/x86_64 lib/windows/x86 lib/windows/arm64 \
    lib/linux/x86_64 lib/linux/x86 \
    lib/macos \
    lib/android/arm64 lib/android/arm32 lib/android/x86 lib/android/x86_64 \
    lib/ios/arm64  # iOS 디바이스만 지원

SQLITE_SRC = sqlite-amalgamation/sqlite3.c
SQLITE_NET_SRC = $(wildcard sqlite-net~/src/*.cs) sqlite-net~/LICENSE.txt
SQLITE_NET_DEST = ../Runtime/sqlite-net
SQLITE_NET_SED_SCRIPT = tools~/fix-library-path.sed

$(BUILD_DIRS):
	mkdir -p $@

.PRECIOUS: lib/%/sqlite3.o~
lib/%/sqlite3.o~: $(SQLITE_SRC) | lib/%
	$(CC) -c -o $@ $< $(CFLAGS)

# Windows
lib/windows/%/gilzoide-sqlite-net.dll: LINKFLAGS += -shared
lib/windows/%/gilzoide-sqlite-net.dll: lib/windows/%/sqlite3.o~ | lib/windows/%
	$(CC) -o $@ $^ $(LDFLAGS) $(LINKFLAGS)

# Linux
lib/linux/%/libgilzoide-sqlite-net.so: CFLAGS += -fPIC
lib/linux/%/libgilzoide-sqlite-net.so: LINKFLAGS += -shared -lm
lib/linux/%/libgilzoide-sqlite-net.so: lib/linux/%/sqlite3.o~ | lib/linux/%
	$(CC) -o $@ $^ $(LINKFLAGS)

# macOS
lib/macos/libgilzoide-sqlite-net.dylib: CFLAGS += -arch arm64 -arch x86_64 -isysroot $(shell xcrun --show-sdk-path --sdk macosx)
lib/macos/libgilzoide-sqlite-net.dylib: LINKFLAGS += -shared -arch arm64 -arch x86_64
lib/macos/libgilzoide-sqlite-net.dylib: lib/macos/sqlite3.o~ | lib/macos
	$(CC) -o $@ $^ $(LINKFLAGS)
ifdef MACOS_CODESIGN_SIGNATURE
	$(CODESIGN) -s "$(MACOS_CODESIGN_SIGNATURE)" $@
endif

# Android
check-ndk-root:
ifndef ANDROID_NDK_ROOT
	$(error ANDROID_NDK_ROOT must be set for Android builds!)
endif

# arm64
lib/android/arm64/libgilzoide-sqlite-net.so: \
  CFLAGS += -I/output/openssl/arm64-v8a/include -fPIC
lib/android/arm64/libgilzoide-sqlite-net.so: \
  LDFLAGS += /output/openssl/arm64-v8a/lib/libcrypto.a

# arm32
lib/android/arm32/libgilzoide-sqlite-net.so: \
  CFLAGS += -I/output/openssl/armeabi-v7a/include -fPIC
lib/android/arm32/libgilzoide-sqlite-net.so: \
  LDFLAGS += /output/openssl/armeabi-v7a/lib/libcrypto.a

# 명시적으로 동적 라이브러리 생성 플래그 추가
# Android 특정 라이브러리 추가 (-llog)
lib/android/%/libgilzoide-sqlite-net.so: LINKFLAGS += -shared -lm -llog -Wl,-z,max-page-size=16384 -Wl,--no-undefined -Wl,--gc-sections

lib/android/%/libgilzoide-sqlite-net.so: lib/android/%/sqlite3.o~ check-ndk-root | lib/android/%
	$(CC) -o $@ $< $(LDFLAGS) $(LINKFLAGS)

lib/android/arm64/libgilzoide-sqlite-net.so: CC = $(wildcard $(ANDROID_NDK_ROOT)/toolchains/llvm/prebuilt/*/bin/aarch64-linux-android21-clang)
lib/android/arm32/libgilzoide-sqlite-net.so: CC = $(wildcard $(ANDROID_NDK_ROOT)/toolchains/llvm/prebuilt/*/bin/armv7a-linux-androideabi21-clang)
lib/android/x86_64/libgilzoide-sqlite-net.so: CC = $(wildcard $(ANDROID_NDK_ROOT)/toolchains/llvm/prebuilt/*/bin/x86_64-linux-android21-clang)
lib/android/x86/libgilzoide-sqlite-net.so: CC = $(wildcard $(ANDROID_NDK_ROOT)/toolchains/llvm/prebuilt/*/bin/i686-linux-android21-clang)

# iOS 설정 (정적 라이브러리 사용)
# iOS는 동적 라이브러리(.so, .dylib)가 아닌 정적 라이브러리(.a)를 사용해야 함
lib/ios/%/libgilzoide-sqlite-net.a: lib/ios/%/sqlite3.o~ | lib/ios/%
	$(AR) rcs $@ $^

# iOS 디바이스 (arm64)
lib/ios/arm64/libgilzoide-sqlite-net.a: CFLAGS += -arch arm64 -isysroot $(shell xcrun --sdk iphoneos --show-sdk-path) -fPIC -fembed-bitcode
lib/ios/arm64/libgilzoide-sqlite-net.a: AR = $(shell xcrun -f ar)
lib/ios/arm64/libgilzoide-sqlite-net.a: CC = $(shell xcrun -f clang)

# iOS 시뮬레이터 관련 부분 제거

# iOS 통합 정적 라이브러리 - 디바이스만 포함하도록 수정
lib/ios/libgilzoide-sqlite-net.a: lib/ios/arm64/libgilzoide-sqlite-net.a
	mkdir -p $(dir $@)
	cp $< $@
ifdef IOS_CODESIGN_SIGNATURE
	$(CODESIGN) -s "$(IOS_CODESIGN_SIGNATURE)" $@
endif

# OpenSSL 링킹을 위한 iOS 특별 설정 - 디바이스만 설정
lib/ios/arm64/libgilzoide-sqlite-net.a: \
  CFLAGS += -I$(CURDIR)/openssl/build/ios-arm64/include
lib/ios/arm64/libgilzoide-sqlite-net.a: \
  LDFLAGS += $(CURDIR)/openssl/build/ios-arm64/lib/libcrypto.a

# iOS 시뮬레이터 설정 제거

# Source
$(SQLITE_NET_DEST)/%.cs: sqlite-net~/src/%.cs $(SQLITE_NET_SED_SCRIPT)
	cat $< | sed -E -f $(SQLITE_NET_SED_SCRIPT) > $@

$(SQLITE_NET_DEST)/License.txt: sqlite-net~/License.txt
	cp $< $@

# Targets
windows-x86_64: lib/windows/x86_64/gilzoide-sqlite-net.dll
windows-x86: lib/windows/x86/gilzoide-sqlite-net.dll
windows-arm64: lib/windows/arm64/gilzoide-sqlite-net.dll

windows-mingw-x86_64: CC = x86_64-w64-mingw32-gcc
windows-mingw-x86_64: LINKFLAGS += -static-libgcc
windows-mingw-x86_64: lib/windows/x86_64/gilzoide-sqlite-net.dll

windows-mingw-x86: CC = i686-w64-mingw32-gcc
windows-mingw-x86: LINKFLAGS += -static-libgcc
windows-mingw-x86: lib/windows/x86/gilzoide-sqlite-net.dll

windows-mingw-arm64: CC = aarch64-w64-mingw32-gcc
windows-mingw-arm64: LINKFLAGS += -static-libgcc
windows-mingw-arm64: lib/windows/arm64/gilzoide-sqlite-net.dll

linux-x86_64: lib/linux/x86_64/libgilzoide-sqlite-net.so
linux-x86: lib/linux/x86/libgilzoide-sqlite-net.so

# 모든 Linux 빌드를 위한 타겟 추가
all-linux: linux-x86_64 linux-x86

macos-universal: lib/macos/libgilzoide-sqlite-net.dylib

android-arm64: lib/android/arm64/libgilzoide-sqlite-net.so
android-arm32: lib/android/arm32/libgilzoide-sqlite-net.so
android-x86_64: lib/android/x86_64/libgilzoide-sqlite-net.so
android-x86: lib/android/x86/libgilzoide-sqlite-net.so

ios-arm64: lib/ios/arm64/libgilzoide-sqlite-net.a
ios-universal: lib/ios/libgilzoide-sqlite-net.a

source: $(SQLITE_NET_DEST)/License.txt $(SQLITE_NET_DEST)/AssemblyInfo.cs $(SQLITE_NET_DEST)/SQLite.cs $(SQLITE_NET_DEST)/SQLiteAsync.cs

all-android: android-arm64 android-arm32 # android-x86_64 android-x86
all-ios: ios-arm64 ios-universal
all-macos: macos-universal
all-apple: all-macos all-ios

# Dockerized builds
docker-all-android:
	docker build -f tools~/Dockerfile.build.android --platform=linux/amd64 -t gilzoide-sqlite-net-build-android:latest $(DOCKER_BUILD_ARGS) .
	docker run --rm -v "$(CURDIR)":/src -w /src --platform=linux/amd64 gilzoide-sqlite-net-build-android:latest make all-android
docker-all-linux:
	docker build -f tools~/Dockerfile.build.linux --platform=linux/amd64 -t gilzoide-sqlite-net-build-linux:latest $(DOCKER_BUILD_ARGS) .
	docker run --rm -v "$(CURDIR)":/src -w /src --platform=linux/amd64 gilzoide-sqlite-net-build-linux:latest make all-linux
docker-all-windows:
	docker build -f tools~/Dockerfile.build.windows -t gilzoide-sqlite-net-build-windows:latest $(DOCKER_BUILD_ARGS) .
	docker run --rm -v "$(CURDIR)":/src -w /src gilzoide-sqlite-net-build-windows:latest make all-windows-mingw
docker-all-windows-llvm:
	docker run --rm -v "$(CURDIR)":/src -w /src mstorsjo/llvm-mingw:latest make all-windows-llvm-mingw

# Download builds from GitHub Actions
download-latest-build:
	$(eval TMPDIR := $(shell mktemp -d))
	$(GITHUB_CLI_BIN) run download $(RUN_ID) --repo $(GITHUB_REPO) -p gilzoide-sqlite-net-* -D $(TMPDIR)
	rsync -r $(TMPDIR)/gilzoide-sqlite-net-windows/* lib/windows/
	rsync -r $(TMPDIR)/gilzoide-sqlite-net-linux/* lib/linux/
	rsync -r $(TMPDIR)/gilzoide-sqlite-net-android/* lib/android/
	rsync -r $(TMPDIR)/gilzoide-sqlite-net-macos/* lib/macos/
	$(RM) -r $(TMPDIR)
