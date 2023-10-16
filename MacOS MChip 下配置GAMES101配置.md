一句話總結----CLion才是最終解決之道。
經過了數個小時的摸爬滾打，終於在m系列芯片下配置好了GAMES101的開發環境。
本文簡單紀錄一下配置歷程，以及詳細配置流程。

<!--more-->


# MacOS MChip 下配置GAMES101配置

> 一句話總結----CLion才是最終解決之道。
>
> 經過了數個小時的摸爬滾打，終於在m系列芯片下配置好了GAMES101的開發環境。
>
> 本文簡單紀錄一下配置歷程，以及詳細配置流程。

## 歷程

最開始覺得沒必要用虛擬機，所以並沒有打算和「Assignment  0」那樣使用Oracle VM VirtualBox虛擬機。於是我就在MacOS環境下開始配置。

### 折騰第一步：vscode

參考了三篇關於在m1芯片配置GAMES101環境的文章。

> 1. https://www.codenong.com/cs106826133
> 2. https://zhuanlan.zhihu.com/p/371245964
> 3. https://zhuanlan.zhihu.com/p/472114465

簡單概括就是：

1. 安裝homebrew（正常mac開發者必備，此處略）

2. 通過homebrew安裝GCC / CMake / Eigen / OpenCV（最新版本目前還沒有遇到問題）。

   ```
   brew install gcc
   brew install cmake
   brew install eigen
   brew install opencv
   ```

3. 分別檢查是否安裝成功：

   1. gcc

      終端輸入以下命令，

      ```
      clang --version
      ```

      得到以下結果

      > ```
      > Apple clang version 14.0.0 (clang-1400.0.29.202)
      > Target: arm64-apple-darwin22.3.0
      > Thread model: posix
      > InstalledDir: /Library/Developer/CommandLineTools/usr/bin
      > ```

      發現並不是剛才通過homebrew安裝的clang，而是Apple系統自帶的。

      我們應通過以下命令檢查：

      ```
      brew ls gcc
      ```

      安裝成功會有以下顯示：

      ```
      /opt/homebrew/Cellar/gcc/12.2.0/bin/aarch64-apple-darwin21-c++-12
      /opt/homebrew/Cellar/gcc/12.2.0/bin/aarch64-apple-darwin21-g++-12
      /opt/homebrew/Cellar/gcc/12.2.0/bin/aarch64-apple-darwin21-gcc-12
      /opt/homebrew/Cellar/gcc/12.2.0/bin/aarch64-apple-darwin21-gcc-ar-12
      /opt/homebrew/Cellar/gcc/12.2.0/bin/aarch64-apple-darwin21-gcc-nm-12
      /opt/homebrew/Cellar/gcc/12.2.0/bin/aarch64-apple-darwin21-gcc-ranlib-12
      /opt/homebrew/Cellar/gcc/12.2.0/bin/aarch64-apple-darwin21-gfortran-12
      /opt/homebrew/Cellar/gcc/12.2.0/bin/c++-12
      /opt/homebrew/Cellar/gcc/12.2.0/bin/cpp-12
      /opt/homebrew/Cellar/gcc/12.2.0/bin/g++-12
      /opt/homebrew/Cellar/gcc/12.2.0/bin/gcc-12
      /opt/homebrew/Cellar/gcc/12.2.0/bin/gcc-ar-12
      /opt/homebrew/Cellar/gcc/12.2.0/bin/gcc-nm-12
      /opt/homebrew/Cellar/gcc/12.2.0/bin/gcc-ranlib-12
      /opt/homebrew/Cellar/gcc/12.2.0/bin/gcov-12
      /opt/homebrew/Cellar/gcc/12.2.0/bin/gcov-dump-12
      /opt/homebrew/Cellar/gcc/12.2.0/bin/gcov-tool-12
      /opt/homebrew/Cellar/gcc/12.2.0/bin/gfortran
      /opt/homebrew/Cellar/gcc/12.2.0/bin/gfortran-12
      /opt/homebrew/Cellar/gcc/12.2.0/bin/lto-dump-12
      /opt/homebrew/Cellar/gcc/12.2.0/include/c++/ (806 files)
      /opt/homebrew/Cellar/gcc/12.2.0/lib/gcc/ (607 files)
      /opt/homebrew/Cellar/gcc/12.2.0/libexec/gcc/ (14 files)
      /opt/homebrew/Cellar/gcc/12.2.0/share/gcc-12/ (4 files)
      /opt/homebrew/Cellar/gcc/12.2.0/share/man/ (11 files)
      ```

      若未成功，則需要安裝檢查是否安裝了Xcode。

   2. cmake

      終端輸入以下命令，

      ```
      cmake --version
      ```

      安裝成功會有以下顯示：

      > ```
      > cmake version 3.26.3
      > 
      > CMake suite maintained and supported by Kitware (kitware.com/cmake).
      > ```

   3. eigen

      終端輸入以下命令，

      ```
      brew ls eigen
      ```

      安裝成功會有以下顯示：

      >  ```
      >  /opt/homebrew/Cellar/eigen/3.4.0_1/include/eigen3/ (531 files)
      >  /opt/homebrew/Cellar/eigen/3.4.0_1/share/cmake/Modules/FindEigen3.cmake
      >  /opt/homebrew/Cellar/eigen/3.4.0_1/share/eigen3/ (4 files)
      >  /opt/homebrew/Cellar/eigen/3.4.0_1/share/pkgconfig/eigen3.pc
      >  ```

      然後運行以下命令將```/opt/homebrew/Cellar/eigen/3.4.0_1/include/eigen3/```軟鏈接到系統文件夾中。

      ```
      brew link --overwrite eigen
      ```

   4. opencv

      同上(3)。

4. 安裝VSCode插件

   - C/C++

     <img src="https://s2.loli.net/2023/04/06/CVpHnODbqAit7lQ.png" alt="image-20230406230431915" style="zoom:50%;" />

   - C/C++ Extension Pack

     <img src="https://s2.loli.net/2023/04/06/jpnRbO2KLma98E6.png" alt="image-20230406230459957" style="zoom:50%;" />

   - CMake Tools

     <img src="https://s2.loli.net/2023/04/06/doX4eqmnBuCGRVQ.png" alt="image-20230406230551690" style="zoom:50%;" />

5. 用VSCode打開Assignment1

   雙擊CMakeLists.txt，Ctrl+s，自動運行Cmake，生成以下文件：

   <img src="https://s2.loli.net/2023/04/06/phdv38Y5Z1egD2l.png" alt="image-20230406232030044" style="zoom:50%;" />

   在VSCode最下面選擇Clang編譯。

   <img src="https://s2.loli.net/2023/04/06/HjQ8chXvaEuK1iO.png" alt="image-20230406232122553" style="zoom:50%;" />

   

至此，上述三篇文章大致已經結束了。但是我依舊出現一個問題：

### 報錯：**「提示未找到eigen」。**

詳細報錯提示：

> 根據 configurationProvider 設定提供的資訊，偵測到 #include 錯誤。已停用此編譯單位 (/Users/remooo/Project/GAMES101_Homework_S2021/Homework1123/Assignment1/Triangle.hpp) 的波浪線。C/C++(1696)
>
> 無法開啟 來源 檔案 "eigen3/Eigen/Eigen"C/C++(1696)
>
> <img src="https://s2.loli.net/2023/04/06/NhKEJmt4Ws2czPV.png" alt="image-20230406231435290" style="zoom:50%;" />
>
> 當前的CmakeLists.txt內容：
>
> ```
> cmake_minimum_required(VERSION 3.10)
> project(Rasterizer)
> 
> find_package(OpenCV REQUIRED)
> 
> set(CMAKE_CXX_STANDARD 17)
> 
> include_directories(/usr/local/include)
> 
> add_executable(Rasterizer main.cpp rasterizer.hpp rasterizer.cpp Triangle.hpp Triangle.cpp)
> target_link_libraries(Rasterizer ${OpenCV_LIBRARIES})
> 
> ```

### 解決「提示未找到eigen問題」

於是經過一番檢索，我將問題定位到CMakeLists.txt中的第8行語句：

```
include_directories(/opt/homebrew/include)
```

也許是vscode不能通過這個路徑找到eigen，於是我在CMakeLists.txt中，将：

```text
include_directories(/usr/local/include)
```

更改为：

```text
include_directories(/opt/homebrew/include)
```

然後Ctrl+s。

成功解決「提示未找到eigen問題」。

但是新的問題又出現了。

### 報錯：**「命名空間 "Eigen" 沒有成員 "xxx"」**

詳細報錯提示：

><img src="https://s2.loli.net/2023/04/06/nyd9GX2KgerYSaO.png" alt="image-20230406232442165" style="zoom:50%;" />
>
><img src="https://s2.loli.net/2023/04/06/JLjP5KpZcmitxlw.png" alt="image-20230406232517791" style="zoom:50%;" />



**目前，這個問題還沒有得到解決。**

於是，我回到了最初的方向----虛擬機。

直接說結論：

### Oracle VM VirtualBox目前還不能在MClip上運行x64虛擬機

https://www.virtualbox.org/wiki/Downloads

截至目前，最新版本7.0.6版本不能安裝GAMES101的虛擬機。

<img src="https://s2.loli.net/2023/04/06/Fj4EXgWQO3TYZfD.png" alt="image-20230406232835890" style="zoom:50%;" />



走到這一步，時間已經過去6個小時了，安裝進度還是在原地踏步。

正當我絕望地瀏覽著討論區的時候，

<img src="https://s2.loli.net/2023/04/06/HLar621VjM7vSfo.png" alt="image-20230406233400996" style="zoom:50%;" />

我在第12頁中找到了這樣一篇文章：「[macbook 使用vscode完成作业0的环境配置](https://games-cn.org/forums/topic/macbook-shiyongvscodewanchengzuoye0dehuanjingpeizhi/)」

<img src="https://s2.loli.net/2023/04/06/mQt7IWfujVRGZ9n.png" alt="image-20230406233436623" style="zoom:50%;" />

文章下面的一條評論寫著：「这边建议用一下clion，至少我在vscode折腾了一下午依旧环境没配好之后，换到clion就完美解决了。」

<img src="https://s2.loli.net/2023/04/06/Vukc4sM1rPEHtva.png" alt="image-20230406233559307" style="zoom:50%;" />

懷著激動的心情，連忙下載並~~破解了~~CLion。

![image-20230406233654693](https://s2.loli.net/2023/04/06/O6mDFW7o2jfHslP.png)

## 折中的解決方案——CLion

確保CMakeLists.txt和我的類似：

```
cmake_minimum_required(VERSION 3.10)
project(Rasterizer)

find_package(OpenCV REQUIRED)

set(CMAKE_CXX_STANDARD 17)
include_directories(/opt/homebrew/Cellar/eigen/3.4.0_1/include)

add_executable(Rasterizer main.cpp rasterizer.hpp rasterizer.cpp Triangle.hpp Triangle.cpp)
target_link_libraries(Rasterizer ${OpenCV_LIBRARIES})

```

注意第8行被我改成了：

```
include_directories(/opt/homebrew/Cellar/eigen/3.4.0_1/include)
```

並且點擊懸浮的編譯cmake按鈕就可以了。（此前在VSCode中無效）

大功告成：<img src="https://s2.loli.net/2023/04/06/7egutlMAUNWmYEf.png" alt="image-20230406234123839" style="zoom:50%;" />

----

終於可以愉快的coding了。