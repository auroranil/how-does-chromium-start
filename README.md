# How does Chromium start?

This is my own notes of me learning how [chromium](https://github.com/chromium/chromium) starts running its program, by exploring the git repository. As of 2020, Chrome is the most popular used browser out of all, so it would be quite interesting to see how the internals work.

It's [basically this](https://chromium.googlesource.com/chromium/src/+/master/docs/design/startup.md), except in more detail.

Please correct me if I get it wrong somewhere throughout this document (I still need to learn more about C and C++). I have checkout commit `4900686dee9aacdb5ac0a203acbef587c292e6fe`, which was pushed on Sunday 12th of April, 2020, at 08:20:43. All links to source files are based on this commit. All years specified before the filename refer to the year specified in the copyright comment at the beginning of each source file.

## Cross platform entry points

As chromium runs on various platforms, there are entry points for each type of operating system there are. For Windows and Mac, there are wrapper functions within the entry point files.

| OS          | File                                                                                                                                                                 | Entry-level function                                                       |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Windows     | [`chrome/app/chrome_exe_main_win.cc`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/chrome/app/chrome_exe_main_win.cc)   | `int APIENTRY wWinMain(HINSTANCE instance, HINSTANCE prev, wchar_t*, int)` |
| Mac         | [`chrome/app/chrome_exe_main_mac.cc`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/chrome/app/chrome_exe_main_mac.cc)   | `int main(int argc, char* argv[])`                                         |
| Linux/other | [`chrome/app/chrome_exe_main_aura.cc`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/chrome/app/chrome_exe_main_aura.cc) | `int main(int argc, const char** argv)`                                    |

In all these files, they all invoke the function ChromeMain.

### Windows platform

#### Year 2011: [`chrome/app/chrome_exe_main_win.cc`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/chrome/app/chrome_exe_main_win.cc)

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

Chromium uses [`wWinMain`](https://docs.microsoft.com/en-us/windows/win32/learnwin32/winmain--the-application-entry-point) as the application entry point for Windows GUI applications.

#### Year 2012: [`chrome/app/main_dll_loader_win.cc`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/chrome/app/main_dll_loader_win.cc)

```c++
namespace {
typedef int (*DLL_MAIN)(HINSTANCE, sandbox::SandboxInterfaceInfo*, int64_t);
}
int MainDllLoader::Launch(HINSTANCE instance,
                          base::TimeTicks exe_entry_point_ticks) {
  // ...
  base::FilePath file;
  dll_ = Load(&file);
  if (!dll_)
    return chrome::RESULT_CODE_MISSING_DATA;
  // ...
  DLL_MAIN chrome_main =
  reinterpret_cast<DLL_MAIN>(::GetProcAddress(dll_, "ChromeMain"));
  int rc = chrome_main(instance, &sandbox_info,
                       exe_entry_point_ticks.ToInternalValue());
  // ...
}
```

Notes:

-   [GetProcAddress](https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getprocaddress) is used to retrieve `ChromeMain` function from a [Dynamic Link Library (DLL)](https://en.wikipedia.org/wiki/Dynamic-link_library), and reinterprets it as `DLL_MAIN`, which is defined in an [unnamed namespace](https://en.cppreference.com/w/cpp/language/namespace#Unnamed_namespaces) as a [function pointer](https://en.wikipedia.org/wiki/Function_pointer).

### Mac platform

#### Year 2015: [`chrome/app/chrome_exe_main_mac.cc`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/chrome/app/chrome_exe_main_mac.cc)

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

#### Year 2011: [`chrome/app/chrome_exe_main_aura.cc`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/chrome/app/chrome_exe_main_aura.cc)

```c++
int main(int argc, const char** argv) {
  return ChromeMain(argc, argv);
}
```

## Function `ChromeMain`

#### Year 2012: [`chrome/app/chrome_main.cc`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/chrome/app/chrome_main.cc)

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

We look at the implementation of `ChromeMain`, and we see that it invokes `content::ContentMain()`. The arguments provided by the command line are passed to this function.

As we are still in the early initialisation process, `ChromeMain` function still has to deal on a platform-by-platform case basis. This is done by using control preprocessor directives such as `#if defined(OS_WIN)` and `#elif defined(OS_POSIX)`. These flags are defined in file [`build/build_config.h`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/build/build_config.h). We have a look at how `ChromeMain` works on a platform-by-platform case basis.

-   We see that for the Windows platform, the arguments are `HINSTANCE instance`, `sandbox::SandboxInterfaceInfo* sandbox_info` and `int64_t exe_entry_point_ticks`.
-   For POSIX platforms, the arguments are simply `int argc` and `const char** argv`.

## Function `content::ContentMain`

#### Year 2012: [`content/app/content_main.cc`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/content/app/content_main.cc)

```c++
int ContentMain(const ContentMainParams& params) {
  ContentServiceManagerMainDelegate delegate(params);
  service_manager::MainParams main_params(&delegate);
#if !defined(OS_WIN) && !defined(OS_ANDROID)
  main_params.argc = params.argc;
  main_params.argv = params.argv;
#endif
  return service_manager::Main(main_params);
}
```

## Function `service_manager::Main`

#### Year 2017: [`services/service_manager/embedder/main.cc`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/services/service_manager/embedder/main.cc)

```c++
int Main(const MainParams& params) {
  MainDelegate* delegate = params.delegate;
  // ...
  int exit_code = -1;
  // ...
  ProcessType process_type = delegate->OverrideProcessType();
  // ...
  const auto& command_line = *base::CommandLine::ForCurrentProcess();
  if (process_type == ProcessType::kDefault) {
    std::string type_switch =
        command_line.GetSwitchValueASCII(switches::kProcessType);
    if (type_switch == switches::kProcessTypeServiceManager) {
      process_type = ProcessType::kServiceManager;
    } else if (type_switch == switches::kProcessTypeService) {
      process_type = ProcessType::kService;
    } else {
      process_type = ProcessType::kEmbedder;
    }
  }
  switch (process_type) {
    case ProcessType::kDefault:
      NOTREACHED();
      break;

    case ProcessType::kServiceManager:
      exit_code = RunServiceManager(delegate);
      break;

    case ProcessType::kService:
      CommonSubprocessInit();
      exit_code = RunService(delegate);
      break;

    case ProcessType::kEmbedder:
      if (delegate->IsEmbedderSubprocess())
        CommonSubprocessInit();
      exit_code = delegate->RunEmbedderProcess();
      break;
  }
  // ...
  return exit_code;
}
```

Notes:

-   There are three process types: `ProcessType::kServiceManager`, `ProcessType::kService` and `ProcessType::kEmbedder`. `ProcessType::kDefault` is not really a process type, and we can see there is a `NOTREACHED()` macro in the switch case
-   The `NOTREACHED()` macro is an assertion that will always fail, which happens when the `ProcessType::kDefault` case has been reached (see [this link](https://chromium.googlesource.com/chromium/src/+/master/styleguide/c++/c++.md#check_dcheck_and-notreached)).
-   Service manager first checks to see if the process type has been overrided. If it hasn't, it checks the command line for a process type switch, and updates the process type accordingly.
-   `switches::kProcessType` is equal to `"type"`
-   `*base::CommandLine::ForCurrentProcess()` is a getter method

## Function `RunLoop::Run`

#### Year 2012: [`base/run_loop.cc`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/base/run_loop.cc)

```c++
void RunLoop::Run() {
  // ...

  if (!BeforeRun())
    return;

  // ...

  const bool application_tasks_allowed =
      delegate_->active_run_loops_.size() == 1U ||
      type_ == Type::kNestableTasksAllowed;
  delegate_->Run(application_tasks_allowed, TimeDelta::Max());

  AfterRun();
}
```

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
