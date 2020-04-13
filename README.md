# How does Chromium start?

This is my own notes of me learning how [chromium](https://github.com/chromium/chromium) starts running its program, by exploring the git repository. As of 2020, Chrome is the most popular used browser out of all, so it would be quite interesting to see how the internals work.

It's [basically this](https://chromium.googlesource.com/chromium/src/+/master/docs/design/startup.md), except in more detail.

Please correct me if I get it wrong somewhere throughout this document (I still need to learn more about C and C++).

## Cross platform entry points

As chromium runs on various platforms, there are entry points for each type of operating system there are. For Windows and Mac, there are wrapper functions within the entry point files.

| OS          | File                                 | Entry-level function                                                       |
| ----------- | ------------------------------------ | -------------------------------------------------------------------------- |
| Windows     | `chrome/app/chrome_exe_main_win.cc`  | `int APIENTRY wWinMain(HINSTANCE instance, HINSTANCE prev, wchar_t*, int)` |
| Mac         | `chrome/app/chrome_exe_main_mac.cc`  | `int main(int argc, char* argv[])`                                         |
| Linux/other | `chrome/app/chrome_exe_main_aura.cc` | `int main(int argc, const char** argv)`                                    |

In all these files, they all invoke the function ChromeMain.

### Windows platform

#### `chrome/app/chrome_exe_main_win.cc`

```c++
#if !defined(WIN_CONSOLE_APP)
int APIENTRY wWinMain(HINSTANCE instance, HINSTANCE prev, wchar_t*, int) {
#else
int main() {
  HINSTANCE instance = GetModuleHandle(nullptr);
#endif
  // ...
  const base::TimeTicks exe_entry_point_ticks = base::TimeTicks::Now();
  // ...
  MainDllLoader* loader = MakeMainDllLoader();
  int rc = loader->Launch(instance, exe_entry_point_ticks);
  // ...
}

```

#### `chrome/app/main_dll_loader_win.cc`

```c++
int MainDllLoader::Launch(HINSTANCE instance,
                          base::TimeTicks exe_entry_point_ticks) {
  // ...
  DLL_MAIN chrome_main =
  reinterpret_cast<DLL_MAIN>(::GetProcAddress(dll_, "ChromeMain"));
  int rc = chrome_main(instance, &sandbox_info,
                       exe_entry_point_ticks.ToInternalValue());
  // ...
}
```

### Mac platform

#### `chrome/app/chrome_exe_main_mac.cc`

```c++
__attribute__((visibility("default"))) int main(int argc, char* argv[]) {
  // ...
  void* library =
    dlopen(framework_path.get(), RTLD_LAZY | RTLD_LOCAL | RTLD_FIRST);
  if (!library) {
    FatalError("dlopen %s: %s.", framework_path.get(), dlerror());
  }

  const ChromeMainPtr chrome_main =
    reinterpret_cast<ChromeMainPtr>(dlsym(library, "ChromeMain"));
  if (!chrome_main) {
    FatalError("dlsym ChromeMain: %s.", dlerror());
  }
  rv = chrome_main(argc, argv);
  // ...
}
```

### Linux platform

#### `chrome/app/chrome_exe_main_aura.cc`

```c++
extern "C" {
int ChromeMain(int argc, const char** argv);
}

int main(int argc, const char** argv) {
  return ChromeMain(argc, argv);
}
```

## Function ChromeMain

#### `chrome/app/chrome_main.cc`

```c++
#if defined(OS_WIN)
#include "base/debug/dump_without_crashing.h"
#include "base/win/win_util.h"
#include "chrome/chrome_elf/chrome_elf_main.h"
#include "chrome/common/chrome_constants.h"
#include "chrome/install_static/initialize_from_primary_module.h"
#include "chrome/install_static/install_details.h"

#define DLLEXPORT __declspec(dllexport)

// We use extern C for the prototype DLLEXPORT to avoid C++ name mangling.
extern "C" {
DLLEXPORT int __cdecl ChromeMain(HINSTANCE instance,
                                 sandbox::SandboxInterfaceInfo* sandbox_info,
                                 int64_t exe_entry_point_ticks);
}
#elif defined(OS_POSIX)
extern "C" {
__attribute__((visibility("default")))
int ChromeMain(int argc, const char** argv);
}
#endif
```

The arguments provided by the command line are passed to this function. As we are still in the early initialisation process, `ChromeMain` function still has to deal on a platform-by-platform case basis. This is done by using control preprocessor directives such as `#if defined(OS_WIN)` and `#elif defined(OS_POSIX)`.

```c++
#if defined(OS_WIN)
DLLEXPORT int __cdecl ChromeMain(HINSTANCE instance,
                                 sandbox::SandboxInterfaceInfo* sandbox_info,
                                 int64_t exe_entry_point_ticks) {
#elif defined(OS_POSIX)
int ChromeMain(int argc, const char** argv) {
  int64_t exe_entry_point_ticks = 0;
#endif
  // ...
  ChromeMainDelegate chrome_main_delegate(
      base::TimeTicks::FromInternalValue(exe_entry_point_ticks));
  content::ContentMainParams params(&chrome_main_delegate);
  // populate params variable with parameters...
  int rv = content::ContentMain(params);
  return rv;
}
```

## Function ContentMain

#### `content/app/content_main.cc`

```c++
int ContentMain(const ContentMainParams& params) {

}
```

-   Uses a scoped pointer (defined in `base/memory/scoped_ptr.h`) to store the return value of `BrowserMainRunner::Create`.
-   Initialises browser by calling `BrowserMainRunnerImpl::Initialize` method.
-   Runs browser by calling `BrowserMainRunnerImpl::Run` method.
-   Once the exit value is returned, `BrowserMainRunnerImpl::Shutdown` is called to shut down the browser, and this exit value is then returned by `ContentMain` function itself.

## Class BrowserMainRunnerImpl

`content/browser/browser_main_runner.cc`

Note: `BrowserMainRunnerImpl` is a derived class of `BrowserMainRunner`. The methods `BrowserMainRunnerImpl::Initialize`, `BrowserMainRunnerImpl::Run` and `BrowserMainRunnerImpl::Shutdown` are virtual methods with the `OVERRIDE` keyword at the end of the method declaration line.

`BrowserMainRunner* BrowserMainRunner::Create()` creates a new `BrowserMainRunnerImpl` instance.

Once instantiated, the following methods perform the following actions:

-   `BrowserMainRunnerImpl::Initialize` initialises `BrowserMainLoop` and stores it in the `main_loop` variable.
-   `BrowserMainRunnerImpl::Run` runs the browser loop via the call `main_loop_->RunMainMessageLoopParts()`
-   `BrowserMainRunnerImpl::Shutdown` calls `main_loop_->ShutdownThreadsAndCleanUp()`, and deassigns pointers to various variables by calling their `reset` methods with `NULL` as the parameter.

## Class BrowserMainLoop

File: `content/browser/browser_main_loop.cc`

Method: `void BrowserMainLoop::Init()`

Statement: `parts_.reset(GetContentClient()->browser()->CreateBrowserMainParts(parameters_));`

Definition of `parts_`: `#include "content/browser/browser_main_loop.h"` `scoped_ptr<BrowserMainParts> parts_;`

Method: `void BrowserMainLoop::RunMainMessageLoopParts()`

Statement: `parts_->MainMessageLoopRun(&result_code_)`

Description:
A boolean value `ran_main_loop` is used to keep check if the main loop has finished running. The method creates a loop by calling the method itself, as long as `ran_main_loop` variable is set to false.

Method: `void BrowserMainLoop::MainMessageLoopRun()`

Statements: `base::RunLoop run_loop; run_loop.Run();`

Description:
[wip]

(skip to section "Class RunLoop" if you are not interested in the `parts_` variable)

## Class ContentClient

File: `content/public/common/content_client.cc`

In the respective header file, within the class `ContentClient` the method `browser()` is defined.

`ContentBrowserClient* browser() { return browser_; }`

## Class BrowserMainParts

File: `content/public/browser/browser_main_parts.cc`

Method: `bool BrowserMainParts::MainMessageLoopRun(int* result_code)`

Statement: `return false;`

Description:
This method simply returns false. Assuming this is the default implementation of a browser main part, it can be overridden.

## Class RunLoop

File: `base/run_loop.cc`

Method: `void RunLoop::Run()`

Method description:
[wip]

Statement: `loop_->RunHandler();`
Definition of `loop_` in class: `loop_(MessageLoop::current())`

Statement description:
[wip]

## Class MessageLoop

File: `base/message_loop/message_loop.cc`

Platform specific include relevant to discussion:

    #include "base/message_loop/message_pump_default.h"
    // ...
    #if defined(OS_MACOSX)
    #include "base/message_loop/message_pump_mac.h"
    #endif
    #if defined(OS_POSIX) && !defined(OS_IOS)
    #include "base/message_loop/message_pump_libevent.h"
    #endif
    #if defined(OS_ANDROID)
    #include "base/message_loop/message_pump_android.h"
    #endif
    #if defined(USE_GLIB)
    #include "base/message_loop/message_pump_glib.h"
    #endif

Method: `void MessageLoop::RunHandler()`

Statement: `pump_->Run(this);`

Description:
The run method passes an instance of `MessageLoop` itself, which is used to call other methods within the same class in the `pump_` instance of the `MessagePump` class.

Definition (in respective header file): `scoped_ptr<MessagePump> pump_;`

## Class MessagePump

File: `base/message_loop/message_pump*.cc`

This class is platform specific. Perhaps the reason for this is that `MessagePump` isolates platform specific code from `MessageLoop`, even though these two classes are quite dependant on each other.

Method: `void MessagePumpDefault::Run(Delegate* delegate)` (from `message_pump_default.cc`)

We're heading towards code that deals with the execution of a single iteration of a loop.

    bool did_work = delegate->DoWork();
    if (!keep_running_)
      break;

    did_work |= delegate->DoDelayedWork(&delayed_work_time_);
    if (!keep_running_)
      break;

    if (did_work)
      continue;

    did_work = delegate->DoIdleWork();
    if (!keep_running_)
      break;

    if (did_work)
      continue;

## Class MessageLoop (again)

Method: `bool MessageLoop::DoWork()`

Method: `void MessageLoop::RunTask(const PendingTask& pending_task)`

## Obligatory License

    // Copyright 2015 The Chromium Authors. All rights reserved.
    //
    // Redistribution and use in source and binary forms, with or without
    // modification, are permitted provided that the following conditions are
    // met:
    //
    //    * Redistributions of source code must retain the above copyright
    // notice, this list of conditions and the following disclaimer.
    //    * Redistributions in binary form must reproduce the above
    // copyright notice, this list of conditions and the following disclaimer
    // in the documentation and/or other materials provided with the
    // distribution.
    //    * Neither the name of Google Inc. nor the names of its
    // contributors may be used to endorse or promote products derived from
    // this software without specific prior written permission.
    //
    // THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
    // "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
    // LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
    // A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
    // OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
    // SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
    // LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
    // DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
    // THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
    // (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
    // OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
