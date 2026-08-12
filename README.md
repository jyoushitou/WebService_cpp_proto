# RPC Proto 仓库

## 编译方法

```bash
cd proto/source
mkdir -p ../build
cd ../build
cmake ../source
make -j$(nproc)
```

## 在其他仓库中使用

### CMake方式引用

```cmake
# 在微服务的CMakeLists.txt中
find_package(rpc_protos REQUIRED)
target_link_libraries(your_service rpc_protos)
```

### 手动引用

```bash
# 将编译后的库和头文件复制到微服务项目中
cp -r proto/build/lib/* your_service/lib/
cp -r proto/build/generated/* your_service/include/
```