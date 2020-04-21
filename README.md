# How does Chromium start?

This is my own notes of me learning how [chromium](https://github.com/chromium/chromium) starts running its program, by exploring the git repository. As of 2020, Chrome is the most popular used browser out of all, so it would be quite interesting to see how the internals work.

It's [basically this](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/docs/design/startup.md), except in more detail.

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
  #if defined(OS_LINUX) || defined(OS_MACOSX) || defined(OS_WIN)
  if (command_line->HasSwitch(switches::kHeadless)) {
    return headless::HeadlessShellMain(params);
  }
  #endif  // defined(OS_LINUX) || defined(OS_MACOSX) || defined(OS_WIN)

  int rv = content::ContentMain(params);

  return rv;
}
```

We look at the implementation of `ChromeMain`, and we see that after constructing `ChromeMainDelegate`, it invokes `content::ContentMain()` (if Chromium was run with `--headless` flag set, then it would instead call `headless::HeadlessShellMain()`). The arguments provided by the command line are passed to this function.

As we are still in the early initialisation process, `ChromeMain` function still has to deal on a platform-by-platform case basis. This is done by using control preprocessor directives such as `#if defined(OS_WIN)` and `#elif defined(OS_POSIX)`. These flags are defined in file [`build/build_config.h`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/build/build_config.h). We have a look at how `ChromeMain` works on a platform-by-platform case basis.

-   We see that for the Windows platform, the arguments are `HINSTANCE instance`, `sandbox::SandboxInterfaceInfo* sandbox_info` and `int64_t exe_entry_point_ticks`.
-   For POSIX platforms, the arguments are simply `int argc` and `const char** argv`.

## Function `ChromeMainDelegate::OverrideProcessType`

#### Year 2012: [`chrome/app/chrome_main_delegate.cc`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/chrome/app/chrome_main_delegate.cc)

```c++
service_manager::ProcessType ChromeMainDelegate::OverrideProcessType() {
  const auto& command_line = *base::CommandLine::ForCurrentProcess();
  if (command_line.GetSwitchValueASCII(switches::kProcessType) ==
      service_manager::switches::kProcessTypeService) {
    // Don't mess with embedded service command lines.
    return service_manager::ProcessType::kDefault;
  }
  return service_manager::ProcessType::kDefault;
}
```

Why we are seeing this function out of no where? Well, note that `ChromeMainDelegate` was constructed in `ChromeMain`. The function `ChromeMainDelegate::OverrrideProcessType()` will make sense once we see `service_manager::Main()` function (down below). For now, all we need to know is that it returns `ProcessType::kDefault`.

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
  DCHECK(delegate);
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

Since the line `delegate->OverrideProcessType()` returns `ProcessType::kDefault` (see `ChromeMainDelegate::OverrideProcessType` up above), it checks to see which process type it should set, based on the command line. Lets assume that no switches have been set. This means that it will select process type `ProcessType::kEmbedder`, and will execute the line `exit_code = delegate->RunEmbedderProcess();`.

Notes:

-   There are three process types: `ProcessType::kServiceManager`, `ProcessType::kService` and `ProcessType::kEmbedder`. `ProcessType::kDefault` is not really a process type, and we can see there is a `NOTREACHED()` macro in the switch case
-   The `DCHECK(delegate)` macro is an assertion that will fail if `delegate` value has not been populated (see [this link](https://chromium.googlesource.com/chromium/src/+/master/styleguide/c++/c++.md#check_dcheck_and-notreached)).
-   The `NOTREACHED()` macro is an assertion that will always fail, which happens when the `ProcessType::kDefault` case has been reached
-   Service manager first checks to see if the process type has been overrided. If it hasn't, it checks the command line for a process type switch, and updates the process type accordingly.
-   `switches::kProcessType` is equal to `"type"`
-   `*base::CommandLine::ForCurrentProcess()` is a getter method

## Class `ContentServiceManagerMainDelegate`

#### Year 2017: [`content/app/content_service_manager_main_delegate.cc`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/content/app/content_service_manager_main_delegate.cc)

```c++
namespace content {

ContentServiceManagerMainDelegate::ContentServiceManagerMainDelegate(
    const ContentMainParams& params)
    : content_main_params_(params),
      content_main_runner_(ContentMainRunnerImpl::Create()) {}

// ...

int ContentServiceManagerMainDelegate::RunEmbedderProcess() {
  return content_main_runner_->Run(start_service_manager_only_);
}
}
```

In the initialiser list within the constructor, we see that `ContentMainRunnerImpl::Create()` (which is a factory pattern) is executed, and gets stored in variable `content_main_runner_`. In the `RunEmbedderProcess()` function, we see that the run method from `content_main_runner_` is invoked.

## Class `ContentMainRunnerImpl`

#### Year 2012: [`content/app/content_main_runner_impl.cc`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/content/app/content_main_runner_impl.cc)

```c++
int ContentMainRunnerImpl::Run(bool start_service_manager_only) {
  DCHECK(is_initialized_);
  DCHECK(!is_shutdown_);
  const base::CommandLine& command_line =
      *base::CommandLine::ForCurrentProcess();
  std::string process_type =
      command_line.GetSwitchValueASCII(switches::kProcessType);

#if !defined(CHROME_MULTIPLE_DLL_BROWSER)
  // Run this logic on all child processes. Zygotes will run this at a later
  // point in time when the command line has been updated.
  if (!process_type.empty() &&
      process_type != service_manager::switches::kZygoteProcess) {
    InitializeFieldTrialAndFeatureList();
    delegate_->PostFieldTrialInitialization();
  }
#endif

  MainFunctionParams main_params(command_line);
  main_params.ui_task = ui_task_;
  main_params.created_main_parts_closure = created_main_parts_closure_;
#if defined(OS_WIN)
  main_params.sandbox_info = &sandbox_info_;
#elif defined(OS_MACOSX)
  main_params.autorelease_pool = autorelease_pool_;
#endif

  RegisterMainThreadFactories();

#if !defined(CHROME_MULTIPLE_DLL_CHILD)
  if (process_type.empty())
    return RunServiceManager(main_params, start_service_manager_only);
#endif  // !defined(CHROME_MULTIPLE_DLL_CHILD)

  return RunOtherNamedProcessTypeMain(process_type, main_params, delegate_);
}
```

## Function `content::BrowserMain`

#### Year 2012: [`content/browser/browser_main.cc`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/content/browser/browser_main.cc)

```c++
namespace content {
// ...
// Main routine for running as the Browser process.
int BrowserMain(const MainFunctionParams& parameters) {
  ScopedBrowserMainEvent scoped_browser_main_event;

  base::trace_event::TraceLog::GetInstance()->set_process_name("Browser");
  base::trace_event::TraceLog::GetInstance()->SetProcessSortIndex(
      kTraceEventBrowserProcessSortIndex);

  std::unique_ptr<BrowserMainRunnerImpl> main_runner(
      BrowserMainRunnerImpl::Create());

  int exit_code = main_runner->Initialize(parameters);
  if (exit_code >= 0)
    return exit_code;

  exit_code = main_runner->Run();

  main_runner->Shutdown();

  return exit_code;
}

}  // namespace content
```

## Class `BrowserMainRunnerImpl`

#### Year 2012: [`content/browser/browser_main_runner_impl.cc`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/content/browser/browser_main_runner_impl.cc)

```c++
int BrowserMainRunnerImpl::Initialize(const MainFunctionParams& parameters) {
  SCOPED_UMA_HISTOGRAM_LONG_TIMER(
      "Startup.BrowserMainRunnerImplInitializeLongTime");
  TRACE_EVENT0("startup", "BrowserMainRunnerImpl::Initialize");

  // On Android we normally initialize the browser in a series of UI thread
  // tasks. While this is happening a second request can come from the OS or
  // another application to start the browser. If this happens then we must
  // not run these parts of initialization twice.
  if (!initialization_started_) {
    initialization_started_ = true;

    const base::TimeTicks start_time_step1 = base::TimeTicks::Now();

    SkGraphics::Init();

    // ...
    notification_service_.reset(new NotificationServiceImpl);

#if defined(OS_WIN)
    // Ole must be initialized before starting message pump, so that TSF
    // (Text Services Framework) module can interact with the message pump
    // on Windows 8 Metro mode.
    ole_initializer_.reset(new ui::ScopedOleInitializer);
#endif  // OS_WIN

    gfx::InitializeFonts();

    main_loop_.reset(
        new BrowserMainLoop(parameters, std::move(scoped_execution_fence_)));

    main_loop_->Init();

    if (parameters.created_main_parts_closure) {
      std::move(*parameters.created_main_parts_closure)
          .Run(main_loop_->parts());
      delete parameters.created_main_parts_closure;
    }

    const int early_init_error_code = main_loop_->EarlyInitialization();
    if (early_init_error_code > 0)
      return early_init_error_code;

    // Must happen before we try to use a message loop or display any UI.
    if (!main_loop_->InitializeToolkit())
      return 1;

    main_loop_->PreMainMessageLoopStart();
    main_loop_->MainMessageLoopStart();
    main_loop_->PostMainMessageLoopStart();

    // WARNING: If we get a WM_ENDSESSION, objects created on the stack here
    // are NOT deleted. If you need something to run during WM_ENDSESSION add it
    // to browser_shutdown::Shutdown or BrowserProcess::EndSession.

    ui::InitializeInputMethod();
    UMA_HISTOGRAM_TIMES("Startup.BrowserMainRunnerImplInitializeStep1Time",
                        base::TimeTicks::Now() - start_time_step1);
  }
  const base::TimeTicks start_time_step2 = base::TimeTicks::Now();
  main_loop_->CreateStartupTasks();
  int result_code = main_loop_->GetResultCode();
  if (result_code > 0)
    return result_code;

  UMA_HISTOGRAM_TIMES("Startup.BrowserMainRunnerImplInitializeStep2Time",
                      base::TimeTicks::Now() - start_time_step2);

  // Return -1 to indicate no early termination.
  return -1;
}

int BrowserMainRunnerImpl::Run() {
  DCHECK(initialization_started_);
  DCHECK(!is_shutdown_);
  main_loop_->RunMainMessageLoopParts();
  return main_loop_->GetResultCode();
}
```

`std::unique_ptr<BrowserMainLoop> main_loop_;`

## Class `BrowserMainLoop`

#### Year 2012: [`content/browser/browser_main_loop.cc`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/content/browser/browser_main_loop.cc)

```c++
bool BrowserMainLoop::InitializeToolkit() {
  TRACE_EVENT0("startup", "BrowserMainLoop::InitializeToolkit");

#if defined(OS_WIN)
  INITCOMMONCONTROLSEX config;
  config.dwSize = sizeof(config);
  config.dwICC = ICC_WIN95_CLASSES;
  if (!InitCommonControlsEx(&config))
    PLOG(FATAL);
#endif

#if defined(USE_AURA)

#if defined(USE_X11)
  if (!parsed_command_line_.HasSwitch(switches::kHeadless) &&
      !gfx::GetXDisplay()) {
    LOG(ERROR) << "Unable to open X display.";
    return false;
  }
#endif

  // Env creates the compositor. Aura widgets need the compositor to be created
  // before they can be initialized by the browser.
  env_ = aura::Env::CreateInstance();
#endif  // defined(USE_AURA)

  if (parts_)
    parts_->ToolkitInitialized();

  return true;
}

void BrowserMainLoop::PreMainMessageLoopStart() {
  TRACE_EVENT0("startup",
               "BrowserMainLoop::MainMessageLoopStart:PreMainMessageLoopStart");
  if (parts_) {
    parts_->PreMainMessageLoopStart();
  }
}

void BrowserMainLoop::MainMessageLoopStart() {
  // DO NOT add more code here. Use PreMainMessageLoopStart() above or
  // PostMainMessageLoopStart() below.

  TRACE_EVENT0("startup", "BrowserMainLoop::MainMessageLoopStart");
  DCHECK(base::MessageLoopCurrentForUI::IsSet());
  InitializeMainThread();
}

void BrowserMainLoop::PostMainMessageLoopStart() {
  // ...
}

void BrowserMainLoop::RunMainMessageLoopParts() {
  // Don't use the TRACE_EVENT0 macro because the tracing infrastructure doesn't
  // expect synchronous events around the main loop of a thread.
  TRACE_EVENT_ASYNC_BEGIN0("toplevel", "BrowserMain:MESSAGE_LOOP", this);

  bool ran_main_loop = false;
  if (parts_)
    ran_main_loop = parts_->MainMessageLoopRun(&result_code_);

  if (!ran_main_loop)
    MainMessageLoopRun();

  TRACE_EVENT_ASYNC_END0("toplevel", "BrowserMain:MESSAGE_LOOP", this);
}

// ...

void BrowserMainLoop::MainMessageLoopRun() {
#if defined(OS_ANDROID)
  // Android's main message loop is the Java message loop.
  NOTREACHED();
#else
  base::RunLoop run_loop;
  parts_->PreDefaultMainMessageLoopRun(run_loop.QuitClosure());
  run_loop.Run();
#endif
}
```

Notes:

-   If you run `chromium-browser` in a TTY terminal (that is, no GUI), it will output `"Unable to open X display."` error message.

## Class `RunLoop`

#### Year 2012: [`base/run_loop.cc`](https://chromium.googlesource.com/chromium/src/+/4900686dee9aacdb5ac0a203acbef587c292e6fe/base/run_loop.cc)

```c++
namespace base {

namespace {

ThreadLocalPointer<RunLoop::Delegate>& GetTlsDelegate() {
  static base::NoDestructor<ThreadLocalPointer<RunLoop::Delegate>> instance;
  return *instance;
}

// ...
}  // namespace
// ...

RunLoop::RunLoop(Type type)
    : delegate_(GetTlsDelegate().Get()),
      type_(type),
      origin_task_runner_(ThreadTaskRunnerHandle::Get()) {
  DCHECK(delegate_) << "A RunLoop::Delegate must be bound to this thread prior "
                       "to using RunLoop.";
  DCHECK(origin_task_runner_);
}

// ...

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
// ...
}  // namespace base
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
