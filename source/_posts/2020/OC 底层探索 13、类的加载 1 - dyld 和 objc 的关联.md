---
title: 类的加载 1 - dyld 和 objc 的关联
date: 2020-10-12 23:07:00
categories: 
- OC底层探索
tags:
- OC
- 底层
---

本文开始探索类的加载。[调试源码 objc 源码工程](https://github.com/andyccc/objc)

_objc_init 函数：



```
1 /***********************************************************************
 2 * _objc_init
 3 * Bootstrap initialization. Registers our image notifier with dyld.
 4 * Called by libSystem BEFORE library initialization time
 5 **********************************************************************/
 7 void _objc_init(void)
 8 {
 9     static bool initialized = false;
10     if (initialized) return;
11     initialized = true;
13     // fixme defer initialization until an objc-using image is found?
14     environ_init();
15     tls_init();
16     static_init();
17     runtime_init();
18     exception_init();
19     cache_init();
20     _imp_implementationWithBlock_init();
22     _dyld_objc_notify_register(&map_images, load_images, unmap_image);
24 #if __OBJC2__
25     didCallDyldNotifyRegister = true;
26 #endif
27 }
```



一、各 init 初始化的简单了解
-----------------

### 1）环境变量的初始化与简单使用

我们从 14 行 **environ_init()** 环境变量初始化开始进行探究。

environ_init() 函数：

读取影响运行时的环境变量



```
1 /***********************************************************************
 2 * environ_init
 3 * Read environment variables that affect the runtime.
 4 * Also print environment variable help, if requested.
 5 **********************************************************************/
 6 void environ_init(void) 
 7 {
 8     if (issetugid()) {
 9         // All environment variables are silently ignored when setuid or setgid
10         // This includes OBJC_HELP and OBJC_PRINT_OPTIONS themselves.
11         return;
12     } 
14     bool PrintHelp = false;
15     bool PrintOptions = false;
16     bool maybeMallocDebugging = false;
18     // Scan environ[] directly instead of calling getenv() a lot.
19     // This optimizes the case where none are set.
20     for (char **p = *_NSGetEnviron(); *p != nil; p++) {
21             ...... // 代码不全部展示了 
22     }
24     // Special case: enable some autorelease pool debugging 
25     // when some malloc debugging is enabled 
26     // and OBJC_DEBUG_POOL_ALLOCATION is not set to something other than NO.
27     if (maybeMallocDebugging) {
28         ...... // 代码不全部展示了
29     }
31     // Print OBJC_HELP and OBJC_PRINT_OPTIONS output.
32     if (PrintHelp  ||  PrintOptions) {
33         if (PrintHelp) {
34             _objc_inform("Objective-C runtime debugging. Set variable=YES to enable.");
35             _objc_inform("OBJC_HELP: describe available environment variables");
36             if (PrintOptions) {
37                 _objc_inform("OBJC_HELP is set");
38             }
39             _objc_inform("OBJC_PRINT_OPTIONS: list which options are set");
40         }
41         if (PrintOptions) {
42             _objc_inform("OBJC_PRINT_OPTIONS is set");
43         }
45         for (size_t i = 0; i < sizeof(Settings)/sizeof(Settings[0]); i++) {
46             const option_t *opt = &Settings[i];            
47             if (PrintHelp) _objc_inform("%s: %s", opt->env, opt->help);
48             if (PrintOptions && *opt->var) _objc_inform("%s is set", opt->env);
49         }
50     }
52     // 我加的调试代码
53     for (size_t i = 0; i < sizeof(Settings)/sizeof(Settings[0]); i++) {
54         const option_t *opt = &Settings[i];
55         _objc_inform("%s: %s", opt->env, opt->help);
56         _objc_inform("%s is set", opt->env);
57     }
58 }
```



我们通过运行工程 (objc 源码工程)，发现上述代码都没有走进去，把打印输出提取出来(52 行) 再次运行，可看到输出信息如下(信息比较长截取部分)：

![](https://img2020.cnblogs.com/blog/842658/202010/842658-20201018123349272-1017030560.png)

### 环境变量的使用

上图输出结果倒数 4 行，我们看到是 **OBJC_DISABLE_NONPOINTER_ISA** 是 isa 的设置，我们在工程中 Edit Scheme 中添加此属性，如下图：

![](https://img2020.cnblogs.com/blog/842658/202010/842658-20201018123743932-1568502516.png) 

main.m 中创建一个 MYPerson，不同场景运行工程结果如下：

![](https://img2020.cnblogs.com/blog/842658/202010/842658-20201018131543337-1458825048.png)

我们可通过环境换量的设置，更便捷的对工程有整体的了解。

更直观的，我们添加 OBJC_PRINT_LOAD_METHODS : YES

注释掉上述调试代码 (52~57)，再次运行，所有的 load 方法都输出了，如下：

![](https://img2020.cnblogs.com/blog/842658/202010/842658-20201018132242405-1236033854.png)

### 2）tls_init() -

关于线程 key 的绑定 - 例如现成的析构函数。(runloop / 自动释放池等都依赖于线程)



```
1 void tls_init(void)
2 {
3 #if SUPPORT_DIRECT_THREAD_KEYS
4     pthread_key_init_np(TLS_DIRECT_KEY, &_objc_pthread_destroyspecific);
5 #else
6     _objc_pthread_key = tls_create(&_objc_pthread_destroyspecific);
7 #endif
8 }
```



### static_init() - objc 中的静态构造函数

运行 C++ 静态构造函数，它们的调用会比 dyld 还早。



```
1 /***********************************************************************
 2 * static_init
 3 * Run C++ static constructor functions.
 4 * libc calls _objc_init() before dyld would call our static constructors, 
 5 * so we have to do it ourselves.
 6 **********************************************************************/
 7 static void static_init()
 8 {
 9     size_t count;
10     auto inits = getLibobjcInitializers(&_mh_dylib_header, &count);
11     for (size_t i = 0; i < count; i++) {
12         inits[i]();
13     }
14 }
```



### runtime_init() 

runtime 运行时环境初始化

```
1 void runtime_init(void)
2 {
3     objc::unattachedCategories.init(32);// 分类处理的初始化
4     objc::allocatedClasses.init();// 表 用来储存加载的类
5 }
```

### exception_init() - 异常

初始化 libobjc 的异常处理系统。

crash 系统抛出异常，程序中断



```
1 /***********************************************************************
2 * exception_init
3 * Initialize libobjc's exception handling system.
4 * Called by map_images().
5 **********************************************************************/
6 void exception_init(void)
7 {
8     old_terminate = std::set_terminate(&_objc_terminate);
9 }
```





```
1 /***********************************************************************
 2 * _objc_terminate
 3 * Custom std::terminate handler.
 4 *
 5 * The uncaught exception callback is implemented as a std::terminate handler. 
 6 * 1. Check if there's an active exception
 7 * 2. If so, check if it's an Objective-C exception
 8 * 3. If so, call our registered callback with the object.
 9 * 4. Finally, call the previous terminate handler.
10 **********************************************************************/
11 static void (*old_terminate)(void) = nil;
12 static void _objc_terminate(void)
13 {
14     if (PrintExceptions) {
15         _objc_inform("EXCEPTIONS: terminating");
16     }
18     if (! __cxa_current_exception_type()) {
19         // No current exception.
20         (*old_terminate)();
21     }
22     else {
23         // There is a current exception. Check if it's an objc exception.
24         @try {
25             __cxa_rethrow();
26         } @catch (id e) {
27             // It's an objc object. Call Foundation's handler, if any. 句柄 函数回调 - 截取异常
28             (*uncaught_handler)((id)e);
29             (*old_terminate)();
30         } @catch (...) {
31             // It's not an objc object. Continue to C++ terminate.
32             (*old_terminate)();
33         }
34     }
35 }
```



### cache_init()

缓存条件初始化



```
1 void cache_init()
 2 {
 3 #if HAVE_TASK_RESTARTABLE_RANGES
 4     mach_msg_type_number_t count = 0;
 5     kern_return_t kr;
 7     while (objc_restartableRanges[count].location) {
 8         count++;
 9     }
11     kr = task_restartable_ranges_register(mach_task_self(),
12                                           objc_restartableRanges, count);
13     if (kr == KERN_SUCCESS) return;
14     _objc_fatal("task_restartable_ranges_register failed (result 0x%x: %s)",
15                 kr, mach_error_string(kr));
16 #endif // HAVE_TASK_RESTARTABLE_RANGES
17 }
```



### _imp_implementationWithBlock_init()

启用回调机制。通常情况下不做什么操作，因为所有的初始化都是懒加载的，但对于某些进程，迫切的需要加载 trampolines dylib.



```
1 /// Initialize the trampoline machinery. Normally this does nothing, as
 2 /// everything is initialized lazily, but for certain processes we eagerly load  3 /// the trampolines dylib.
 4 void
 5 _imp_implementationWithBlock_init(void)
 6 {
 7 #if TARGET_OS_OSX
 8     // Eagerly load libobjc-trampolines.dylib in certain processes. Some
 9     // programs (most notably QtWebEngineProcess used by older versions of
10     // embedded Chromium) enable a highly restrictive sandbox profile which
11     // blocks access to that dylib. If anything calls
12     // imp_implementationWithBlock (as AppKit has started doing) then we'll
13     // crash trying to load it. Loading it here sets it up before the sandbox
14     // profile is enabled and blocks it.
15     //
16     // This fixes EA Origin (rdar://problem/50813789)
17     // and Steam (rdar://problem/55286131)
18     if (__progname &&
19         (strcmp(__progname, "QtWebEngineProcess") == 0 ||
20          strcmp(__progname, "Steam Helper") == 0)) {
21         Trampolines.Initialize();
22     }
23 #endif
24 }
```



二、_dyld_objc_notify_register() - 
---------------------------------

![](https://img2020.cnblogs.com/blog/842658/202010/842658-20201018142229936-1774839960.png) 

代码 --> 编译 --> machO --> 内存

把 macho 各式各样数据加载到内存中，如何加载？--> map_images 镜像文件装载到内存 

_dyld_objc_notify_register() 是 dyld 库 的调用 (跨库，实现在 dyld 库中)。

执行流程：**dyld 中 从 dyld_start 开始 --> 一系列操作 --> 跳 objc 中 处理完成 --> 回到 dyld 中继续执行 --> main 入口**。 具体流程可见 [OC 底层探索 12]().



```
1 /**
 2     mapped -- &map_images
 3     init -- load_images
 4 */
 5 void _dyld_objc_notify_register(_dyld_objc_notify_mapped    mapped,
 6                                 _dyld_objc_notify_init      init,
 7                                 _dyld_objc_notify_unmapped  unmapped)
 8 {
 9     if ( gUseDyld3 )
10         return dyld3::_dyld_objc_notify_register(mapped, init, unmapped);
12     DYLD_LOCK_THIS_BLOCK;
13     static bool (*p)(_dyld_objc_notify_mapped, _dyld_objc_notify_init, _dyld_objc_notify_unmapped) = NULL;
15     if(p == NULL)
16         _dyld_func_lookup("__dyld_objc_notify_register", (void**)&p);
17     p(mapped, init, unmapped);
18 }
```



map_images / load_images / unmap_image 何时调:

**dyld:** 



```
1 void registerObjCNotifiers(_dyld_objc_notify_mapped mapped, _dyld_objc_notify_init init, _dyld_objc_notify_unmapped unmapped)
 2 {
 3     // record functions to call
 4     sNotifyObjCMapped = mapped;  5     sNotifyObjCInit = init;  6     sNotifyObjCUnmapped = unmapped;  7 
 8     // call 'mapped' function with all images mapped so far
 9     try {
10         notifyBatchPartial(dyld_image_state_bound, true, NULL, false, true);
11     }
12     catch (const char* msg) {
13         // ignore request to abort during registration
14     }
16     // <rdar://problem/32209809> call 'init' function on all images already init'ed (below libSystem)
17     for (std::vector<ImageLoader*>::iterator it=sAllImages.begin(); it != sAllImages.end(); it++) {
18         ImageLoader* image = *it;
19         if ( (image->getState() == dyld_image_state_initialized) && image->notifyObjC() ) {
20             dyld3::ScopedTimer timer(DBG_DYLD_TIMING_OBJC_INIT, (uint64_t)image->machHeader(), 0, 0);
21             (*sNotifyObjCInit)(image->getRealPath(), image->machHeader()); 22         }
23     }
24 }
```



**sNotifyObjCMapped** --> **notifyBatchPartial() : (*sNotifyObjCMapped)(objcImageCount, paths, mhs);**

**sNotifyObjCInit** --> **notifySingle() : (*sNotifyObjCInit)(image->getRealPath(), image->machHeader());**

**sNotifyObjCUnmapped** --> **removeImage() : (*sNotifyObjCUnmapped)(image->getRealPath(), image->machHeader());**

### 1）map_images 



```
1 /***********************************************************************
 2 * map_images
 3 * Process the given images which are being mapped in by dyld.
 4 * Calls ABI-agnostic code after taking ABI-specific locks.
 5 *
 6 * Locking: write-locks runtimeLock
 7 **********************************************************************/
 8 void
 9 map_images(unsigned count, const char * const paths[],
10            const struct mach_header * const mhdrs[])
11 {
12     mutex_locker_t lock(runtimeLock);
13     return map_images_nolock(count, paths, mhdrs);
14 }
```



跳到 map_images_nolock() 实现: 

![](https://img2020.cnblogs.com/blog/842658/202010/842658-20201018151527890-1733268486.png)

**_read_images()**：代码比较长



```
1 /***********************************************************************
  2 * _read_images
  3 * Perform initial processing of the headers in the linked 
  4 * list beginning with headerList. 
  5 *
  6 * Called by: map_images_nolock
  7 *
  8 * Locking: runtimeLock acquired by map_images
  9 **********************************************************************/
 10 void _read_images(header_info **hList, uint32_t hCount, int totalClasses, int unoptimizedTotalClasses)
 11 {
 12     header_info *hi;
 13     uint32_t hIndex;
 14     size_t count;
 15     size_t i;
 16     Class *resolvedFutureClasses = nil;
 17     size_t resolvedFutureClassCount = 0;
 18     static bool doneOnce;
 19     bool launchTime = NO;
 20     TimeLogger ts(PrintImageTimes);
 22     runtimeLock.assertLocked();
 24 #define EACH_HEADER \
 25     hIndex = 0;         \
 26     hIndex < hCount && (hi = hList[hIndex]); \
 27     hIndex++
 29     if (!doneOnce) {
 30         doneOnce = YES;
 31         launchTime = YES;
 33 #if SUPPORT_NONPOINTER_ISA
 34         // Disable non-pointer isa under some conditions.
 36 # if SUPPORT_INDEXED_ISA
 37         // Disable nonpointer isa if any image contains old Swift code
 38         for (EACH_HEADER) {
 39             if (hi->info()->containsSwift()  &&
 40                 hi->info()->swiftUnstableVersion() < objc_image_info::SwiftVersion3)
 41             {
 42                 DisableNonpointerIsa = true;
 43                 if (PrintRawIsa) {
 44                     _objc_inform("RAW ISA: disabling non-pointer isa because "
 45                                  "the app or a framework contains Swift code "
 46                                  "older than Swift 3.0");
 47                 }
 48                 break;
 49             }
 50         }
 51 # endif
 53 # if TARGET_OS_OSX
 54         // Disable non-pointer isa if the app is too old
 55         // (linked before OS X 10.11)
 56         if (dyld_get_program_sdk_version() < DYLD_MACOSX_VERSION_10_11) {
 57             DisableNonpointerIsa = true;
 58             if (PrintRawIsa) {
 59                 _objc_inform("RAW ISA: disabling non-pointer isa because "
 60                              "the app is too old (SDK version " SDK_FORMAT ")",
 61                              FORMAT_SDK(dyld_get_program_sdk_version()));
 62             }
 63         }
 65         // Disable non-pointer isa if the app has a __DATA,__objc_rawisa section
 66         // New apps that load old extensions may need this.
 67         for (EACH_HEADER) {
 68             if (hi->mhdr()->filetype != MH_EXECUTE) continue;
 69             unsigned long size;
 70             if (getsectiondata(hi->mhdr(), "__DATA", "__objc_rawisa", &size)) {
 71                 DisableNonpointerIsa = true;
 72                 if (PrintRawIsa) {
 73                     _objc_inform("RAW ISA: disabling non-pointer isa because "
 74                                  "the app has a __DATA,__objc_rawisa section");
 75                 }
 76             }
 77             break;  // assume only one MH_EXECUTE image
 78         }
 79 # endif
 81 #endif
 83         if (DisableTaggedPointers) {
 84             disableTaggedPointers();
 85         }
 87         initializeTaggedPointerObfuscator();
 89         if (PrintConnecting) {
 90             _objc_inform("CLASS: found %d classes during launch", totalClasses);
 91         }
 93         // namedClasses
 94         // Preoptimized classes don't go in this table.
 95         // 4/3 is NXMapTable's load factor
 96         int namedClassesSize = 
 97             (isPreoptimized() ? unoptimizedTotalClasses : totalClasses) * 4 / 3;
 98         gdb_objc_realized_classes =
 99             NXCreateMapTable(NXStrValueMapPrototype, namedClassesSize);
101         ts.log("IMAGE TIMES: first time tasks");
102     }
104     // Fix up @selector references
105     static size_t UnfixedSelectors;
106     {
107         mutex_locker_t lock(selLock);
108         for (EACH_HEADER) {
109             if (hi->hasPreoptimizedSelectors()) continue;
111             bool isBundle = hi->isBundle();
112             SEL *sels = _getObjc2SelectorRefs(hi, &count);
113             UnfixedSelectors += count;
114             for (i = 0; i < count; i++) {
115                 const char *name = sel_cname(sels[i]);
116                 SEL sel = sel_registerNameNoLock(name, isBundle);
117                 if (sels[i] != sel) {
118                     sels[i] = sel;
119                 }
120             }
121         }
122     }
124     ts.log("IMAGE TIMES: fix up selector references");
126     // Discover classes. Fix up unresolved future classes. Mark bundle classes.
127     bool hasDyldRoots = dyld_shared_cache_some_image_overridden();
129     for (EACH_HEADER) {
130         if (! mustReadClasses(hi, hasDyldRoots)) {
131             // Image is sufficiently optimized that we need not call readClass()
132             continue;
133         }
135         classref_t const *classlist = _getObjc2ClassList(hi, &count);
137         bool headerIsBundle = hi->isBundle();
138         bool headerIsPreoptimized = hi->hasPreoptimizedClasses();
140         for (i = 0; i < count; i++) {
141             Class cls = (Class)classlist[i];
142             Class newCls = readClass(cls, headerIsBundle, headerIsPreoptimized);
144             if (newCls != cls  &&  newCls) {
145                 // Class was moved but not deleted. Currently this occurs 
146                 // only when the new class resolved a future class.
147                 // Non-lazily realize the class below.
148                 resolvedFutureClasses = (Class *)
149                     realloc(resolvedFutureClasses, 
150                             (resolvedFutureClassCount+1) * sizeof(Class));
151                 resolvedFutureClasses[resolvedFutureClassCount++] = newCls;
152             }
153         }
154     }
156     ts.log("IMAGE TIMES: discover classes");
158     // Fix up remapped classes
159     // Class list and nonlazy class list remain unremapped.
160     // Class refs and super refs are remapped for message dispatching.
162     if (!noClassesRemapped()) {
163         for (EACH_HEADER) {
164             Class *classrefs = _getObjc2ClassRefs(hi, &count);
165             for (i = 0; i < count; i++) {
166                 remapClassRef(&classrefs[i]);
167             }
168             // fixme why doesn't test future1 catch the absence of this?
169             classrefs = _getObjc2SuperRefs(hi, &count);
170             for (i = 0; i < count; i++) {
171                 remapClassRef(&classrefs[i]);
172             }
173         }
174     }
176     ts.log("IMAGE TIMES: remap classes");
178 #if SUPPORT_FIXUP
179     // Fix up old objc_msgSend_fixup call sites
180     for (EACH_HEADER) {
181         message_ref_t *refs = _getObjc2MessageRefs(hi, &count);
182         if (count == 0) continue;
184         if (PrintVtables) {
185             _objc_inform("VTABLES: repairing %zu unsupported vtable dispatch "
186                          "call sites in %s", count, hi->fname());
187         }
188         for (i = 0; i < count; i++) {
189             fixupMessageRef(refs+i);
190         }
191     }
193     ts.log("IMAGE TIMES: fix up objc_msgSend_fixup");
194 #endif
196     bool cacheSupportsProtocolRoots = sharedCacheSupportsProtocolRoots();
198     // Discover protocols. Fix up protocol refs.
199     for (EACH_HEADER) {
200         extern objc_class OBJC_CLASS_$_Protocol;
201         Class cls = (Class)&OBJC_CLASS_$_Protocol;
202         ASSERT(cls);
203         NXMapTable *protocol_map = protocols();
204         bool isPreoptimized = hi->hasPreoptimizedProtocols();
206         // Skip reading protocols if this is an image from the shared cache
207         // and we support roots
208         // Note, after launch we do need to walk the protocol as the protocol
209         // in the shared cache is marked with isCanonical() and that may not
210         // be true if some non-shared cache binary was chosen as the canonical
211         // definition
212         if (launchTime && isPreoptimized && cacheSupportsProtocolRoots) {
213             if (PrintProtocols) {
214                 _objc_inform("PROTOCOLS: Skipping reading protocols in image: %s",
215                              hi->fname());
216             }
217             continue;
218         }
220         bool isBundle = hi->isBundle();
222         protocol_t * const *protolist = _getObjc2ProtocolList(hi, &count);
223         for (i = 0; i < count; i++) {
224             readProtocol(protolist[i], cls, protocol_map, 
225                          isPreoptimized, isBundle);
226         }
227     }
229     ts.log("IMAGE TIMES: discover protocols");
231     // Fix up @protocol references
232     // Preoptimized images may have the right 
233     // answer already but we don't know for sure.
234     for (EACH_HEADER) {
235         // At launch time, we know preoptimized image refs are pointing at the
236         // shared cache definition of a protocol.  We can skip the check on
237         // launch, but have to visit @protocol refs for shared cache images
238         // loaded later.
239         if (launchTime && cacheSupportsProtocolRoots && hi->isPreoptimized())
240             continue;
241         protocol_t **protolist = _getObjc2ProtocolRefs(hi, &count);
242         for (i = 0; i < count; i++) {
243             remapProtocolRef(&protolist[i]);
244         }
245     }
247     ts.log("IMAGE TIMES: fix up @protocol references");
249     // Discover categories. Only do this after the initial category
250     // attachment has been done. For categories present at startup,
251     // discovery is deferred until the first load_images call after
252     // the call to _dyld_objc_notify_register completes. rdar://problem/53119145
253     if (didInitialAttachCategories) {
254         for (EACH_HEADER) {
255             load_categories_nolock(hi);
256         }
257     }
259     ts.log("IMAGE TIMES: discover categories");
261     // Category discovery MUST BE Late to avoid potential races
262     // when other threads call the new category code before
263     // this thread finishes its fixups.
265     // +load handled by prepare_load_methods()
267     // Realize non-lazy classes (for +load methods and static instances)
268     for (EACH_HEADER) {
269         classref_t const *classlist = 
270             _getObjc2NonlazyClassList(hi, &count);
271         for (i = 0; i < count; i++) {
272             Class cls = remapClass(classlist[i]);
273             if (!cls) continue;
275             addClassTableEntry(cls);
277             if (cls->isSwiftStable()) {
278                 if (cls->swiftMetadataInitializer()) {
279                     _objc_fatal("Swift class %s with a metadata initializer "
280                                 "is not allowed to be non-lazy",
281                                 cls->nameForLogging());
282                 }
283                 // fixme also disallow relocatable classes
284                 // We can't disallow all Swift classes because of
285                 // classes like Swift.__EmptyArrayStorage
286             }
287             realizeClassWithoutSwift(cls, nil);
288         }
289     }
291     ts.log("IMAGE TIMES: realize non-lazy classes");
293     // Realize newly-resolved future classes, in case CF manipulates them
294     if (resolvedFutureClasses) {
295         for (i = 0; i < resolvedFutureClassCount; i++) {
296             Class cls = resolvedFutureClasses[i];
297             if (cls->isSwiftStable()) {
298                 _objc_fatal("Swift class is not allowed to be future");
299             }
300             realizeClassWithoutSwift(cls, nil);
301             cls->setInstancesRequireRawIsaRecursively(false/*inherited*/);
302         }
303         free(resolvedFutureClasses);
304     }
306     ts.log("IMAGE TIMES: realize future classes");
308     if (DebugNonFragileIvars) {
309         realizeAllClasses();
310     }
313     // Print preoptimization statistics
314     if (PrintPreopt) {
315         static unsigned int PreoptTotalMethodLists;
316         static unsigned int PreoptOptimizedMethodLists;
317         static unsigned int PreoptTotalClasses;
318         static unsigned int PreoptOptimizedClasses;
320         for (EACH_HEADER) {
321             if (hi->hasPreoptimizedSelectors()) {
322                 _objc_inform("PREOPTIMIZATION: honoring preoptimized selectors "
323                              "in %s", hi->fname());
324             }
325             else if (hi->info()->optimizedByDyld()) {
326                 _objc_inform("PREOPTIMIZATION: IGNORING preoptimized selectors "
327                              "in %s", hi->fname());
328             }
330             classref_t const *classlist = _getObjc2ClassList(hi, &count);
331             for (i = 0; i < count; i++) {
332                 Class cls = remapClass(classlist[i]);
333                 if (!cls) continue;
335                 PreoptTotalClasses++;
336                 if (hi->hasPreoptimizedClasses()) {
337                     PreoptOptimizedClasses++;
338                 }
340                 const method_list_t *mlist;
341                 if ((mlist = ((class_ro_t *)cls->data())->baseMethods())) {
342                     PreoptTotalMethodLists++;
343                     if (mlist->isFixedUp()) {
344                         PreoptOptimizedMethodLists++;
345                     }
346                 }
347                 if ((mlist=((class_ro_t *)cls->ISA()->data())->baseMethods())) {
348                     PreoptTotalMethodLists++;
349                     if (mlist->isFixedUp()) {
350                         PreoptOptimizedMethodLists++;
351                     }
352                 }
353             }
354         }
356         _objc_inform("PREOPTIMIZATION: %zu selector references not "
357                      "pre-optimized", UnfixedSelectors);
358         _objc_inform("PREOPTIMIZATION: %u/%u (%.3g%%) method lists pre-sorted",
359                      PreoptOptimizedMethodLists, PreoptTotalMethodLists, 
360                      PreoptTotalMethodLists
361                      ? 100.0*PreoptOptimizedMethodLists/PreoptTotalMethodLists 
362                      : 0.0);
363         _objc_inform("PREOPTIMIZATION: %u/%u (%.3g%%) classes pre-registered",
364                      PreoptOptimizedClasses, PreoptTotalClasses, 
365                      PreoptTotalClasses 
366                      ? 100.0*PreoptOptimizedClasses/PreoptTotalClasses
367                      : 0.0);
368         _objc_inform("PREOPTIMIZATION: %zu protocol references not "
369                      "pre-optimized", UnfixedProtocolReferences);
370     }
372 #undef EACH_HEADER
373 }
```



read_images 做了什么，概括：

1.  条件控制进行一次性的加载 - once
2.  修复预编译阶段的 ‘@selector’ 的混乱问题
3.  错误混乱的 类 的处理
4.  修复 重映射 一些没有被镜像文件加载进来的类
5.  修复一些消息
6.  当类里面有协议时，readProtocol
7.  修复没被加载的协议
8.  分类的处理
9.  非懒加载的类的加载处理
10.  实现新解析的未来类，以防 CF 操作它们

**下面对 read_images 详细解析**：

#### 1、fix up @selector(上面源码的 104 行)： 

selector 并非简单的字符串，而是一串 带地址的字符串.

![](https://img2020.cnblogs.com/blog/842658/202010/842658-20201018155256340-616621741.png)

#### 2、Discover classes 类的处理 (上面代码的 126 行开始)

144 行：Class was moved but not deleted - 出现类的混乱的场景比较少，代码暂时执行不进去。

对于类，一般加载内存后不会移动，相对移动来说更倾向于删除重建，移动对性能能消耗太大。

142 行代码 readClass():

![](https://img2020.cnblogs.com/blog/842658/202010/842658-20201018161340532-1161517106.png)

readClass() 做了什么？.

**readClass()** 的实现源码：



```
1 /***********************************************************************
 2 * readClass
 3 * Read a class and metaclass as written by a compiler.
 4 * Returns the new class pointer. This could be: 
 5 * - cls
 6 * - nil  (cls has a missing weak-linked superclass)
 7 * - something else (space for this class was reserved by a future class)
 8 *
 9 * Note that all work performed by this function is preflighted by 
10 * mustReadClasses(). Do not change this function without updating that one.
11 *
12 * Locking: runtimeLock acquired by map_images or objc_readClassPair
13 **********************************************************************/
14 Class readClass(Class cls, bool headerIsBundle, bool headerIsPreoptimized)
15 {
16     const char *mangledName = cls->mangledName();
18     if (missingWeakSuperclass(cls)) {
19         // No superclass (probably weak-linked). 
20         // Disavow any knowledge of this subclass.
21         if (PrintConnecting) {
22             _objc_inform("CLASS: IGNORING class '%s' with "
23                          "missing weak-linked superclass", 
24                          cls->nameForLogging());
25         }
26         addRemappedClass(cls, nil);
27         cls->superclass = nil;
28         return nil;
29     }
31     cls->fixupBackwardDeployingStableSwift();
33     Class replacing = nil;
34     if (Class newCls = popFutureNamedClass(mangledName)) {
35         // This name was previously allocated as a future class.
36         // Copy objc_class to future class's struct.
37         // Preserve future's rw data block.
39         if (newCls->isAnySwift()) {
40             _objc_fatal("Can't complete future class request for '%s' "
41                         "because the real class is too big.", 
42                         cls->nameForLogging());
43         }
45         class_rw_t *rw = newCls->data();
46         const class_ro_t *old_ro = rw->ro();
47         memcpy(newCls, cls, sizeof(objc_class));
48         rw->set_ro((class_ro_t *)newCls->data());
49         newCls->setData(rw);
50         freeIfMutable((char *)old_ro->name);
51         free((void *)old_ro);
53         addRemappedClass(cls, newCls);
55         replacing = cls;
56         cls = newCls;
57     }
59     if (headerIsPreoptimized  &&  !replacing) {
60         // class list built in shared cache
61         // fixme strict assert doesn't work because of duplicates
62         // ASSERT(cls == getClass(name));
63         ASSERT(getClassExceptSomeSwift(mangledName));
64     } else {
65         addNamedClass(cls, mangledName, replacing);
66         addClassTableEntry(cls);
67     }
69     // for future reference: shared cache never contains MH_BUNDLEs
70     if (headerIsBundle) {
71         cls->data()->flags |= RO_FROM_BUNDLE;
72         cls->ISA()->data()->flags |= RO_FROM_BUNDLE;
73     }
75     return cls;
76 }
```



这里因系统类较多，不好调试，我们添加自己的类进行探究:

![](https://img2020.cnblogs.com/blog/842658/202010/842658-20201018164041713-480698231.png) 

 继续断点调试：上面代码 34 行的并未进去！

走进了 65 行，即下图 3247 行：

![](https://img2020.cnblogs.com/blog/842658/202010/842658-20201018164837388-1017352855.png)

**2.1）addNamedClass()**：--> name 加到 cls 里面



```
1 /***********************************************************************
 2 * addNamedClass
 3 * Adds name => cls to the named non-meta class map.// 将 name=>cls 添加到费元类的映射
 4 * Warns about duplicate class names and keeps the old mapping.// 重复的类名将保留旧的映射
 5 * Locking: runtimeLock must be held by the caller
 6 **********************************************************************/
 7 static void addNamedClass(Class cls, const char *name, Class replacing = nil)
 8 {
 9     runtimeLock.assertLocked();
10     Class old;
11     if ((old = getClassExceptSomeSwift(name))  &&  old != replacing) {
12         inform_duplicate(name, old, cls);
14         // getMaybeUnrealizedNonMetaClass uses name lookups.
15         // Classes not found by name lookup must be in the
16         // secondary meta->nonmeta table.
17         addNonMetaClass(cls);
18     } else {
19         NXMapInsert(gdb_objc_realized_classes, name, cls);// 插入
20     }
21     ASSERT(!(cls->data()->flags & RO_META));
23     // wrong: constructed classes are already realized when they get here
24     // ASSERT(!cls->isRealized());
25 }
```



类名来自于 mangleName：--> 已经初始化 or 未来类则从 ro 里面读，否则从 macho 的 data 里面读：

![](https://img2020.cnblogs.com/blog/842658/202010/842658-20201018170052986-525652926.png)

**2.2）addClassTableEntry(cls)** 



```
1 /***********************************************************************
 2 * addClassTableEntry
 3 * Add a class to the table of all classes. If addMeta is true,
 4 * automatically adds the metaclass of the class as well.
 5 * Locking: runtimeLock must be held by the caller.
 6 **********************************************************************/
 7 static void
 8 addClassTableEntry(Class cls, bool addMeta = true)
 9 {
10     runtimeLock.assertLocked();
12     // This class is allowed to be a known class via the shared cache or via
13     // data segments, but it is not allowed to be in the dynamic table already.
14     auto &set = objc::allocatedClasses.get();// 初始化位置在 runtime_init()
16     ASSERT(set.find(cls) == set.end());
18     if (!isKnownClass(cls))
19         set.insert(cls);// 添加 insert
20     if (addMeta)
21         addClassTableEntry(cls->ISA(), false);// 元类就添加到元类
22 }
```



**readClass 做了：将 cls 从 macho 的格式里面 读取添加到内存里面**。

篇幅较长，接下篇：[OC 底层探索 14、类的加载 2]()
