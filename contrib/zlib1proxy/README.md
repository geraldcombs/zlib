# zlib1proxy: Optional CMake target that creates a proxy zlib1.dll

This adds a CMake target that creates a proxy DLL named *zlib1.dll* which forwards calls to the main libz.dll.
Microsoft Visual Studio is currently required to build the DLL.
To create the proxy, do the following:

* Pass `-DZLIB_BUILD_ZLIB1_PROXY_DLL=ON` to CMake.
* Build the `zlib1proxy` target.