---
layout: post
title: OS X El Capitan에서 OpenJDK7 빌드하기
date: 2016-08-11 22:11:00 +0900
tags: [Java, OpenJDK]
---
최신 Mac OS X El Capitan에서 OpenJDK 7을 빌드해야 할 필요가 있어서 진행하는 중에 시행착오를 많이 겪어서
이를 기록해 둡니다.
혹시 저와 같은 환경에서 빌드해야 할 경우 도움이 되면 좋겠습니다.

# OpenJDK 7 바이너리 생성
* Mac OS X El Capitan에서 OpenJDK 7 빌드 시 필요한 내용을 정리합니다.

## 필수 구성 요소 사전 설치
* 기본적으로 Homebrew가 설치되어 있다는 전제 하에 진행합니다.
* 만약 Homebrew가 설치되어 있지 않다면 0. Homebrew 설치 항목을 참고하여 설치합니다.

### 0. Homebrew 설치
* 홈페이지 : [http://brew.sh/index_ko.html](http://brew.sh/index_ko.html)
* 터미널에 다음 스크립트를 붙여 넣은 후 실행합니다.
  ```
  /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
  ```

### 1. JDK 1.6 설치
* [https://support.apple.com/kb/DL1572?locale=ko_KR&viewlocale=en_US](https://support.apple.com/kb/DL1572?locale=ko_KR&viewlocale=en_US) 페이지 접속 후 [다운로드] 버튼을 클릭하여 javaforosx.dmg 파일을 다운로드 받아 설치합니다.

### 2. Mercurial 설치
* version : 3.8.4 +
* brew install mercurial

### 3. XCode 설치
* version : 7.3.1
* [https://developer.apple.com/xcode/](https://developer.apple.com/xcode/) 페이지 접속 후 애플 ID로 로그인 하여 버전 다운로드 및 설치합니다.

### 4. Ant 설치
* version : 1.9.7 +
* brew install ant

### 5. FreeType 설치
* version : 2.6.4 +
* brew install freetype

## OpenJDK 7 소스 받기
* hg clone http://hg.openjdk.java.net/jdk7u/jdk7u ${OpenJDK7_Installed_Directory}
* cd ${OpenJDK7_Installed_Directory}
* sh ./get_source.sh

## Env.sh 파일 생성 및 실행
* Env.sh 파일을 생성하여 다음 내용을 추가합니다.
* FreeType 관련 항목, ALT_OUTPUTDIR, ANT_HOME 값은 로컬 PC에 설치된 정보에 따라 수정합니다.
* 파일 생성 후 source ./Env.sh 실행
  ```
  export LANG=C

  export CC=clang

  export COMPILER_WARNINGS_FATAL=false

  export LFLAGS='-Xlinker -lstdc++'

  export USE_CLANG=true

  export ALLOW_DOWNLOADS=true

  export HOTSPOT_BUILD_JOBS=`sysctl -n hw.ncpu`

  export ALT_PARALLEL_COMPILE_JOBS=`sysctl -n hw.ncpu`

  export SKIP_COMPARE_IMAGES=true

  export USE_PRECOMPILED_HEADER=true

  export INCREMENTAL_BUILD=true

  export SKIP_DEBUG_BUILD=true
  export SKIP_FASTDEBUG_BUILD=false
  export DEBUG_NAME=debug

  export BUILD_DEPLOY=false
  export BUILD_INSTALL=false

  # FreeType
  export FREETYPE_LIB_PATH=/usr/X11/lib
  export FREETYPE_HEADERS_PATH=/usr/X11/include
  export ALT_FREETYPE_LIB_PATH=/usr/local/Cellar/freetype/2.6.4/lib
  export ALT_FREETYPE_HEADERS_PATH=/usr/local/Cellar/freetype/2.6.4/include

  export ALT_BOOTDIR=`/usr/libexec/java_home -v 1.6`

  export ALT_OUTPUTDIR=/Users/${USER_ID}/${OpenJDK7_Installed_Directory}/build

  export ANT_HOME=/usr/local/Cellar/ant/1.9.7

  unset JAVA_HOME
  unset CLASSPATH
  unset LD_LIBRARY_PATH
  ```

## 설정 확인
* make sanity 명령어를 실행합니다.
* error 없이 실행이 완료되면 build 폴더 내 sanityCheckMessages.txt 파일이 생성됩니다.
* 위 파일에서 구체적인 내용을 확인해 볼 수 있습니다.
* 만약 에러 발생 시 출력된 메시지를 토대로 에러 수정 후 재 수행합니다.

## 파일 수정
* 한글 Mac에서 빌드하는 경우 javac의 encoding 문제로 인해 corba project를 빌드하다가 에러가 발생하는데 다음을 참고합니다.
  * error 내용 : "unmappable character for encoding ascii"
  * corba/make/common/shared/Defs-java.gmk 파일을 수정합니다.
    ```
    JAVACFLAGS  += -encoding ascii -&gt; JAVACFLAGS  += -encoding ms949
    ```
* clang unknown argument 에러가 발생하면 다음을 참고합니다.
  * error 내용 : clang: error: unknown argument: '-fpch-deps'
  * hotspot/make/bsd/makefiles/gcc.make 파일을 수정합니다.
    * line 216
      ```
      # Flags for generating make dependency flags.
      #ifneq ("${CC_VER_MAJOR}", "2")
      #DEPFLAGS = -fpch-deps -MMD -MP -MF $(DEP_DIR)/$(@:%=%.d)
      #endif
      # Flags for generating make dependency flags.
      DEPFLAGS = -MMD -MP -MF $(DEP_DIR)/$(@:%=%.d)
      ifeq ($(USE_CLANG),)
          ifneq ($(CC_VER_MAJOR), 2)
              DEPFLAGS += -fpch-deps
          endif
      endif
      ```
* 기본 파라미터 값으로 인한 에러가 발생하면 다음을 참고합니다.
  * error 내용
    ```
    ${OpenJDK7}/hotspot/src/share/vm/code/relocInfo.hpp:374:27: error: friend declaration specifying a default argument must be a definition
      inline friend relocInfo prefix_relocInfo(int datalen = 0);

    ${OpenJDK7}/hotspot/src/share/vm/code/relocInfo.hpp:469:18: error: friend declaration specifying a default argument must be the only declaration
    inline relocInfo prefix_relocInfo(int datalen) {
    ```
  * hotspot/src/share/vm/code/relocInfo.hpp 파일을 수정합니다.
    * line 374
      ```
      [수정 전]
      inline friend relocInfo prefix_relocInfo(int datalen = 0);

      [수정 후]
      inline friend relocInfo prefix_relocInfo(int datalen);
      ```
  * error 내용
    ```
    ${OpenJDK7}/hotspot/src/share/vm/code/relocInfo.hpp:470:21: error: 'fits_into_immediate' is a protected member of 'relocInfo'
      assert(relocInfo::fits_into_immediate(datalen), "datalen in limits");

    ${OpenJDK7}/hotspot/src/share/vm/code/relocInfo.hpp:471:59: error: 'RAW_BITS' is a protected member of 'relocInfo'
      return relocInfo(relocInfo::data_prefix_tag, relocInfo::RAW_BITS, relocInfo::datalen_tag | datalen);

    ${OpenJDK7}/hotspot/src/share/vm/code/relocInfo.hpp:471:10: error: calling a protected constructor of class 'relocInfo'
  return relocInfo(relocInfo::data_prefix_tag, relocInfo::RAW_BITS, relocInfo::datalen_tag | datalen);
    ```
  * hotspot/src/share/vm/code/relocInfo.hpp 파일을 수정합니다.
    * line 469
      ```
      [수정 전]
      inline relocInfo prefix_relocInfo(int datalen) {

      [수정 후]
      inline relocInfo prefix_relocInfo(int datalen = 0) {
      ```
* clang linker command failed 에러가 발생하면 다음을 참고합니다.
  * error 내용
    ```
    Undefined symbols for architecture x86_64:
    "_attachCurrentThread", referenced from:
        +[ThreadUtilities getJNIEnv] in ThreadUtilities.o
        +[ThreadUtilities getJNIEnvUncached] in ThreadUtilities.o
    ld: symbol(s) not found for architecture x86_64
    clang: error: linker command failed with exit code 1 (use -v to see invocation)
    ```
  * jdk/src/macosx/native/sun/osxapp/ThreadUtilities.m 파일을 수정합니다.
    * line 38
      ```
      [수정 전]
      inline void attachCurrentThread(void** env) {

      [수정 후]
      static inline void attachCurrentThread(void** env) {
      ```

## Build
* make debug_build 2>&1 |tee $ALT_OUTPUTDIR/build.log를 실행합니다.
* 빌드 완료되면 build-debug 폴더 내 결과물이 생성됩니다.

## 빌드 결과
* 빌드 완료 시 다음과 같이 결과를 확인할 수 있습니다.

   ```
    >>>Finished making images @ Fri Aug 12 16:50:57 KST 2016 ...
    ########################################################################
    ##### Leaving jdk for target(s) sanity all  images                 #####
    ########################################################################
    ##### Build time 00:45:26 jdk for target(s) sanity all  images     #####
    ########################################################################

    #-- Build times ----------
    Target debug_build
    Start 2016-08-12 14:41:39
    End   2016-08-12 16:50:57
    00:03:26 corba
    01:18:50 hotspot
    00:00:26 jaxp
    00:00:32 jaxws
    00:45:26 jdk
    00:00:38 langtools
    02:09:18 TOTAL
    -------------------------
   ```

## 확인
* ./build-debug/bin/java -version
* 버전 정보를 확인합니다.
  ```
  openjdk version "1.7.0-internal-debug"
  OpenJDK Runtime Environment (build 1.7.0-internal-debug-b00)
  OpenJDK 64-Bit Server VM (build 24.95-b00-jvmg, mixed mode)
  ```

## 참고 자료
* [http://zhongmingmao.me/2016/07/13/openjdk.html](http://zhongmingmao.me/2016/07/13/openjdk.html)
* [https://blogs.oracle.com/arungupta/entry/build_open_jdk_7_on](https://blogs.oracle.com/arungupta/entry/build_open_jdk_7_on)
* [https://github.com/hgomez/obuildfactory/wiki/Building-and-Packaging-OpenJDK7-for-OSX](https://github.com/hgomez/obuildfactory/wiki/Building-and-Packaging-OpenJDK7-for-OSX)
* [http://seyoony.tistory.com/entry/OpenJDK-%EB%B9%8C%EB%93%9C%ED%95%98%EA%B8%B0](http://seyoony.tistory.com/entry/OpenJDK-%EB%B9%8C%EB%93%9C%ED%95%98%EA%B8%B0)
* [http://mail.openjdk.java.net/pipermail/macosx-port-dev/2013-December/006318.html](http://mail.openjdk.java.net/pipermail/macosx-port-dev/2013-December/006318.html)
* [http://openjdk.5641.n7.nabble.com/Heads-up-anyone-who-keeps-their-Mac-software-up-to-date-td155350.html](http://openjdk.5641.n7.nabble.com/Heads-up-anyone-who-keeps-their-Mac-software-up-to-date-td155350.html)
* [http://www.programering.com/a/MTN1kTMwATg.html](http://www.programering.com/a/MTN1kTMwATg.html)
* [http://gvsmirnov.ru/blog/tech/2014/02/07/building-openjdk-8-on-osx-maverick.html#just-a-couple-of-fixes-here-and-there-and-also-there-and-there](http://gvsmirnov.ru/blog/tech/2014/02/07/building-openjdk-8-on-osx-maverick.html#just-a-couple-of-fixes-here-and-there-and-also-there-and-there)
* [https://github.com/manasthakur/tech/wiki/Building-OpenJDK-8-on-Mac-OS-X-Yosemite](https://github.com/manasthakur/tech/wiki/Building-OpenJDK-8-on-Mac-OS-X-Yosemite)
* [http://hadwinzhy.github.io/2013/01/21/compile-jdk/](http://hadwinzhy.github.io/2013/01/21/compile-jdk/)
* [https://bugs.openjdk.java.net/browse/JDK-8043646](https://bugs.openjdk.java.net/browse/JDK-8043646)
