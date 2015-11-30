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

## "BrowserMainRunner::Create" function
`content/browser/browser_main_runner.cc` 

`BrowserMainRunner* BrowserMainRunner::Create()` creates a new `BrowserMainRunnerImpl` instance.

