# New Touch Stretch

Aplicativo Android (Kotlin + Jetpack Compose, tema branco e azul) que **estica a proporção da tela inteira do celular**, não só uma imagem dentro do app. Ele altera a resolução lógica do sistema (`wm size` / `wm density`) usando o **Shizuku**, que dá ao app a permissão de "shell" (a mesma do ADB) **direto no próprio celular, sem computador**.

> **Leia isto antes:** o Android não tem nenhuma API pública para esticar a tela de outros apps. O app usa o único caminho real que existe, e o resultado visual **depende do seu aparelho**. Em alguns celulares a imagem preenche a tela esticada; em muitos outros o Android preserva a proporção e mostra **barras pretas**. O app tem um botão **"Testar meu aparelho"** que descobre qual é o seu caso. Detalhes na seção [Limitações](#limitações-explicadas-sem-enrolação).

---

## O que o app faz

| Recurso | Como funciona |
|---|---|
| **Horizontal Stretch** | Deslizante de 100% a 160%. Reduz os pixels renderizados na horizontal e deixa o sistema ampliar. |
| **Vertical Stretch** | Mesmo princípio, no eixo vertical. |
| **Intensidade** | Multiplica a esticada: fator final = 1 + (esticada − 1) × intensidade. |
| **Aplicar** | Envia `wm size LxA` (e opcionalmente `wm density`) ao sistema. |
| **Restaurar proporção original** | `wm size reset` + restaura o DPI de antes. |
| **Ativar/desativar efeito** | Interruptor no topo (liga = aplica, desliga = restaura). |
| **Pré-visualização** | Simulação dentro do app mostrando os dois resultados possíveis: "preenche a tela" e "barras pretas". |
| **Valores atuais** | Fatores, resolução atual, resolução prevista, resolução e DPI originais. |
| **Ajustar densidade (DPI)** | Opcional: tenta manter o tamanho dos elementos. |
| **Modo paisagem** | Troca os eixos para quando o celular está deitado (jogos). |
| **Testar meu aparelho** | Aplica 125% por 15 s e pergunta o que você viu; guarda se o seu celular "estica" ou "põe barras". |
| **Trava de segurança** | Depois de aplicar, você tem 15 s para tocar em **Manter**. Se não tocar, volta ao normal sozinho, mesmo que o app seja fechado (o timer roda no processo shell). |
| **Ícone nas Configurações rápidas** | Botão de emergência: liga/desliga o efeito sem abrir o app. |

---

## Pesquisa: o que o Android permite (e o que não permite)

| Abordagem | Dá para esticar a tela globalmente? | Observações |
|---|---|---|
| Transformar a View/Canvas dentro do app | ❌ Só afeta o próprio app | Não é o que você pediu. |
| API pública do WindowManager | ❌ Não existe | Apps comuns não controlam a composição da tela. |
| Serviço de Acessibilidade (magnificação) | ❌ Só zoom **uniforme** | Escala igual nos dois eixos. Não deforma a proporção. |
| Sobreposição + captura de tela (MediaProjection) | ⚠️ Em teoria sim, na prática ruim | A captura inclui a própria sobreposição (efeito espelho), o toque não acompanha a imagem esticada e conteúdo protegido (bancos, DRM) vira preto. Não implementado, para não entregar algo que finge funcionar. |
| **`wm size` / `wm density` (shell)** | ✅ **Global e real** | Muda a resolução lógica do sistema para todos os apps; o Android também ajusta o toque. Exige permissão de shell (ADB/Shizuku). **É o que o app usa.** |
| Root | ✅ | Mesmo comando, só que via `su`. Não é necessário aqui. |

**Por que precisa de permissão especial?** Mudar a resolução do sistema exige a permissão `WRITE_SECURE_SETTINGS`/identidade shell, que apps comuns não têm. Quem tem é o "shell" do ADB. O Shizuku inicia um processo com essa identidade usando a **Depuração sem fio do próprio celular** (Android 11+), e o New Touch Stretch conversa com ele.

**Sobre o resultado visual:** o `wm size` define a resolução lógica. O sistema (SurfaceFlinger) depois a ajusta ao painel. Em AOSP/Pixel e vários fabricantes ele **preserva a proporção** (barras pretas); em outros aparelhos pode preencher a tela. Relatos de usuários e a própria documentação mostram os dois comportamentos. Por isso o app não promete: ele testa no seu aparelho.

---

## Requisitos

- Android **8.0 ou superior** para instalar o app.
- Para o efeito funcionar: **Android 11 ou superior** (Depuração sem fio permite o Shizuku iniciar sem computador). Em Android 10 ou inferior o Shizuku só inicia com um computador via ADB.
- Uma rede **Wi-Fi** (só para iniciar o Shizuku; depois de iniciado o efeito não depende dela, mas o Shizuku para ao reiniciar o celular ou ao perder o Wi-Fi em alguns aparelhos).
- Uma conta gratuita no GitHub.

---

## Parte 1: gerar o APK pelo celular (GitHub Actions)

Você só precisa do navegador do celular. Se o GitHub mostrar a versão mobile limitada, abra o menu do navegador e marque **"Site para computador"**.

### 1. Criar o repositório
1. Entre em <https://github.com> e faça login (ou crie a conta).
2. Toque em **+** → **New repository**.
3. Nome: `NewTouchStretch`. Deixe **Public** (Actions e Releases são gratuitos em repositórios públicos). Toque em **Create repository**.

### 2. Enviar os arquivos (o jeito mais fácil: um único .zip)
1. Na página do repositório recém-criado, toque em **uploading an existing file** (ou **Add file → Upload files**).
2. Escolha o arquivo **`NewTouchStretch.zip`** que você baixou. **Não precisa extrair.**
3. Toque em **Commit changes**.

### 3. Criar o arquivo do workflow
O GitHub só executa workflows que estejam na pasta `.github/workflows/` do repositório. Pelo celular, o jeito mais simples é criar o arquivo digitando o caminho:

1. **Add file → Create new file**.
2. No campo do nome, digite exatamente: `.github/workflows/build.yml` (as barras `/` criam as pastas).
3. Cole o conteúdo da seção [Conteúdo do build.yml](#conteúdo-do-buildyml) (ou do arquivo `build.yml` que veio junto).
4. Toque em **Commit changes** → **Commit directly to the main branch**.

> Se você extraiu o zip e enviou a pasta inteira de uma vez (por exemplo, usando o Termux/git), o workflow já está no lugar e este passo não é necessário.

### 4. Acompanhar a compilação
1. Aba **Actions**. Você verá o workflow **Build APK** rodando (amarelo → verde, em torno de 5 a 8 minutos na primeira vez).
2. Se não iniciou sozinho: **Actions → Build APK → Run workflow → Run workflow**.
3. Se ficar vermelho, toque na execução e abra o passo que falhou para ver o erro.

### 5. Baixar o APK
1. Volte à página inicial do repositório e toque em **Releases** (lado direito/baixo da página).
2. Abra a versão mais recente (`New Touch Stretch v1.0.N`) e toque no arquivo **`New-Touch-Stretch-v1.0.N.apk`** para baixar.

Cada novo build gera uma nova versão (o número sobe sozinho), e o APK novo **atualiza** o anterior sem perder dados porque todos usam a mesma assinatura.

### 6. Instalar
1. Abra o APK baixado. Se o Android pedir, permita **"Instalar apps desconhecidos"** para o navegador/gerenciador de arquivos.
2. Toque em **Instalar**.

---

## Parte 2: ativar o Shizuku (sem computador)

> Necessário **Android 11+**. Leva uns 3 minutos e precisa ser refeito depois de reiniciar o celular.

1. Instale o **Shizuku** (Google Play, ou o APK oficial em <https://shizuku.rikka.app/download/>).
2. Ative as **Opções do desenvolvedor**: *Configurações → Sobre o telefone → toque 7 vezes em "Número da versão"*.
3. Conecte-se ao **Wi-Fi**.
4. *Configurações → Opções do desenvolvedor → **Depuração sem fio*** → ative. (Em alguns aparelhos: *Depuração por Wi-Fi*.)
5. Abra o **Shizuku** → **"Iniciar via Depuração sem fio"** → siga o guia do próprio Shizuku:
   - toque em **Parear dispositivo com código de pareamento** (em Depuração sem fio);
   - o Android mostra um código de 6 dígitos; digite-o na **notificação** do Shizuku (ela aparece com campo de resposta);
   - volte ao Shizuku e toque em **Iniciar**.
6. O Shizuku deve mostrar **"Shizuku está em execução"**.
7. Abra o **New Touch Stretch** → **Conceder permissão** → **Permitir**. O cartão fica verde: **Conectado ao Shizuku**.

---

## Parte 3: usar o app

1. Escolha **Horizontal** e/ou **Vertical** (comece com 110% a 120%).
2. Ajuste a **Intensidade** se quiser suavizar.
3. Toque em **Testar meu aparelho** (recomendado na primeira vez). Responda **Esticou** ou **Barras pretas**. A tela volta ao normal sozinha.
4. Toque em **Aplicar**. Em seguida aparece o aviso **"Manter esta configuração?"** com contagem de 15 s:
   - **Manter**: o efeito fica ativo.
   - **Reverter**, ou não fazer nada: volta ao normal.
5. Para sair do efeito: **Restaurar proporção original** ou o interruptor no topo.
6. Adicione o ícone do app às **Configurações rápidas** (arraste a barra de notificações → lápis/editar → "New Touch Stretch"). Ele serve de **botão de emergência**.

**Jogos em paisagem:** ative **Modo paisagem** para que "Horizontal" signifique o lado maior da tela.

**Dica sobre DPI:** reduzir a resolução num eixo deixa as telas dos apps "mais estreitas" em dp. Se algum app quebrar o layout, ative **Ajustar densidade (DPI)**.

---

## Se a tela ficar estranha (recuperação)

1. **Não faça nada por 15 s**: a trava de segurança restaura sozinha.
2. Puxe as Configurações rápidas e toque no ícone **New Touch Stretch** (restaura quando o efeito está ativo).
3. Abra o app → **Restaurar proporção original**.
4. Se o Shizuku não estiver mais rodando (reiniciou o celular, perdeu o Wi-Fi): inicie o Shizuku de novo (Parte 2, passo 5) e use o app para restaurar.
5. Alternativa manual no Android: algumas versões têm *Configurações → Tela → Resolução da tela / Tamanho da tela* para voltar ao padrão.

Importante: a resolução alterada **persiste após reiniciar o celular** (reiniciar não restaura), por isso o Shizuku é necessário para desfazer. Mantenha valores moderados (até ~130%) e **Wi-Fi ligado** nas primeiras tentativas.

---

## Limitações explicadas sem enrolação

- **Nada de mágica:** o app não "desenha" por cima da tela. Ele pede ao sistema para renderizar com outra resolução. Se o seu Android manter a proporção, o efeito vira **redução de resolução com barras pretas** (útil para desempenho em jogos, mas não é esticar).
- **A pré-visualização é uma simulação** desenhada no app. A tela real é a que você vê depois de aplicar.
- **Shizuku é obrigatório** para o efeito global. Sem ele, o app abre, mostra o estado e a simulação, mas não altera o sistema.
- **Android 10 ou inferior:** o Shizuku só inicia com computador (ADB).
- **Reinício:** o Shizuku para; a resolução alterada continua.
- **Apps com layout fixo** podem ficar com elementos cortados ou apertados quando a resolução lógica muda.
- **Fabricantes:** alguns (Samsung, Xiaomi, etc.) podem limitar ou reverter o `wm size` em certas versões do sistema. O app verifica se o sistema aceitou a mudança e avisa quando não aceitou.
- **Telas dobráveis / vários monitores:** só a tela principal é tratada.

### Equivalente com ADB (para quem tem computador)
```bash
adb shell wm size            # mostra Physical size e Override size
adb shell wm size 900x2400   # aplica (exemplo: 120% na horizontal em 1080x2400)
adb shell wm density 420     # opcional
adb shell wm size reset      # restaura o tamanho
adb shell wm density reset   # restaura o DPI
```

---

## Estrutura do projeto

```
NewTouchStretch/
├── .github/workflows/build.yml      # compila o APK e publica em Releases
├── signing/nts-release.p12          # chave de assinatura (pública, ver abaixo)
├── settings.gradle.kts, build.gradle.kts, gradle.properties
└── app/
    ├── build.gradle.kts
    └── src/main/
        ├── AndroidManifest.xml
        ├── aidl/.../IShellService.aidl      # contrato com o processo shell
        ├── java/com/newtouch/stretch/
        │   ├── MainActivity.kt, NtsApp.kt
        │   ├── StretchViewModel.kt          # estado da tela, aplicar/reverter/contagem
        │   ├── StretchEngine.kt             # aplica e restaura (usado também pelo ícone rápido)
        │   ├── DisplayOps.kt                # cálculo da resolução e comandos wm
        │   ├── ShizukuController.kt         # permissão e conexão com o Shizuku
        │   ├── ShellService.kt              # roda no processo shell; executa os comandos
        │   ├── StretchTileService.kt        # ícone das Configurações rápidas
        │   ├── Store.kt                     # preferências
        │   └── ui/ (Theme.kt, StretchApp.kt)
        └── res/
```

### Como o cálculo funciona
Para esticar 120% na horizontal em uma tela de 1080 px: resolução lógica = 1080 / 1,2 = **900 px**. O sistema renderiza em 900 px e amplia para o painel. O valor é limitado a no mínimo 50% da resolução física e arredondado para número par.

### Compilar localmente (opcional)
Com JDK 17 e Gradle 8.10+ instalados: `gradle :app:assembleRelease`. O APK sai em `app/build/outputs/apk/release/`.

### Sobre a chave de assinatura
`signing/nts-release.p12` é uma chave **pública** do projeto (senha no `app/build.gradle.kts`). Ela existe para que cada build seja assinado igual e o APK novo atualize o antigo. É adequada para uso pessoal. Se for publicar o app numa loja, gere sua própria chave e guarde-a como segredo do GitHub.

---

## Conteúdo do build.yml

```yaml
name: Build APK

on:
  push:
    branches: [main, master]
  workflow_dispatch:

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Baixar o repositório
        uses: actions/checkout@v4

      # Aceita o projeto de duas formas:
      #  1) arquivos soltos no repositório (settings.gradle.kts na raiz), ou
      #  2) um .zip do projeto enviado pelo navegador do celular (é extraído aqui).
      - name: Localizar o projeto
        id: locate
        shell: bash
        run: |
          set -e
          shopt -s nullglob
          if [ ! -f settings.gradle.kts ]; then
            zips=(*.zip)
            if [ ${#zips[@]} -eq 0 ]; then
              echo "::error::Não encontrei settings.gradle.kts nem um arquivo .zip do projeto na raiz do repositório."
              exit 1
            fi
            echo "Extraindo ${zips[0]} ..."
            mkdir -p _src
            unzip -q "${zips[0]}" -d _src
          fi
          SETTINGS=$(find . -name settings.gradle.kts -not -path './.git/*' -print -quit)
          ROOT=$(dirname "$SETTINGS")
          echo "Projeto em: $ROOT"
          echo "root=$ROOT" >> "$GITHUB_OUTPUT"

      - name: Instalar Java 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Instalar Gradle
        uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.10.2'

      - name: Aceitar licenças do Android SDK
        run: yes | "$ANDROID_HOME/cmdline-tools/latest/bin/sdkmanager" --licenses > /dev/null || true

      - name: Compilar APK (release)
        working-directory: ${{ steps.locate.outputs.root }}
        env:
          NTS_VERSION_CODE: ${{ github.run_number }}
          NTS_VERSION_NAME: 1.0.${{ github.run_number }}
        run: gradle --no-daemon :app:assembleRelease

      - name: Preparar o arquivo do APK
        shell: bash
        run: |
          set -e
          mkdir -p out
          cp "${{ steps.locate.outputs.root }}/app/build/outputs/apk/release/app-release.apk" \
             "out/New-Touch-Stretch-v1.0.${{ github.run_number }}.apk"
          ls -la out

      - name: Guardar o APK como artefato do build
        uses: actions/upload-artifact@v4
        with:
          name: New-Touch-Stretch-APK
          path: out/*.apk

      - name: Publicar em Releases
        uses: softprops/action-gh-release@v2
        with:
          tag_name: v1.0.${{ github.run_number }}
          name: New Touch Stretch v1.0.${{ github.run_number }}
          body: |
            APK do **New Touch Stretch** (build ${{ github.run_number }}).

            Baixe o arquivo `.apk` abaixo, instale no celular e siga o tutorial do README
            (é preciso o app Shizuku para o efeito funcionar).
          files: out/*.apk
```

---

## Créditos

- [Shizuku](https://github.com/RikkaApps/Shizuku) / [Shizuku-API](https://github.com/RikkaApps/Shizuku-API), de RikkaApps.
- Licença do projeto: MIT (veja `LICENSE`).
