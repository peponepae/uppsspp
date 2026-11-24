# Fork lowend — PPSSPP para hardware fraco

Objetivo: máxima performance jogável em PCs fracos, preservando compatibilidade.
Alvo primário: **Celeron 847, 2 GB RAM, Intel HD Gen6, Debian 13 CLI**.

Regra do projeto: nenhuma mudança entra sem número antes/depois. O que foi
testado e rejeitado vale tanto quanto o que entrou.

## Hardware alvo (medido em 2025-11-22)

| Item | Valor | Consequência |
|---|---|---|
| CPU | Celeron 847, 2C/2T, Sandy Bridge, 1.1 GHz fixo | Sem HT, sem Turbo; `-j1` obrigatório no build |
| Cache | 2 MB L3 | Pouca folga para JIT + texturas |
| GPU | Intel HD (SNB GT1, 6 EUs), driver `i915` | Gen6 → **sem Vulkan** |
| OpenGL | Mesa 25.0.7, `crocus_dri.so` | GL 3.3 Core é o teto prático |
| RAM | 1834 MB total, ~1.2 GB disponível em idle | Recurso escasso |
| Swap | 1.9 GB em `/dev/sda3` | Rede de segurança, não estratégia |
| Disco | 430 GB livres em `/dev/sda2` | Sem restrição |
| Áudio | HDA em `/dev/asound/card0` | ALSA direto, sem PulseAudio |
| Display | **Nenhum** (CLI puro, sem X/Wayland) | SDL3 + KMSDRM, ou X mínimo |

Governor: `schedutil`. Considerar `performance` durante medições.

## Estado do build

- Upstream exige **SDL3** (não SDL2) — `CMakeLists.txt:362` aborta sem ele.
  SDL2 foi instalado por engano e é inútil aqui.
- Caminho GL: o frontend SDL usa `SDL_GL_CreateContext`
  (`SDL/SDLGLGraphicsContext.cpp:375`) quando `USING_EGL` está desligado.
  Isso serve tanto para X11 quanto para KMSDRM → **um binário cobre os dois**.
- Risco aberto: `glewInit()` (`SDLGLGraphicsContext.cpp:437`). No Linux o GLEW
  tende a resolver símbolos via GLX, que não existe em KMSDRM. Se falhar, o
  plano B é X mínimo (`xserver-xorg-core` + `xinit`, sem WM).

## Configuração de build

```
-DCMAKE_BUILD_TYPE=Release
-O3 -DNDEBUG -pipe -march=sandybridge -mtune=sandybridge
-DUNITTEST=OFF -DHEADLESS=OFF -DATLAS_TOOL=OFF -DLIBRETRO=OFF
-DUSE_DISCORD=OFF -DUSE_MINIUPNPC=OFF
-DUSE_FFMPEG=ON   # mantido: vídeos de jogo. Candidato a OFF se tempo de build doer
-j1               # nunca mais que isso em 2 GB
```

## Decisões e rejeições

| Item | Decisão | Motivo |
|---|---|---|
| Desligar Vulkan no build | **Rejeitado** | `VULKAN` no CMake só controla WSI (`CMakeLists.txt:104`, `:248`); `GPU/CMakeLists.txt:35` compila `GPU_Vulkan.cpp` incondicionalmente. Ganho seria só de tempo de build, com risco de conflito no rebase. |
| Usar `USING_EGL` | **Rejeitado por ora** | O caminho EGL do frontend é escrito para fbdev/X11. `SDL_GL_CreateContext` cobre os dois cenários com o mesmo binário. |
| X11 completo com WM | **Rejeitado** | Um WM/compositor custa RAM sem retorno. Se X for necessário, será `xinit` sem WM. |

## Tabela de medições

| Data | Mudança | FPS antes | FPS depois | Gargalo | Observação |
|---|---|---|---|---|---|
| — | linha de base (build em andamento) | — | — | — | aguardando primeiro binário |

## Como reproduzir

```bash
./setup-deps.sh      # instala e valida dependências
./build.sh           # configura, compila -j1, valida execução
./build.sh clean     # reconstrói do zero
```
