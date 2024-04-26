## lua的热更新

### 什么是热更新

在实际项目中，热更新的Lua代码通过资源更新的方式下载到包中，程序加载Lua代码并交给Lua虚拟机执行

即在运行时候，lua代码文件每次调用时重新读取

我们知道lua加载一个文件的方式可以有：dofile，loadfile以及 require。其中loadfile是只编译不执行，dofile和require是同时编译和执行。而dofile和require的区别是dofile同一个文件每次都要加载，也就是说，dofile两次返回来的是两个不同的地址。而require同一个文件，不管多少次都是都返回同一个地址，其原因是lua的地址缓存在了package.load（）中。所以效率比dofile要高许多，因而现在一般都是用require加载文件。

```
function reload_module(module_name)
    local old_module = package.loaded[module_name] or {}
    package.loaded[module_name] = nil
    require (module_name)
 
    local new_module = package.loaded[module_name]
    for k, v in pairs(new_module) do
        old_module[k] = v
    end
 
    package.loaded[module_name] = old_module
    return old_module
end
```



### 热更新流程

1.代码更改 2.上传新代码 3.重载lua代码（loadfile之类）4.切换代码帧 5.GC回收旧代码 6.错误处理

![liu](F:\github_project\lua\liu.png)

热更新模块

一般来说需要热更的话，是你修改了某个`XXXModel.lua`文件，这个文件在`package.loaded`中名为`XXXSystem.XXXModel`。其中`XXXSystem`是这个Lua模块存放的文件夹名称。

```
local oldModule
if package.loaded[packageName] then
    oldModule = package.loaded[packageName]
    package.loaded[packageName] = nil
end
--热更之前要先保存旧模块的全部数据
```

之后直接`require`新的模块，然后把新模块记录下来，遍历新模块的所有数据。总体来说，遍历的过程中，元素如果是table就保留旧模块的，如果是function就用新模块的。

当然要注意，table会嵌套table和function，因此这是一个递归的过程。

还有，function要用新的，但是function的的upvalue要用旧的。

table中的metatable同样作为table处理，使用`debug.getmetatable`获取一个table的metatable然后进行与table一样的操作。

对于可能出现循环引用的情况，可以在更新表的时候记录已更新的table，避免重复处理死循环。

监听模块

热更可以用在编辑器下，同样可以在线上环境使用（当然要有更严格的限制）。在编辑器下热更的话，要监听本地lua文件的变化，



```
--Unity编辑器中可以使用`FileSystemWatcher`来实现监听，
--可以把这个功能封装到一个`DirectoryWatcher`类里，方便监听指定的多个文件夹。
if (!Directory.Exists(dirPath)) 
	return;
var watcher = new FileSystemWatcher();
watcher.IncludeSubdirectories = true;
watcher.Path = dirPath;
watcher.NotifyFilter = NotifyFilters.LastWrite;
watcher.Filter = "*.lua";
watcher.Changed += handler;
watcher.EnableRaisingEvents = true;
watcher.InternalBufferSize = 10240;

```

编辑器下游戏启动时创建`DirectoryWatcher`监听指定文件夹，并写处理函数`LuaFileOnChanged`

```
var luaDirWatcher = new DirectoryWatcher(LuaConst.luaDir, new FileSystemEventHandler(LuaFileOnChanged));//监听lua文件
```

触发`LuaFileOnChanged`的时候调用对应的Lua方法重载该文件模块即可。