# M15 Camera Pro – Premium Edition

Aplicativo de câmera Android nativo (Kotlin, CameraX + Camera2 Interop) pensado para o
**Samsung Galaxy M15 5G**. Foco em qualidade de imagem: cores naturais e vivas, modo noturno
com combinação multi-quadro e controles manuais, sem filtros destrutivos.

> ## ⚠️ Estado atual — leia primeiro
> O código foi escrito **sem acesso a um compilador Android nem a um aparelho**. Isso significa:
> - **Ainda não foi compilado.** O primeiro build no GitHub Actions pode apontar erros de
>   compilação pequenos; eles devem ser corrigidos a partir do log.
> - Os testes unitários (`app/src/test`) foram escritos mas **ainda não foram executados**.
> - **Nada foi testado no Galaxy M15 5G.** A compatibilidade **não está comprovada**.
>   Veja `TESTES_NO_APARELHO.md` para o roteiro de validação.

## Instalação (APK pelo GitHub Actions)

1. Envie este projeto para uma **branch própria** do seu repositório (não sobrescreva projetos anteriores).
2. Abra a aba **Actions** → execução **Build APK** → baixe os artefatos
   `M15CameraPro-debug-apk` (recomendado) ou `M15CameraPro-release-apk`.
3. Descompacte o `.zip` baixado e copie o `.apk` para o celular.
4. No celular, permita "instalar apps desconhecidos" para o app usado para abrir o arquivo e instale.
5. Abra **M15 Camera Pro** e conceda as permissões de câmera (e microfone, para vídeo).

O APK *release* é assinado com a chave de **debug** apenas para ser instalável. Para publicar na
loja, gere seu próprio keystore e troque o `signingConfig` em `app/build.gradle.kts`.

### Compilar localmente
Android Studio (Koala ou mais novo) → *Open* na pasta do projeto → *Run*. Pela linha de comando,
com Gradle 8.9 e JDK 17: `gradle :app:testDebugUnitTest :app:assembleDebug`.
(O Gradle Wrapper não está incluído; o Android Studio o gera, ou rode `gradle wrapper`.)

## Recursos

**Modos:** PHOTO, VIDEO, NIGHT, PRO e PORTRAIT. PORTRAIT só aparece se o aparelho oferecer a
extensão oficial BOKEH — nada é simulado.

**Perfis de processamento (botão no topo e em Ajustes):**
- **ORIGINAL** – o JPEG da câmera é salvo sem nenhum processamento adicional.
- **NATURAL** – ajuste discreto: contraste leve, vibrance baixa, nitidez mínima.
- **PREMIUM** – perfil **PREMIUM COLOR**: curva em S, sombras levemente clareadas, altas luzes
  recuperadas, vibrance (age mais nas cores fracas), proteção de tons de pele, reforço leve de céu e
  vegetação, nitidez só na luminância com *coring* e limite anti-halo.
- "Salvar também o original" (Ajustes) grava a foto ORIGINAL ao lado da processada, para comparar.

**HDR:** usa a extensão HDR oficial (CameraX Extensions) quando disponível; o botão HDR só aparece nesse caso.

**NIGHT:**
1. Usa a extensão **Night** do fabricante se existir.
2. Se não existir: rajada de 4/6/8 quadros → escolhe o mais nítido como referência → alinhamento
   por translação em duas escalas → média por pixel com **rejeição de fantasmas** → redução de
   manchas coloridas **apenas no croma** (não borra a luminância) → curva tonal e nitidez controlada.
3. Flash sempre desligado. Em perfil ORIGINAL o Night faz um disparo simples, sem empilhar.

**VIDEO:** maior qualidade realmente suportada (`Quality.HIGHEST`), estabilização quando o aparelho
declara suporte, foco contínuo padrão da câmera, e — em NATURAL/PREMIUM — taxa fixa de 30 fps e curva de contraste
leve aplicadas pelo próprio ISP quando o aparelho as expõe. **Sem processamento por software em
tempo real** (evita quedas de quadros e aquecimento).

**PRO** (cada controle só habilita se o aparelho informar suporte): compensação de exposição,
ISO + obturador (exposição manual), foco manual, balanço de branco (presets).

**Interface:** pré-visualização em tela ampla, zoom por pinça e por atalhos (inclui a ultra-angular se o
sistema expor zoom < 1×), toque para focar, flash/lanterna, temporizador 3/10 s, grade de terços,
nível horizontal, proporções 4:3 e 16:9, miniatura da última mídia, alternância frontal/traseira.

**Relatório de capacidades:** Ajustes → *Recursos reais do aparelho* lista resoluções, foco, ISO,
obturador, estabilização, extensões, qualidades de vídeo, etc., e permite compartilhar o texto.

## Arquitetura

```
com.m15.camerapro
├── MainActivity.kt              orquestra a tela
├── camera/    CameraSession (CameraX), ProControls, ProMath, DeviceCapabilities, CameraMode
├── capture/   PhotoCapturer (foto + night multi-quadro), VideoRecorder
├── processing/ StripProcessor (cor/nitidez, Kotlin puro), ImageProcessor (Bitmap em faixas),
│              FrameAligner (Kotlin puro), FrameStacker (empilhamento), JpegIO (JPEG + EXIF)
├── settings/  AppSettings, SettingsActivity, CapabilitiesActivity
├── storage/   MediaStorage (MediaStore: Pictures/ e Movies/M15CameraPro)
├── permissions/ PermissionHelper
└── ui/        ShutterButton, GridOverlayView, LevelView, ProPanelController
```

## Limitações técnicas conhecidas

- **Não compilado / não testado em aparelho** (ver topo).
- **Resolução de 50 MP:** o app usa a maior resolução do fluxo JPEG normal do CameraX. O modo de
  máxima resolução do sensor (remosaico 50 MP via Camera2) **não está implementado**. No NIGHT
  multi-quadro o quadro é limitado a ~13 MP para não estourar a memória.
- Alinhamento do NIGHT compensa só **translação** (tremor de mão). Rotação, paralaxe e movimento do
  objeto não são corrigidos; pixels em movimento usam o quadro de referência (sem fantasmas, porém
  com menos redução de ruído nessas áreas). Quadros muito diferentes são descartados.
- Pós-processamento roda por software e leva alguns segundos por foto em imagens grandes. Se faltar
  memória, o original é salvo sem perda.
- Câmeras **macro (2 MP)** e acesso direto às câmeras físicas dependem do que o Android expor; o app
  não as força. Ultra-angular só via zoom < 1× se o sistema oferecer.
- **Vídeo:** limitado ao que o aparelho suporta (esperado Full HD 30 fps no M15). Sem 4K/60 fps.
  Balanço de branco e exposição seguem o automático da câmera (sem trava extra).
- Extensões HDR/Night/Bokeh **dependem do fabricante**; se a Samsung não as expõe ao CameraX, o app
  usa as alternativas descritas acima ou oculta o recurso.
- Modo PRO: com controles manuais ativos, extensões não são usadas (limitação do CameraX).
- Interface travada em retrato (a orientação do arquivo segue o sensor de orientação).

## Licença
Defina a licença do seu projeto.
