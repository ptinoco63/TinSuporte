# TinSuporte

Software de acesso remoto para Windows, self-hosted, com sincronização de
clipboard. Construído a partir do **RustDesk** e re-brandado para a TinLabs /
TinSuporte.

> **Licença: AGPL-3.0.** Este é um trabalho derivado do RustDesk.
> Atribuição completa em [`NOTICE`](NOTICE). O código-fonte completo que
> corresponde aos binários distribuídos é este repositório.

---

## Atribuição

| Papel | Quem |
|---|---|
| Código base | [RustDesk](https://github.com/rustdesk/rustdesk) (AGPL-3.0), © Purslane Tech Pte. Ltd. e contribuidores |
| Re-branding, tooling, deploy | TinLabs / **TinSuporte** |
| Motor IA e co-autoria de build | **Zé** — motor de IA do [opencode](https://opencode.ai), modelo `opencode/mimo-v2.6-flash-free` |

A motoria de IA escreveu parte do código e das configurações incluídas neste
repositório; essa parte faz parte da Corresponding Source sob a AGPL-3.0.

Não somos afiliados, endossados ou patrocinados pelo projeto RustDesk ou pela
Purslane Tech Pte. Ltd.

---

## Estado

Funcional e verificado em Windows 11 x64:

- [x] Cliente Windows compilado (`TinSuporte.exe`)
- [x] Re-branding total (nome, ícone, metadados do .exe, página About, links)
- [x] Servidor `hbbs` + `hbbr` em Docker
- [x] Chave de sessão gerada
- [x] Cliente configurado para `tinsuporte.ddns.net`
- [x] Codec de vídeo por hardware (NVENC H.264/HEVC detetado)
- [x] Port forwarding no router (21115/21116 TCP+UDP, 21117 TCP)
- [x] Teste de ligação ao servidor (estado "Pronto", ID `91 971 646`)
- [ ] Teste end-to-end entre duas máquinas

---

## Build do cliente (Windows x64)

### Pré-requisitos

| Ferramenta | Versão |
|---|---|
| Visual Studio Build Tools 2022 | MSVC 14.44 + Windows SDK |
| Rust | 1.98.1 (target `x86_64-pc-windows-msvc`) |
| Python | 3.12+ |
| Flutter | **3.24.5** (obrigatório: `pubspec.lock` exige `>=3.24.0`) |
| Git | 2.x |
| vcpkg | com `VCPKG_ROOT` definido |
| LLVM/Clang | para `libclang.dll` (bindgen, usado pelo `hwcodec`) |
| Docker Desktop | para o servidor |

### Passos

```powershell
# 1. Clonar (inclui submódulo hbb_common)
git clone --recursive https://github.com/<tu>/tinsuporte.git
cd tinsuporte

# 2. Gerar a ponte flutter_rust_bridge (ficheiros ignorados pelo git)
cargo install flutter_rust_bridge_codegen --version 1.80.1 --features uuid --locked
cargo install cargo-expand --version 1.0.95 --locked
cd flutter && flutter pub get && cd ..
flutter_rust_bridge_codegen `
    --rust-input ./src/flutter_ffi.rs `
    --dart-output ./flutter/lib/generated_bridge.dart `
    --c-output ./flutter/macos/Runner/bridge_generated.h

# 3. Bibliotecas C via vcpkg (ffmpeg, libvpx, libyuv, opus, aom)
#    O vcpkg.json na raiz declara o baseline e as ports necessárias.

# 4. Compilar
python build.py --portable --flutter --skip-portable-pack --hwcodec --vram
```

O executável fica em:

```
flutter\build\windows\x64\runner\Release\tinsuporte.exe
```

Ou usa o script directo (vcvars64 + PATH + etapas separadas com detecção de
erro): [`build-tinsuporte.bat`](#).

> **Nota:** não chames `flutter.bat` dentro de um `.bat` sem `call` — o
> controlo é transferido e o script morre. Usa sempre `call flutter`.

---

## Servidor self-hosted

```powershell
cd rustdesk
docker compose up -d
docker logs tinsuporte-hbbs
```

O `hbbs` gera automaticamente `data/id_ed25519` e `data/id_ed25519.pub` no
primeiro arranque. A **chave pública** aparece no log:

```
INFO [...] Key: <SUA_CHAVE_AQUI>
```

### Portas

| Porta | Proto | Serviço | Obrigatório |
|---|---|---|---|
| 21115 | TCP | Teste NAT (`hbbs`) | sim |
| 21116 | TCP **e** UDP | Rendezvous (`hbbs`) | **sim** |
| 21117 | TCP | Relay (`hbbr`) | **sim** |
| 21118 | TCP | WebSocket `hbbs` | não |
| 21119 | TCP | WebSocket `hbbr` | não |

> **UDP 21116 é obrigatório.** Sem ele o rendezvous não funciona mesmo que o
> TCP responda.

### Port forwarding no router

Encaminhar para o IP LAN da máquina que corre o Docker (por exemplo
`192.168.1.211`):

```
21115/tcp -> <IP-LAN>
21116/tcp -> <IP-LAN>
21116/udp -> <IP-LAN>
21117/tcp -> <IP-LAN>
```

Verificar de fora (não da própria LAN, senão o NAT loopback pode dar falso
negativo):

```powershell
# deve dar True a partir de outra rede
Test-NetConnection tinsuporte.ddns.net -Port 21116
```

Ou por serviço externo:

```
https://check-host.net/check-tcp?host=tinsuporte.ddns.net:21116
```

---

## Configuração do cliente

As opções gravam-se em **`%APPDATA%\TinSuporte\config\TinSuporte2.toml`**
(não em `TinSuporte.toml` — `Config::get_options()` lê do `CONFIG2`):

```toml
rendezvous_server = 'tinsuporte.ddns.net'

[options]
custom-rendezvous-server = 'tinsuporte.ddns.net'
relay-server = 'tinsuporte.ddns.net'
key = '<chave-pública-do-hbbs>'
```

As quatro opções que a UI grava são:
`custom-rendezvous-server`, `relay-server`, `api-server`, `key`.

Isto é equivalente a preencher *Network → ID Server / Relay Server / Key* na
interface do TinSuporte.

---

## Re-branding aplicado

| Onde | O quê |
|---|---|
| `libs/hbb_common/src/config.rs` | `APP_NAME = "TinSuporte"` — propaga o nome a todas as strings via `is_rustdesk()` |
| `flutter/windows/runner/Runner.rc` | CompanyName, ProductName, FileDescription, Copyright (com atribuição AI) |
| `flutter/windows/runner/resources/app_icon.ico` | Ícone próprio, 6 tamanhos (256→16) |
| `flutter/lib/desktop/pages/desktop_setting_page.dart` | Bloco About com as 4 linhas de atribuição |

A pasta de configuração, pipe IPC e logs passam a ser `TinSuporte`:
`%APPDATA%\TinSuporte\`, `\\.\pipe\TinSuporte\query`, `tinsuporte_rCURRENT.log`.

---

## Estrutura

```
.
├── NOTICE              Atribuição legal (RustDesk + motor IA)
├── LICENSE             AGPL-3.0
├── README.md           Este ficheiro
├── README.rustdesk.md  README original do upstream
├── build.py            Build oficial do RustDesk
├── src/                Código Rust (inclui bridge_generated.rs, gerado)
├── libs/hbb_common     Submódulo (APP_NAME)
├── flutter/            UI Flutter
├── res/vcpkg*          Overlay de ports/triplets do vcpkg
└── docker-compose.yml  Servidor hbbs + hbbr
```

---

## Segurança

- Use sempre a chave pública do teu próprio `hbbs`. Nunca comeces uma sessão
  com o servidor de outrem.
- Defina palavra-passe / PIN na app antes de expor o serviço à Internet.
- O serviço está exposto em todas as interfaces; restringe no firewall à
  WAN se possível.
