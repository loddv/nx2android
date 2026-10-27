# nx2android

Faça o porte nativo dos seus próprios jogos de Nintendo Switch para Android: o código arm64 do próprio jogo é executado no dispositivo, com os serviços de sistema do Switch simulados por um host e os gráficos NVN reproduzidos no Vulkan. Escolha a pasta do jogo, clique em **Portar** e obtenha um APK e os dados do jogo, com a opção de instalá-los no dispositivo. Use seus próprios arquivos de jogo e chaves; nenhum conteúdo de jogo está neste repositório.

Esta é a versão para Android do [nxnative](https://github.com/seedds/nxnative), que faz o mesmo no iPhone e no iPad. Veja como o projeto é organizado em [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Jogos

Cada jogo tem um perfil em `nxnative/dm/games/<title ID>/` com um arquivo `android.tsv`. Consulte [docs/PORT_STATUS.md](docs/PORT_STATUS.md) para saber quais jogos funcionam, em quais dispositivos e quais foram verificados para porte em cada versão. A GPU do dispositivo precisa ler texturas compactadas em BC (Adreno, Xclipse; não Mali nem PowerVR).

## Para jogadores: faça o porte de um jogo sem programar

1. Baixe `nx2android-<version>-windows-x64.zip` em [Releases](https://github.com/shirai91/nx2android/releases/latest), descompacte-o e inicie **nx2android.exe**.
2. Na aba **Setup**, clique em **Download missing tools** (o Android NDK, o .NET SDK e o platform-tools, cerca de 1 GB, do Google e da Microsoft) e configure seu **prod.keys**. Todos os itens ficarão verdes.
3. Na aba **Port**, escolha a pasta com os arquivos NSP ou XCI do jogo (jogo base, atualização e DLC) e clique em **Port**.
4. Marque **Install on the device when done** com o dispositivo conectado e a depuração USB ativada, ou copie o APK e a pasta de dados manualmente.

O guia passo a passo está em [packaging/INSTALL.md](packaging/INSTALL.md). Você precisa dos seus próprios arquivos de jogo e do `prod.keys`, extraídos do seu próprio Switch; a ferramenta não inclui nem baixa nenhum deles. Primeiro, é necessário descompactar arquivos NSZ para NSP.

## Configuração a partir do repositório

Instale o [Python](https://www.python.org) 3.11 ou posterior e execute `python -m pip install -r requirements.txt`; em seguida, execute `python -m porter tools --download` (ou instale as ferramentas manualmente):
- [Android NDK r27c](https://developer.android.com/ndk/downloads) (somente o NDK: sem Android Studio, SDK ou JDK)
- [.NET SDK 10](https://dotnet.microsoft.com/download)
- [platform-tools](https://developer.android.com/tools/releases/platform-tools) do Android (adb), para instalar usando a ferramenta
- seu próprio `prod.keys`, extraído do seu Switch

## Fazer o porte de um jogo

O aplicativo:

```
python -m gui
```

Confira a aba **Setup**, escolha a pasta do jogo (o NSP ou XCI do jogo base, sua atualização e DLC) na aba **Port** e clique em **Port**.

Ou use a linha de comando:

```
python -m porter doctor
python -m porter port "D:\Games\NEO" -k "D:\keys\prod.keys" --install
```

O APK (`NeoTheWorldEndsWithYou.apk`) e o pacote de dados (`NeoTheWorldEndsWithYou-data/`) serão criados na pasta do jogo (`-o` permite escolher outra pasta). O primeiro porte demora um pouco (extração, reescrita do código do jogo e tradução dos shaders); nos portes seguintes, o que não mudou é reutilizado. Execute `python -m porter -h` para ver os comandos.

## Estrutura

- `porter/`: o fluxo de porte (`python -m porter`).
- `gui/`: o aplicativo para desktop (`python -m gui`).
- `nxnative/`: nxnative, mantido como uma subtree Git de `seedds/nxnative` (com histórico consolidado), com o runtime Android em `nxnative/dm/android/` e os perfis de jogos em `nxnative/dm/games/`.
- `packaging/`: o pacote para jogadores (`python packaging/build_bundle.py`).
- `devtools/`: scripts para iniciar um jogo em um dispositivo conectado (`devtools/device.json` identifica o dispositivo: `{"serial": "...", "package": "..."}`).
- `probes/`, `RESEARCH.md`: o estudo de viabilidade e as primeiras sondagens no dispositivo.

## Compilar o pacote

```
python -m pip install -r requirements.txt pyinstaller
python packaging/build_bundle.py
```

`dist/nx2android/` e `dist/nx2android-<version>.zip`: o aplicativo, o `nxnative/` incluído no repositório e o guia de instalação; sem toolchains, chaves ou arquivos de jogos.

## Testes

```
python -m unittest discover tests
```

## Atualizar o nxnative

```
git subtree pull --prefix nxnative https://github.com/seedds/nxnative.git main --squash
```

## Licença

MIT, assim como o nxnative (consulte `nxnative/LICENSE`). O código de terceiros mantém sua própria licença.
