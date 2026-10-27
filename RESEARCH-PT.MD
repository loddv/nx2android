# A abordagem do NX2Apple pode levar jogos do Switch ao Android?

*Notas de pesquisa, 2026-10-24. Baseadas na leitura de [LNPANDY/NX2Apple](https://github.com/LNPANDY/NX2Apple) @ `6da7b16` (upstream [seedds/nxnative](https://github.com/seedds/nxnative) @ `ebe75c9`) e em pesquisas na web. Nada foi compilado nem executado em um dispositivo.*

## Resumo

**Sim, tecnicamente, e a parte da CPU pode ser reaproveitada quase sem alterações.** O projeto não é um emulador. Ele reescreve estaticamente o próprio código ARM64 do jogo para que seja executado nativamente e substitui o Horizon OS e a API gráfica por uma camada host. Os celulares Android também são ARM64, então a ideia central se aplica.

O custo está no **backend gráfico**. Cerca de 1 MB de código-fonte reimplementa a API NVN da Nintendo sobre o Apple Metal; seria necessário reescrevê-lo para Vulkan. A fragmentação das GPUs Android é o principal risco técnico: o suporte a texturas BCn varia, a importação de memória do host não é confiável e os drivers têm bugs.

> **Atualização após os testes no dispositivo (§6):** no Lenovo Legion Y700 (Snapdragon 8 Elite / Adreno 830, Android 16), todas as hipóteses sobre a CPU testadas se confirmaram. Há suporte de hardware a texturas BCn. Afinal, os pools de memória do jogo podem usar cópia zero: aloque um `AHardwareBuffer` e mapeie seu dma-buf no endereço do jogo com `mmap(MAP_FIXED)`. Esse mapeamento é armazenado em cache pela CPU e coerente com a GPU. O maior risco de arquitetura listado abaixo está resolvido neste dispositivo.

A vantagem é menor do que no iOS. O Android já tem emuladores de Switch que executam o código do jogo nativamente (NCE do yuzu e seus sucessores). A **exposição jurídica é a mesma** dos emuladores que a Nintendo visou em fevereiro de 2026, porque o fluxo ainda descriptografa os jogos com `prod.keys`.

---

## 1. O que o NX2Apple realmente é

- Um fork do **nxnative** (MIT, ~2,6 mil estrelas, ~490 forks). O NX2Apple adiciona uma interface gráfica para macOS ("NS Port Builder"), um arquivo RomFS fragmentado (`ROMFSARC`) para que jogos enormes como Tears of the Kingdom caibam no instalador do iOS, compilações retomáveis e sincronização diária com o upstream.
- Como o fluxo é executado em um Mac:

```
seus NSPs + prod.keys
  → nx_extract (C#/LibHac)          descriptografa, aplica a atualização, extrai ExeFS + RomFS
  → patch_nso.py                    reescrita estática de cada NSO (ARM64 → ARM64)
  → shader_split + dmsh_xlate       binários de shaders Maxwell → SPIR-V (tradutor do Ryujinx)
  → spirv-cross → metal/metallib    SPIR-V → MSL → bibliotecas Metal   (etapa exclusiva da Apple)
  → compilação do dm/host + dm/gfx + interface iOS pelo Xcode
  → IPA reduzido + arquivo RomFS    instalação lateral; copie o RomFS para Documents/
```

- **Perfis por jogo** (`dm/games/<titleID>/`) são obrigatórios. Eles contêm IDs de compilação, layout de memória, tabelas de shaders e cortes de código específicos de cada compilação. Há 48 perfis, e estima-se que 13–20 jogos sejam jogáveis no iPhone. Para um jogo novo, a documentação descreve um ciclo de levantamento → perfil provisório → inicialização manual.
- O upstream exige um iPhone 15 Pro / iPad M1 ou mais recente, porque as texturas dos jogos são compactadas em BC e as GPUs Apple mais antigas não conseguem amostrar BC. Jogos grandes precisam da autorização de memória ampliada (~3,4 GB → ~6 GB).

### O que o reescritor altera (`dm/tools/patch_nso.py`)

| Instrução original | Motivo da reescrita | Necessário no Android? |
|---|---|---|
| leituras/gravações de `x18` | O kernel da Apple sobrescreve x18 | **Provavelmente manter.** O Android reserva x18 (ShadowCallStack); a reescrita é inofensiva |
| `mrs/msr tpidr_el0`, `mrs tpidrro_el0` | A libc do host controla o registrador TLS | **Sim.** O Bionic usa `tpidr_el0` para TLS |
| `svc #n`, `mrs cntpct_el0` | Chamadas de sistema / temporizador do Horizon → interface do host | **Sim.** `svc` chamaria o kernel Linux |
| desalinhamento em `sub sp, sp, xN` | A Apple gera uma falha quando sp está desalinhado | **Sim.** O Linux também define SA0 |
| loops de espera ativa | Uma consulta ocupada não custa muito no Switch, mas consome bateria no celular | Sim (consumo de energia) |

A saída (`*.patched.bin`, `fixups.tsv`, `gates.tsv`) é independente da plataforma. **O reescritor pode ser reutilizado sem alterações**, exceto por pequenas convenções do carregador e da página de chaves.

## 2. Portabilidade componente por componente

| Componente | Tamanho (código-fonte) | Dependência da Apple | Trabalho no Android |
|---|---|---|---|
| `nx_extract` (.NET / LibHac) | pequeno | nenhuma | **Nenhum.** Também funciona no Windows/Linux |
| Reescritor `patch_nso.py` + cortes por compilação | ~2,5 mil linhas | nenhuma | **Nenhum / trivial** |
| HLE do host `dm/host` (svc, threads, sistema de arquivos, hid, áudio, vídeos, …) | ~500 KB | APIs Mach (`vm_allocate`, portas de thread, `thread_get_state` para dumps, política de restrição de tempo, `os_proc_available_memory`), Core Audio | **Médio.** Substituir por `mmap`, sinais / `tgkill`, sessões de dicas ADPF, AAudio/Oboe, estatísticas de `/proc`. A maior parte da lógica dos serviços (sistema de arquivos, hid, conta, tempo, decodificação de vídeo via libvpx) é C puro e pode ser reaproveitada |
| Assembly `entry.S`, `nvn_thunks.S` (gerado) | pequeno | sintaxe de assembly Mach-O (prefixos `_`, relocação `@PAGE`) | **Baixo.** Mudança mecânica para a sintaxe ELF |
| Camada de gravação NVN `gfx/nvn/hle_nvn.c` | 229 KB | alguns tipos / hooks Metal | **Médio.** Em grande parte reutilizável |
| **Camada de reprodução NVN `gfx/nvn/nvn_mtl.m`** | **842 KB de Objective-C** | **100% Metal** | **Alto. Reescrita completa como backend Vulkan.** É a maior parte do projeto |
| Jogos Vulkan (`gfx/vk/hle_vk.m`: Hades, Mario Galaxy) | 1,5 mil linhas | camada de compatibilidade Vulkan → Metal | **Mais fácil que no iOS.** As chamadas Vulkan do jogo podem, em grande parte, ser encaminhadas ao driver Vulkan real do celular |
| Jogos GL (`gfx/gl/hle_gl.m`: Digimon, Balatro) | 2 mil linhas | desktop GL 4.5 → Metal | Médio: camada GL → Vulkan (ou no estilo ANGLE / Zink) |
| Cadeia de shaders | ferramentas com 4,6 mil linhas | As duas últimas etapas são SPIRV-Cross → MSL → `metallib` | **Médio e mais simples do que hoje.** A cadeia já produz SPIR-V; elimine as etapas MSL/Metal. Adapte os bindings ao Vulkan móvel (indexação de descritores para bindless) e gere novamente os dados de reflexão. **A máquina de compilação não precisaria mais de um Mac** |
| Texturas (`gfx/tex`) | pequeno | pressupõe BCn em hardware | **Médio.** Mali não tem BCn e o suporte da Adreno varia por GPU/driver; portanto, é necessária uma decodificação BCn por shader de computação (ou CPU) durante o carregamento, o que consome memória |
| Interface iOS `dm/ios` | ~800 linhas | UIKit, GameController | **Baixo.** Criar um aplicativo GameActivity/NativeActivity com entrada Android e empacotamento Gradle |
| Empacotamento / ROMFSARC | — | contorna limites do instalador iOS | Menos crítico: colocar o RomFS em `Android/data/<pkg>/files`. O arquivo ainda acelera a cópia via MTP/adb |
| Etapas de compilação exclusivas do Mac (verificação de heap, captura de shaders GLSLC / Vulkan) | — | iniciar o jogo no host macOS | É necessário um host Linux ou Android para executar essas capturas |

### A única hipótese profunda de arquitetura que não se sustenta

O backend Metal cria cada pool de memória NVN como um **`MTLBuffer` de cópia zero sobre a própria memória do jogo** (`newBufferWithBytesNoCopy`), contando com a memória unificada da Apple. Por isso, as gravações da CPU feitas pelo jogo ficam imediatamente visíveis à GPU, sem rastreamento.

No Android, o equivalente Vulkan é `VK_EXT_external_memory_host`, que **não está disponível de maneira confiável no driver Adreno da Qualcomm**. A alternativa é usar buffers GPU sombra com rastreamento de páginas sujas e uploads — a infraestrutura de "cache de buffers" dos emuladores da família yuzu. É mais lento e muito mais complexo. É provável que isso determine boa parte da reformulação; primeiro, verifique a extensão em cada dispositivo-alvo.

## 3. Android versus iOS para essa abordagem

**Mais fácil no Android**
- Não há proibição de JIT (que, de todo modo, não é necessário), nem truques de assinatura de código ou de autorização.
- Memória: não há limite rígido por aplicativo como os ~3,4 GB do jetsam do iOS; celulares topo de linha têm 12–24 GB.
- Páginas de 16 KB já são compatíveis (a Apple as usa), então kernels Android com páginas de 4 KB e 16 KB funcionam.
- Jogos que renderizam com Vulkan podem usar o driver real em vez de uma camada de tradução.
- Todo o fluxo de compilação poderia ser executado no Windows/Linux.

**Mais difícil no Android**
- **Fragmentação de GPU:** Adreno, Mali, Xclipse e PowerVR diferem em recursos e bugs de driver. A Apple tem uma única família de GPUs.
- Não há o atalho "endereço GPU = endereço CPU" (veja acima); o suporte a texturas BCn é irregular.
- Questões térmicas e clocks sustentados em celulares; muitos dispositivos têm apenas 6–8 GB de RAM.
- **A instalação lateral está ficando mais restrita.** A verificação de desenvolvedores do Google está em vigor desde 30 de setembro de 2026 no Brasil, na Indonésia, em Singapura e na Tailândia (para instalações por lojas participantes), com expansão global planejada para 2027. APKs não verificados precisam do novo "fluxo avançado" (Opções do desenvolvedor, espera de 24 h e, então, permissão para instalar apps não verificados) ou de `adb install`, que continua isento.

## 4. Cenário: o que já existe no Android

- **O yuzu adicionou NCE (Native Code Execution) no Android no fim de 2023**, executando diretamente no processador do celular o código ARM64 do Switch, com ganhos de FPS reportados entre 20% e 100%. É necessário que os jogos usem o espaço de endereçamento de 39 bits; os demais usam o JIT dynarmic como alternativa. Seus derivados (Eden, Citron/Citron Neo, Kenji-NX, …) são as opções atuais no Android.
- Portanto, o principal motivo da existência do nxnative no iOS não se aplica ao Android: o iOS proíbe JIT e impede que emuladores executem rapidamente o código convidado, enquanto emuladores Android já conseguem executar o código dos jogos nativamente. O que um porte Android no estilo nxnative ainda acrescentaria:
  - shaders traduzidos offline (sem travamentos de compilação);
  - HLE ajustado manualmente por jogo (avanço de frames, espera com recuo progressivo, cortes específicos por jogo);
  - nenhuma necessidade de firmware ou chaves no celular.
- **Não encontrei nenhum porte ou fork Android do nxnative** (pesquisa no GitHub, 2026-10-24).
- **Aspectos jurídicos:** em fevereiro de 2026, a Nintendo enviou notificações DMCA contra 13 projetos de emuladores de Switch (Eden, Citron, Kenji-NX, MeloNX, Ryujinx/Ryubing, Sudachi, Suyu, yuzu, …). A justificativa declarada foi a descriptografia de jogos com `prod.keys` como forma de contornar medidas de proteção. nxnative/NX2Apple fazem exatamente a mesma descriptografia durante a compilação, então um porte Android teria a mesma exposição, independentemente da alegação "traga seu próprio dump".
- Licenciamento: nxnative é MIT; ele inclui o tradutor de shaders do Ryujinx (MIT). Reutilizar código do yuzu/Eden (GPLv3), como os decodificadores Vulkan BCn, tornaria o porte GPL.

## 5. Conclusão e caminho sugerido

**É viável, mas é um projeto de vários meses para alguém com domínio de Vulkan, ARM64 e dos detalhes internos do Switch.** A maior parte do esforço vai para o backend de reprodução NVN → Vulkan. Ordem sugerida, começando pelas validações mais baratas:

1. **Host Linux/aarch64, sem interface gráfica.** Remover as dependências de APIs Mach de `dm/host` e adaptar o assembly para a sintaxe ELF. Iniciar um jogo reescrito até `nnMain` usando `adb shell` (ou em uma máquina Linux ARM64). Isso valida o reescritor e o HLE no Linux sem trabalho gráfico. *(~semanas)*
2. **Primeiros pixels com um jogo Vulkan.** Usar Hades (perfil `0100535012974000`), cujo renderizador é Vulkan, de modo que `hle_vk` possa encaminhar a maioria das chamadas ao driver do celular. Adicionar uma interface GameActivity mínima com AAudio e entrada de controle.
3. **Backend NVN → Vulkan.** Reescrever `nvn_mtl.m` usando Vulkan 1.3 + indexação de descritores, com uma estratégia explícita para pools de memória (importação do host quando disponível; caso contrário, memória sombra + rastreamento de páginas sujas). Começar por uma família de GPUs: Adreno 7xx/8xx, com pelo menos 12 GB de RAM. *(a maior parte: meses)*
4. **Cadeia de shaders → SPIR-V para Vulkan** (eliminar SPIRV-Cross/Metal), além de decodificação BCn, pré-aquecimento do cache de pipelines e um empacotador Gradle/APK.
5. Ampliar o suporte a GPUs (Mali, Xclipse) e adicionar mais perfis. Os perfis por jogo e os cortes do reescritor podem ser reutilizados sem alterações, porque são identificados pelos IDs de compilação do jogo, não pela plataforma host.

**Alternativa se o objetivo for apenas "jogar meus jogos no Android":** os emuladores NCE existentes já atendem a esse objetivo. O caminho nxnative faz mais sentido como projeto de pesquisa ou desempenho: sem tradução de shaders em tempo de execução e com ajustes específicos por jogo.

## 6. Resultados no dispositivo: Lenovo Legion Y700 (TB322FC), 2026-10-24

O dispositivo: **Snapdragon 8 Elite (SM8750P)**, 8 núcleos Oryon (6 × 3,53 GHz + 2 × 4,32 GHz), **Adreno 830** (Vulkan 1.3.284, driver Qualcomm de 02/2026), 12 GB de RAM + 12 GB de swap, **Android 16** (SDK 36), kernel 6.6 com páginas de 4 KB, SELinux em modo enforcing.

Os programas de teste estão em `probes/`, compilados com NDK r27c. A saída bruta está em `build/*.txt`. Eles foram executados duas vezes:

- por `adb shell` (domínio SELinux `shell`);
- **dentro de um aplicativo comum**: `build/apk/nx2a-probe.apk`, um NativeActivity sem Java (targetSdk 36, depurável), compilado e assinado com v2 por `probes/app/mkapk.py`, sem JDK nem Android SDK. Foi executado no contexto SELinux `u:r:untrusted_app:s0:c60,c257,c512,c768`, o domínio que um porte real usaria. Os resultados estão em `build/app_results.txt`.

**Todos os resultados abaixo foram idênticos nos dois domínios.**

### CPU / kernel (`probes/cpuprobe.c`)

| Verificação | Resultado | Consequência para o porte |
|---|---|---|
| x18 preservado durante 200 mil chamadas de sistema, 1,8 s de preempção por 16 threads e SIGALRM a cada 0,5 ms | **Preservado** (0 alterações) | Diferentemente da Apple, o kernel nunca sobrescreve x18. A reescrita de x18 do reescritor pode se tornar opcional, desde que o código host nos pontos de entrada não dependa de x18 (ShadowCallStack). Mantenha-a inicialmente; removê-la é uma otimização |
| `tpidr_el0` | Contém o ponteiro TLS do Bionic; um valor estrangeiro sobrevive às trocas de contexto | A reescrita continua necessária (o Bionic e os manipuladores de sinais precisam do TLS) |
| `tpidrro_el0` | Legível, valor 0 | A reescrita para um slot da thread continua necessária (ponteiro TLR do Horizon) |
| `cntvct_el0` / `cntfrq_el0` | Legíveis; **19,2 MHz, igual ao Switch** | Leituras convidadas de `cntpct_el0` podem ser reescritas para um único `mrs cntvct_el0`, em vez de uma chamada ao host, o que custa menos do que na Apple |
| `cntpct_el0` | **SIGILL** | A reescrita é necessária (o nxnative já faz isso) |
| Acesso a `sp` desalinhado | **SIGBUS** (SA0 está definido) | O arredondamento de `sp` pelo reescritor é necessário (já implementado) |
| Espaço de endereçamento | VA de usuário de 39 bits; a maior reserva `PROT_NONE` é de **256 GB** | Suficiente para o bloco convidado de 5 GB (Tears of the Kingdom) e o pool esparso de 64 GB (Monster Hunter Rise) |
| Memória executável | RW→RX anônimo, RWX, código dentro de um bloco reservado, mapeamento RX baseado em arquivo **do diretório `files/` do próprio aplicativo** e um memfd mapeado como RW e RX ao mesmo tempo: tudo **funciona** em `untrusted_app` | Um host pode usar `mmap` diretamente em `*.patched.bin` no armazenamento do aplicativo (a regra W^X do Android para targetSdk ≥ 29 bloqueia `execve`, não `mmap(PROT_EXEC)`) ou copiá-lo para memória anônima |

### GPU (`probes/vkprobe.c`)

- **BC1–BC7 são amostradas e filtradas em hardware** (`textureCompressionBC` = yes), além de ASTC LDR/HDR e ETC2. Não é necessária decodificação de textura por software nesta GPU.
- Compatíveis: shaders de geometria e tesselação, `shaderInt64`, mistura de fonte dupla, operação lógica, limite de profundidade, linhas largas, imagens de armazenamento sem formato, distância de recorte/eliminação, todo o conjunto de **indexação de descritores**, **endereço de dispositivo de buffer** (com captura/reprodução), semáforos de linha do tempo, renderização dinâmica, sync2, feedback de transformação, renderização condicional, cores de borda personalizadas, vértice provocador, controle de recorte de profundidade, `index_type_uint8`, `robustness2`, `descriptor_buffer`, `push_descriptor` e `multi_draw`.
- Ausentes: `shaderFloat64`, `VK_EXT_external_memory_host`, `extended_dynamic_state3`, `graphics_pipeline_library` / `shader_object` (portanto, espere compilações de pipelines; as listas de pré-aquecimento do nxnative são aplicáveis), `memory_budget`, shaders de malha, `map_memory_placed`.
- Limites: intervalo máximo de buffer de armazenamento de 2 GB, 16,7 milhões de imagens amostradas por estágio (na prática, bindless), 32 KB de memória compartilhada de computação, tamanho de subgroup 64. O heap de memória tem 11 GB, é local ao dispositivo e visível ao host, coerente e armazenado em cache (memória unificada).

### Pools de memória do jogo com cópia zero (`probes/vkzc.c`)

O backend Metal do nxnative encapsula a própria memória do jogo como buffers GPU. Sem importação de ponteiro do host, o teste seguiu o caminho inverso: alocar memória compartilhável com a GPU e posicioná-la em um endereço convidado fixo, dentro de uma reserva `PROT_NONE` de 8 GB, em offsets alinhados a 4 KB.

| Método | Resultado |
|---|---|
| `vkMapMemory` e depois `mremap(MREMAP_FIXED)` para o intervalo convidado | ✗ EINVAL (o mapeamento KGSL não pode ser movido) |
| Exportar como `OPAQUE_FD` e depois `mmap(MAP_FIXED)` | ✗ `vkGetMemoryFdKHR` falha |
| **`AHardwareBuffer` (BLOB), fd dma-buf recebido via `AHardwareBuffer_sendHandleToUnixSocket`, `mmap(MAP_SHARED, MAP_FIXED)` no endereço convidado, AHB importado para Vulkan** | ✓ **Funciona.** Gravações da CPU no endereço convidado → cópia pela GPU → leituras da CPU no endereço convidado: 0 palavras incorretas em 2 × 64 MB, sem chamadas de flush ou invalidate. Uma cópia GPU de 64 MB leva ~5,5 ms. A velocidade de leitura/gravação da CPU pelo mapeamento é igual à da memória anônima comum (16,5 contra 15,6 ms de leitura, 2,4 contra 2,1 ms de gravação por 64 MB), portanto, é **armazenada em cache e coerente** |

Como um backend usaria isso: em `nvnMemoryPoolInitialize(pool, ptr, size)`, alocar um AHB de `size`, copiar para ele o conteúdo atual de `[ptr, ptr+size)` e, em seguida, mapear (`mmap(MAP_FIXED)`) o dma-buf sobre esse intervalo convidado. A partir daí, o ponteiro do jogo *é* memória da GPU. Ao finalizar, mapear de volta memória anônima com o conteúdo. Os tamanhos e endereços dos pools NVN já são alinhados a 4 KB, o que corresponde ao tamanho de página neste dispositivo.

Os endereços GPU continuam diferentes dos endereços CPU; portanto, a camada de reprodução precisa de uma busca por intervalo de pool → `VkDeviceAddress`. O nxnative já tem uma: `g_pools` em `nvn_mtl.m` mantém `buf.gpuAddress` por pool.

### O que não é possível concluir com este dispositivo

- Um build de lançamento não depurável (o APK de teste é `debuggable`; isso normalmente não altera o domínio SELinux, mas não foi verificado).
- Limites de memória sob pressão: quanta memória um aplicativo em primeiro plano pode usar antes de o low-memory killer agir. Tears of the Kingdom precisa de mais de 3,4 GB residentes.
- Outras famílias de GPU (Mali, Xclipse), que diferem no suporte a BCn e no modo como a memória AHB é armazenada em cache.
- Código real de jogos. Isso exige seus próprios dumps, processados por `nx_extract` e `patch_nso.py`, que requerem .NET e Python neste PC.

## Fontes

- [LNPANDY/NX2Apple](https://github.com/LNPANDY/NX2Apple) · [README do seedds/nxnative](https://github.com/seedds/nxnative/blob/main/README.md)
- No repositório: `README.md`, `AGENTS.md`, `docs/development.md`, `docs/agents/03-bringup.md`, `dm/gfx/README.md`, `dm/tools/patch_nso.py`, `dm/host/*`, `dm/gfx/nvn/nvn_mtl.m`, `dm/gfx/vk/hle_vk.m`, `dm/gfx/tex/nvn_fmt_mtl.c`
- NCE do yuzu no Android: [overclock3d](https://overclock3d.net/news/software/yuzu-switch-emulator-achieves-20-100-performance-boost-on-android/), [Emulation General Wiki – Emuladores de Switch](https://emulation.gametechwiki.com/index.php/Nintendo_Switch_emulators)
- Onda DMCA de fevereiro de 2026: [Nintendo Life](https://www.nintendolife.com/news/2026/02/13-switch-emulators-hit-by-latest-wave-of-nintendo-takedowns), [VGC](https://www.videogameschronicle.com/news/nintendo-issues-dmca-takedown-on-more-switch-emulators-but-at-least-one-is-fighting-back/), [Android Authority](https://www.androidauthority.com/nintendo-switch-emulator-github-dmca-3640722/)
- Verificação de desenvolvedores Android: [Android Authority](https://www.androidauthority.com/android-developer-verification-rollout-sideloading-flow-3653395/), [9to5Google](https://9to5google.com/2026/08/18/google-gradually-rolling-out-androids-advanced-sideloading-ahead-of-developer-verification/), [Foresight Mobile](https://foresightmobile.com/blog/android-developer-verification-2026)
- BCn em GPUs móveis: [web3dsurvey (BC no Android)](https://web3dsurvey.com/webgpu/features/texture-compression-bc/platform/Android), [detalhes internos do Vortek (Mali)](https://kitemetric.com/blogs/vortek-internals-part-2-mastering-driver-workarounds-for-directx-gaming-on-mali)
- Importação de memória do host na Adreno: [fórum de desenvolvedores Qualcomm](https://developer.qualcomm.com/forum/qdn-forums/software/adreno-gpu-sdk/68161)
