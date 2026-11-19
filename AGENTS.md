# AGENTS.md — F75 Studio

Aplicativo desktop e CLI para configurar o teclado AULA F75 no Linux. O binário se chama `f75-studio`; a interface usa Rust (edição 2024), egui/eframe com Wayland e X11, e a comunicação com o teclado usa `hidapi`. O projeto é um fork de [`trenbolone1122/aula-f75-linux`](https://github.com/trenbolone1122/aula-f75-linux); `origin` é [`fredjunior54/aula-f75-linux`](https://github.com/fredjunior54/aula-f75-linux).

## Onde mexer

| Arquivos | Responsabilidade |
| --- | --- |
| `src/main.rs`, `src/doctor.rs` | Entrada do app, comandos da CLI e diagnóstico do hardware. Sem argumentos, abre a interface. |
| `src/ui.rs` | Interface egui: Lighting, Keys, Macros, Audio e Power. Encaminha operações do teclado ao worker. |
| `src/worker.rs` | Thread de I/O da GUI, canais `Job`/`Event`, conexão, aplicação de perfis e envio de frames ao teclado. |
| `src/device.rs`, `src/protocol.rs` | `F75` abre e opera o dispositivo; `protocol.rs` define comandos, formatos de pacote e codificação. Consulte [`docs/PROTOCOL.md`](docs/PROTOCOL.md) antes de alterar o protocolo. |
| `src/layout.rs`, `src/kbview.rs` | Mapeamento físico das teclas para índices da matriz e desenho interativo do teclado (incluindo a vista 3D). |
| `src/macros.rs`, `src/state.rs` | Codificação de macros e persistência dos perfis. Os perfis ficam em `profiles/*.txt`, e `state.txt` guarda o perfil ativo, sob `$XDG_CONFIG_HOME/f75-studio` ou `~/.config/f75-studio`. |
| `src/audio.rs`, `src/fx.rs`, `src/aula_music.rs` | Captura e análise do áudio de saída via PulseAudio/PipeWire e renderização dos efeitos de iluminação em tempo real, incluindo os modos de música da AULA. |
| `src/instance.rs`, `src/tray.rs`, `src/perms.rs` | Instância única via socket Unix, integração com a bandeja e ajuste de permissões udev. |
| `install.sh`, `uninstall.sh`, `packaging/`, `.github/workflows/` | Instalação, regra udev, atalho e empacotamento. O CI testa instalação/remoção em Ubuntu, Debian, Arch e Fedora; o workflow de release gera um AppImage e um tarball. |

## Regras para mudanças

- Mantenha I/O HID e operações bloqueantes fora da thread do egui. Na GUI, use o fluxo `Job`/`Event` em vez de abrir o dispositivo no painel. Preserve a exclusividade entre a GUI/bandeja e a CLI.
- Respeite os limites por conexão: **USB-C** tem suporte completo; **receptor 2.4 GHz** suporta efeitos integrados, RGB por tecla, remapeamento e perfis, mas não macros nem streaming de Studio FX/áudio; **Bluetooth** é somente visualização, sem configuração. Alterações já gravadas por USB-C ou 2.4 GHz continuam ativas em Bluetooth. A detecção prefere USB-C, depois 2.4 GHz e, por último, Bluetooth.
- Não confunda os efeitos executados pelo firmware com os efeitos que enviam frames continuamente. Ao mudar o protocolo, valide a variante de cada transporte afetado e preserve as verificações antes de gravar keymaps. Para mudanças de geometria ou remapeamento, confirme os índices em `src/layout.rs`; a posição visual não é o índice da matriz.
- Preserve a leitura de perfis existentes quando alterar a persistência; confira a gravação e a leitura do novo campo. Mantenha os textos da interface em inglês, como os demais textos do app. Se o comportamento público mudar, atualize a ajuda da CLI e/ou o [`README.md`](README.md); para mudanças no protocolo, atualize `docs/PROTOCOL.md`.

## Build e validação

Use Rust **1.95+**, conforme o README, e as dependências nativas da distribuição listadas na seção *Build by hand*: HID/udev, bibliotecas gráficas e PulseAudio. Mantenha `Cargo.lock` versionado.

```sh
cargo build --locked
cargo test --locked
cargo clippy --locked --all-targets
cargo fmt --all -- --check
```

Os testes atuais cobrem protocolo, macros e modos de música, sem exigir teclado. O código herdado já tem avisos do Clippy e diferenças de formatação; não reformate o repositório inteiro em uma mudança pontual. Verifique se a alteração não introduz novos avisos ou diferenças de formatação.

Para conferir a CLI sem acessar o teclado, teste os dois transportes com `--dry-run` (o padrão simula o receptor):

```sh
cargo run --locked -- --dry-run solid 24 48 96
cargo run --locked -- --dry-run --wired solid 24 48 96
```

Para testar a interface sem o teclado, use `F75_STUDIO_DRY=wired cargo run --locked` ou `F75_STUDIO_DRY=24g cargo run --locked` em uma sessão gráfica, sem outra instância do app aberta. A simulação não valida respostas nem comportamento do firmware.

No hardware real, comece por `f75-studio probe` ou `f75-studio keymap` (somente leitura), com a GUI/bandeja fechada. Sem `--dry-run`, `doctor` pode alterar temporariamente a iluminação e o remapeamento, além de gravar um relatório em `$HOME`. Não rode comandos de escrita, instaladores ou ajustes do udev como parte dos testes automáticos.
