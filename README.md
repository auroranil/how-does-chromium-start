# How does Chromium start?
Just me learning how chromium starts running its program by exploring the git repository. Please correct me if I get it wrong somewhere throughout this document (I still need to learn more about C and C++).

## Cross platform entry points
As chromium runs on various platforms, there are entry points for each type of operating system there are. For Windows and Mac, there are wrapper functions within the entry point files.

Windows: `chrome/app/chrome_exe_main_win.cc` `int APIENTRY wWinMain(HINSTANCE instance, HINSTANCE prev, wchar_t*, int)`

Mac: `chrome/app/chrome_exe_main_mac.cc` `int main(int argc, char* argv[])`

Linux/other: `chrome/app/chrome_exe_main_aura.cc` `int main(int argc, const char** argv)`

## "ChromeMain" function
`chrome/app/chrome_main.cc`

Windows: `DLLEXPORT int __cdecl ChromeMain(HINSTANCE instance, sandbox::SandboxInterfaceInfo* sandbox_info)`

POSIX: `int ChromeMain(int argc, const char** argv)`

Arguments are delegated into the `params` variable. As we are still in the early initialisation process, `ChromeMain` function still has to deal on a platform-by-platform case basis. 
## "ContentMain" function
`content/app/content_main.cc` `int ContentMain(const ContentMainParams& params)`
* Uses a scoped pointer (defined in `base/memory/scoped_ptr.h`) to store the return value of `BrowserMainRunner::Create`.
* Initialises browser by calling `BrowserMainRunnerImpl::Initialize` method.
* Runs browser by calling `BrowserMainRunnerImpl::Run` method.
* Once the exit value is returned, `BrowserMainRunnerImpl::Shutdown` is called to shut down the browser, and this exit value is then returned by `ContentMain` function itself.

## "BrowserMainRunnerImpl" class
`content/browser/browser_main_runner.cc` 

Note: `BrowserMainRunnerImpl` is a derived class of `BrowserMainRunner`. The methods `BrowserMainRunnerImpl::Initialize`, `BrowserMainRunnerImpl::Run` and `BrowserMainRunnerImpl::Shutdown` are virtual methods with the `OVERRIDE` keyword at the end of the method declaration line.

`BrowserMainRunner* BrowserMainRunner::Create()` creates a new `BrowserMainRunnerImpl` instance.

Once instantiated, the following methods perform the following actions:
* `BrowserMainRunnerImpl::Initialize` initialises `BrowserMainLoop` and stores it in the `main_loop` variable.
* `BrowserMainRunnerImpl::Run` runs the browser loop via the call `main_loop_->RunMainMessageLoopParts()`
* `BrowserMainRunnerImpl::Shutdown` calls `main_loop_->ShutdownThreadsAndCleanUp()`, and deassigns pointers to various variables by calling their `reset` methods with `NULL` as the parameter.
