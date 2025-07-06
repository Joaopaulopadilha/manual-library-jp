# 📚 Manual: Criando Bibliotecas Nativas para JP

## 🎯 Visão Geral

As bibliotecas nativas JP permitem que você estenda a linguagem com código C++ personalizado, oferecendo performance máxima e acesso completo às APIs do sistema operacional.

## 🏗️ Tipos de Bibliotecas

### 1. **📐 Biblioteca Simples** 
Usa apenas funções C++ padrão (`<cmath>`, `<string>`, etc.)

### 2. **⚡ Biblioteca Nativa**
Inclui código C++ customizado para funcionalidades específicas

### 3. **📦 Biblioteca Bundled**  
Inclui DLLs/bibliotecas externas empacotadas

---

## 🚀 Guia Passo a Passo

### **Passo 1: Criar Repositório GitHub**

1. Crie repositório com nome: `import_<nome_modulo>`
   - Exemplo: `import_window`, `import_graphics`, `import_audio`

2. Estrutura básica:
```
import_window/
├── module.json          # ← Configuração principal
├── README.md           # ← Documentação
├── src/                # ← Código C++ (se necessário)
│   ├── window.hpp
│   └── window.cpp
├── examples/           # ← Exemplos de uso
│   └── basic_window.jp
└── bin/                # ← Bibliotecas bundled (opcional)
    ├── windows/
    ├── linux/
    └── macos/
```

### **Passo 2: Configurar module.json**

#### **Template Base:**
```json
{
  "name": "window",
  "version": "1.0.0",
  "description": "Native window creation for JP language",
  "author": "seu_usuario_github",
  "license": "MIT",
  
  "native": true,
  
  "includes": [
    "\"window.hpp\""
  ],
  
  "libraries": {
    "windows": ["-lgdi32", "-luser32", "-lkernel32"],
    "linux": ["-lX11", "-lGL"],
    "macos": ["-framework Cocoa"]
  },
  
  "source_files": [
    "src/window.cpp"
  ],
  
  "functions": {
    "createWindow": {
      "description": "Creates a native window",
      "cpp_name": "jp_create_window",
      "params": [
        {"name": "title", "type": "string"},
        {"name": "width", "type": "int"},
        {"name": "height", "type": "int"}
      ],
      "return": {
        "type": "void*",
        "description": "Window handle"
      }
    }
  }
}
```

#### **Campos Obrigatórios:**
- `name`: Nome do módulo (sem "import_")
- `version`: Versão semântica (1.0.0)
- `description`: Descrição clara do módulo
- `functions`: Dicionário de funções disponíveis

#### **Campos para Bibliotecas Nativas:**
- `native: true`: Marca como biblioteca nativa
- `includes`: Headers C++ necessários
- `source_files`: Arquivos .cpp a serem compilados
- `libraries`: Bibliotecas por plataforma
- `custom_code`: Código C++ inline (alternativa a source_files)

### **Passo 3: Implementar Código C++**

#### **src/window.hpp** (Header):
```cpp
#pragma once

#ifdef _WIN32
    #include <windows.h>
    typedef HWND WindowHandle;
#elif __linux__
    #include <X11/Xlib.h>
    typedef Window WindowHandle;
#elif __APPLE__
    typedef void* WindowHandle;
#endif

#include <string>

// Funções exportadas para JP
extern "C" {
    WindowHandle jp_create_window(const std::string& title, int width, int height);
    void jp_show_window(WindowHandle window);
    bool jp_poll_events(WindowHandle window);
    void jp_destroy_window(WindowHandle window);
}
```

#### **src/window.cpp** (Implementação):
```cpp
#include "window.hpp"
#include <iostream>

#ifdef _WIN32
// Implementação Windows
LRESULT CALLBACK WindowProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam) {
    switch (uMsg) {
        case WM_DESTROY:
            PostQuitMessage(0);
            return 0;
        case WM_CLOSE:
            DestroyWindow(hwnd);
            return 0;
    }
    return DefWindowProc(hwnd, uMsg, wParam, lParam);
}

WindowHandle jp_create_window(const std::string& title, int width, int height) {
    HINSTANCE hInstance = GetModuleHandle(nullptr);
    
    // Registra classe da janela
    WNDCLASS wc = {};
    wc.lpfnWndProc = WindowProc;
    wc.hInstance = hInstance;
    wc.lpszClassName = L"JPWindowClass";
    wc.hbrBackground = (HBRUSH)(COLOR_WINDOW + 1);
    wc.hCursor = LoadCursor(nullptr, IDC_ARROW);
    
    RegisterClass(&wc);
    
    // Cria janela
    HWND hwnd = CreateWindowEx(
        0,
        L"JPWindowClass",
        std::wstring(title.begin(), title.end()).c_str(),
        WS_OVERLAPPEDWINDOW,
        CW_USEDEFAULT, CW_USEDEFAULT, width, height,
        nullptr, nullptr, hInstance, nullptr
    );
    
    return hwnd;
}

void jp_show_window(WindowHandle window) {
    ShowWindow(window, SW_SHOW);
    UpdateWindow(window);
}

bool jp_poll_events(WindowHandle window) {
    MSG msg;
    while (PeekMessage(&msg, window, 0, 0, PM_REMOVE)) {
        if (msg.message == WM_QUIT) {
            return false;
        }
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }
    return true;
}

void jp_destroy_window(WindowHandle window) {
    DestroyWindow(window);
}

#elif __linux__
// Implementação Linux com X11
#include <X11/Xlib.h>

static Display* display = nullptr;

WindowHandle jp_create_window(const std::string& title, int width, int height) {
    display = XOpenDisplay(nullptr);
    if (!display) return 0;
    
    int screen = DefaultScreen(display);
    Window root = RootWindow(display, screen);
    
    Window window = XCreateSimpleWindow(
        display, root, 0, 0, width, height, 1,
        BlackPixel(display, screen),
        WhitePixel(display, screen)
    );
    
    XStoreName(display, window, title.c_str());
    XSelectInput(display, window, ExposureMask | KeyPressMask | StructureNotifyMask);
    
    return window;
}

void jp_show_window(WindowHandle window) {
    XMapWindow(display, window);
    XFlush(display);
}

bool jp_poll_events(WindowHandle window) {
    XEvent event;
    while (XCheckWindowEvent(display, window, -1, &event)) {
        if (event.type == DestroyNotify) {
            return false;
        }
    }
    return true;
}

#elif __APPLE__
// Implementação macOS com Cocoa
// (Requer Objective-C++)

#endif
```

### **Passo 4: Documentar no README.md**

```markdown
# JP Window Module

Native window creation for JP programming language.

## Installation
\`\`\`bash
jp install window
\`\`\`

## Usage
\`\`\`jp
import window

win = createWindow("My App", 800, 600)
showWindow(win)

while pollEvents(win) {
    // Game loop
}

destroyWindow(win)
\`\`\`

## Functions

| Function | Description | Parameters |
|----------|-------------|------------|
| `createWindow(title, width, height)` | Creates window | title: string, width: int, height: int |
| `showWindow(window)` | Shows window | window: handle |
| `pollEvents(window)` | Processes events | window: handle |

## Requirements

### Windows
- Visual Studio or MinGW
- No additional libraries

### Linux  
- X11 development libraries: `sudo apt install libx11-dev`

### macOS
- Xcode command line tools
```

### **Passo 5: Criar Exemplos**

#### **examples/basic_window.jp:**
```jp
import window

print("Creating window...")

win = createWindow("JP Window Example", 800, 600)
showWindow(win)

print("Window created! Close the window to exit.")

while pollEvents(win) {
    // Window is running
}

print("Window closed!")
```

---

## 🔧 Configurações Avançadas

### **Custom Code Inline (Alternativa)**

Para código simples, use `custom_code` ao invés de arquivos:

```json
{
  "custom_code": "int jp_add(int a, int b) { return a + b; }\ndouble jp_multiply(double a, double b) { return a * b; }",
  "functions": {
    "add": {"cpp_name": "jp_add", "return": {"type": "int"}},
    "multiply": {"cpp_name": "jp_multiply", "return": {"type": "double"}}
  }
}
```

### **Bibliotecas Bundled**

Para incluir DLLs/bibliotecas externas:

```json
{
  "bundled": true,
  "bundle_info": {
    "windows": {
      "dlls": ["bin/windows/SDL2.dll"],
      "libs": ["bin/windows/SDL2.lib"]
    },
    "linux": {
      "libs": ["bin/linux/libSDL2.so"]
    }
  }
}
```

### **Dependências do Sistema**

```json
{
  "dependencies": {
    "ubuntu": "sudo apt install libsdl2-dev",
    "windows": "Download SDL2 from libsdl.org",
    "macos": "brew install sdl2"
  }
}
```

---

## 📋 Checklist de Qualidade

### **✅ Antes de Publicar:**

#### **Estrutura:**
- [ ] Repositório `import_<nome>` criado
- [ ] `module.json` configurado corretamente
- [ ] `README.md` com documentação clara
- [ ] Exemplos funcionais em `examples/`

#### **Código:**
- [ ] Headers com guards apropriados (`#pragma once`)
- [ ] Funções com prefixo `jp_` 
- [ ] Suporte multiplataforma com `#ifdef`
- [ ] Tratamento de erros adequado
- [ ] Memory management correto

#### **Teste:**
- [ ] Compilação sem erros no Windows
- [ ] Compilação sem erros no Linux
- [ ] Exemplos funcionando
- [ ] Sem vazamentos de memória

#### **Documentação:**
- [ ] Todas as funções documentadas
- [ ] Exemplos práticos incluídos
- [ ] Requisitos do sistema especificados
- [ ] Instruções de instalação claras

---

## 🎯 Exemplos de Bibliotecas por Categoria

### **🎮 Graphics & Games**
```
import_window    - Criação de janelas nativas
import_graphics  - Desenho 2D (GDI/Cairo/Quartz)
import_opengl    - OpenGL moderno
import_input     - Keyboard/mouse/gamepad
import_audio     - Reprodução de som
```

### **🌐 Network & I/O**
```
import_http      - Cliente HTTP
import_socket    - Network sockets  
import_serial    - Comunicação serial
import_file      - Operações de arquivo avançadas
```

### **💻 System & OS**
```
import_process   - Gerenciamento de processos
import_registry  - Windows registry
import_service   - Serviços do sistema
import_crypto    - Criptografia
```

### **🔧 Utilities**
```
import_json      - Parser JSON nativo
import_xml       - Parser XML
import_regex     - Expressões regulares
import_compress  - Compressão/descompressão
```

---

## 🚀 Publicação e Distribuição

### **1. Upload para GitHub:**
```bash
git init
git add .
git commit -m "Initial JP module"
git remote add origin https://github.com/usuario/import_modulo.git
git push -u origin main
```

### **2. Teste de Instalação:**
```bash
jp install modulo
jp list
jp info modulo
```

### **3. Teste de Uso:**
```jp
import modulo
// Use as funções...
```

---

## ⚠️ Boas Práticas

### **Performance:**
- Use tipos primitivos C++ quando possível
- Evite alocações desnecessárias
- Implemente cache para operações custosas

### **Segurança:**
- Valide todos os parâmetros de entrada
- Use RAII para gerenciamento de recursos
- Trate exceções apropriadamente

### **Compatibilidade:**
- Teste em múltiplas plataformas
- Use APIs padrão quando disponível
- Documente requisitos específicos

### **Manutenibilidade:**
- Código bem comentado
- Testes automatizados
- Versionamento semântico
- Changelog atualizado

---

## 🆘 Troubleshooting

### **Erro: "Module not found"**
- Verifique se o repositório existe
- Confirme o nome: `import_<modulo>`
- Teste URL manualmente

### **Erro: "Compilation failed"**
- Verifique sintaxe C++
- Confirme includes corretos
- Teste bibliotecas necessárias

### **Erro: "Function not recognized"**
- Confirme `cpp_name` no JSON
- Verifique assinatura da função
- Teste tipos de parâmetros

---

## 🎉 Conclusão

Com este manual, você pode criar bibliotecas nativas poderosas para JP, expandindo as capacidades da linguagem com performance nativa e acesso completo ao sistema.

**Boas práticas:**
- Comece simples e evolua gradualmente
- Documente tudo claramente  
- Teste em múltiplas plataformas
- Siga convenções da comunidade

**Happy coding!** 🚀
