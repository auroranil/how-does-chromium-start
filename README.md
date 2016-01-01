# How does Chromium start?
Just me learning how chromium starts running its program by exploring the git repository. Please correct me if I get it wrong somewhere throughout this document (I still need to learn more about C and C++).

It's [basically this](https://www.chromium.org/developers/design-documents/startup), except in more detail.

## Cross platform entry points
As chromium runs on various platforms, there are entry points for each type of operating system there are. For Windows and Mac, there are wrapper functions within the entry point files.

Windows: `chrome/app/chrome_exe_main_win.cc` `int APIENTRY wWinMain(HINSTANCE instance, HINSTANCE prev, wchar_t*, int)`

Mac: `chrome/app/chrome_exe_main_mac.cc` `int main(int argc, char* argv[])`

Linux/other: `chrome/app/chrome_exe_main_aura.cc` `int main(int argc, const char** argv)`

## Function ChromeMain
`chrome/app/chrome_main.cc`

Windows: `DLLEXPORT int __cdecl ChromeMain(HINSTANCE instance, sandbox::SandboxInterfaceInfo* sandbox_info)`

POSIX: `int ChromeMain(int argc, const char** argv)`

Arguments are delegated into the `params` variable. As we are still in the early initialisation process, `ChromeMain` function still has to deal on a platform-by-platform case basis. 
## Function ContentMain
`content/app/content_main.cc` `int ContentMain(const ContentMainParams& params)`
* Uses a scoped pointer (defined in `base/memory/scoped_ptr.h`) to store the return value of `BrowserMainRunner::Create`.
* Initialises browser by calling `BrowserMainRunnerImpl::Initialize` method.
* Runs browser by calling `BrowserMainRunnerImpl::Run` method.
* Once the exit value is returned, `BrowserMainRunnerImpl::Shutdown` is called to shut down the browser, and this exit value is then returned by `ContentMain` function itself.

## Class BrowserMainRunnerImpl
`content/browser/browser_main_runner.cc` 

Note: `BrowserMainRunnerImpl` is a derived class of `BrowserMainRunner`. The methods `BrowserMainRunnerImpl::Initialize`, `BrowserMainRunnerImpl::Run` and `BrowserMainRunnerImpl::Shutdown` are virtual methods with the `OVERRIDE` keyword at the end of the method declaration line.

`BrowserMainRunner* BrowserMainRunner::Create()` creates a new `BrowserMainRunnerImpl` instance.

Once instantiated, the following methods perform the following actions:
* `BrowserMainRunnerImpl::Initialize` initialises `BrowserMainLoop` and stores it in the `main_loop` variable.
* `BrowserMainRunnerImpl::Run` runs the browser loop via the call `main_loop_->RunMainMessageLoopParts()`
* `BrowserMainRunnerImpl::Shutdown` calls `main_loop_->ShutdownThreadsAndCleanUp()`, and deassigns pointers to various variables by calling their `reset` methods with `NULL` as the parameter.

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

Definition (in respective header file): `scoped_ptr<MessagePump> pump_;`

## Class MessagePump

File: `base/message_loop/message_pump*.cc`

This class is platform specific. [wip]

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
