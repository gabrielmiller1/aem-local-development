🇺🇸 [Read this in English](../../README.md)

# aem-local-development

Um ambiente local para **AEM as a Cloud Service (AEMaaCS)**, pronto pra
desenvolvimento, com o mínimo possível de configuração manual. Clone este
repositório, coloque o SDK oficial da Adobe dentro de `sdk/`, e rode
`./aem start`.

Este projeto entrega **apenas infraestrutura**: Author, Publish, Dispatcher,
o Universal Editor Service, HTTPS, e a configuração OSGi do lado do AEM que
o Universal Editor precisa. Ele **não** gerencia o seu projeto AEM — nada de
`pom.xml`, componentes, frontend, archetype ou código. Seu projeto continua
no repositório dele, e conversa com as portas que este projeto expõe.

Tudo aqui segue a documentação *atual* da Adobe (Experience League), não
tutoriais de terceiros — veja [Referências](#referências) para as páginas
exatas.

**Escopo:** construído e testado contra o **AEM as a Cloud Service**. Não
cobre AEM 6.5 / on-premise (instalador diferente, exige
`license.properties`, artefato de Dispatcher diferente).

## Duas formas de rodar

| | Dev Container | Nativo |
|---|---|---|
| Author / Publish | ✅ | ✅ |
| Universal Editor + HTTPS | ✅ | ✅ |
| Dispatcher | ✅ | ❌ (as Dispatcher Tools da Adobe exigem Docker) |
| Precisa instalar Java/Node/Maven você mesmo | Não — o container já traz tudo | Sim |

Os dois modos rodam exatamente o mesmo script `./aem` e os mesmos comandos.
Escolha o que fizer mais sentido pra sua máquina — veja
[Começando rápido](#começando-rápido) pros dois.

## Pré-requisitos

**Modo Dev Container:**
- Docker Desktop, OrbStack, ou outro runtime compatível com Docker,
  rodando, com `docker` disponível no seu `PATH`.
- VS Code com a extensão Dev Containers.

**Modo Nativo:**
- Java 21 (o AEM SDK exige isso desde a versão de manutenção 2026.2.0 da
  Adobe — JDKs mais novos ou mais antigos no seu `PATH` serão rejeitados).
- Node.js 20+ (exigido pelo Universal Editor Service) e `npx`.
- `openssl` (pros certificados locais), `unzip`, bash.
- Docker só se você também quiser o Dispatcher — veja abaixo.

Só macOS/Linux por enquanto (baseado em bash). Desenvolvedores Windows
devem usar o Dev Container, que é o modo principal e multiplataforma; o
modo nativo no Windows precisaria de Git Bash/WSL e ainda não foi montado
nesta primeira versão.

## Baixando o AEM SDK e o Universal Editor Service

Os dois vêm do **Adobe Software Distribution**, na aba
**AEM as a Cloud Service** — são dois downloads separados, em categorias
diferentes na mesma página:

![Adobe Software Distribution, aba AEM as a Cloud Service, mostrando os downloads do AEM SDK e do Universal Editor Service](../../software-distribution.png)

- **AEM SDK** (categoria *Tooling*, ~1.1 GB) → descompacte, ou simplesmente
  coloque o zip inteiro, dentro de `./sdk/`.
- **Universal Editor Service** (categoria *Server utilities*, ~500 KB) →
  coloque o zip (ou o arquivo `.cjs`) dentro de `./ues/`. Só necessário se
  você for usar o Universal Editor.

Nenhum dos dois vai neste repositório — os dois são proprietários da Adobe
e nunca devem ser commitados.

## Começando rápido

### Opção A — Nativo (sem Docker)

```bash
git clone <este-repositório>
cd aem-local-development
# baixe o AEM SDK (veja acima) e coloque em ./sdk/
./aem configure
./aem start          # Author + Publish
./aem status
```

### Opção B — Dev Container

Abra o repositório no VS Code e escolha **"Reopen in Container"**. O dev
container fornece Java 21, Maven, Node 20, e Docker-in-Docker (pro
Dispatcher) — você não precisa instalar nada disso na sua máquina. Uma vez
dentro, os mesmos comandos se aplicam: `./aem configure`, `./aem start`,
etc. O estado do `crx-quickstart` do AEM fica em volumes nomeados do
Docker (não na pasta montada do workspace) por questão de performance; os
arquivos do seu repositório continuam sendo uma pasta normal, visível no
Finder/Explorer, no seu computador.

### De qualquer forma, uma vez rodando

- Author: http://localhost:4502 (`/crx/de`, `/crx/packmgr`, `/system/console` funcionam normalmente — é o SDK da Adobe sem nenhuma modificação)
- Publish: http://localhost:4503
- As credenciais locais padrão são as próprias `admin`/`admin` do AEM —
  este projeto não define nem sobrescreve isso; mude pelo próprio AEM
  quando ele estiver de pé.
- Publicar (Author → Publish) já funciona de fábrica: o agente de
  replicação padrão do SDK vem desabilitado, com valores de exemplo, então
  o `./aem` habilita ele e aponta pra sua porta de Publish configurada
  automaticamente, em segundo plano, logo depois que o Author termina de
  subir. Nada pra configurar — só ativar o conteúdo pelo Author como de
  costume, e ele chega no Publish.

Pro Universal Editor, suba também o UES e o HTTPS:

```bash
./aem start ues ssl
```

Pra subir tudo de uma vez (Author, Publish, UES, HTTPS, Dispatcher — o
Dispatcher exige Docker de qualquer forma, veja a tabela acima):

```bash
./aem start full
```

## Estrutura de pastas

Tudo abaixo é gerado pelo `./aem configure`/`./aem start` e está no
`.gitignore` — pode apagar a qualquer momento, que é reconstruído na
próxima execução. O que de fato fica versionado neste repositório é só o
script, os templates de configuração, e as duas pastas-placeholder onde
você coloca o SDK/UES.

```
sdk/            você coloca o zip do SDK da Adobe (ou só o quickstart jar) aqui
  .extracted/   o quickstart.jar descompactado uma vez (um cache — veja abaixo)
ues/            você coloca o zip/.cjs do Universal Editor Service aqui
  .extracted/   o .cjs extraído uma vez
certs/          certificado/chave autoassinados, compartilhados pelo local-ssl-proxy e pelo UES
instances/
  author/       a própria cópia do jar do author + seu crx-quickstart (estado)
  publish/      o mesmo, pro publish
  ues/          diretório de trabalho do processo do UES rodando
  run/          *.pid + *.out de cada serviço — é como status/stop/logs funcionam
                (veja "Rastreamento de processos" abaixo)
dispatcher/
  src/          config baseline do Dispatcher da própria Adobe, copiada no primeiro uso
.vscode/
  launch.json   configs geradas de "Attach" pro debug Java (veja abaixo)
.env            sua cópia de config/.env.example
```

**Por que cada instância tem sua própria cópia inteira do jar, em vez de
uma compartilhada:** `sdk/.extracted/quickstart.jar` é descompactado uma
vez a partir do seu zip do SDK, e depois *copiado* (não linkado) pra dentro
de cada uma das pastas `instances/author/` e `instances/publish/` — cerca
de 480MB cada, ~1GB no total pras duas. O próprio procedimento de setup da
Adobe manda copiar o quickstart jar pra uma pasta separada por instância;
compartilhar um único arquivo quebra isso, porque o launcher do Quickstart
resolve o caminho canônico do jar pra decidir seu próprio local de
instalação — um symlink ou hard link faz as duas instâncias resolverem pro
mesmo lugar e colidirem ali.

**Rastreamento de processos (`instances/run/`):** cada serviço roda como
um processo simples em segundo plano (`nohup ... &`), sem nenhum
supervisor cuidando dele, então os arquivos `.pid` são como
`status`/`stop`/`restart` sabem o que está realmente rodando. Os arquivos
`.out` capturam a saída bruta (stdout/stderr) de cada processo desde o
instante em que ele começa — pro Dispatcher/UES/os proxies SSL, esse é o
único log que existe; pro Author/Publish, é uma captura secundária das
primeiras linhas da JVM (antes do próprio sistema de log do AEM começar),
ao lado do log de verdade em `crx-quickstart/logs/error.log`.

## Comandos

| Comando | O que faz | Argumentos |
|---|---|---|
| `./aem configure` | Setup único (idempotente): cria o `.env`, detecta/extrai o jar do SDK, gera certificados e a config de debug do VS Code | nenhum |
| `./aem start` | Sobe serviços | `[serviço...\|full]` — padrão `author publish` |
| `./aem stop` | Para serviços | `[serviço...]` — padrão: tudo que estiver rodando |
| `./aem restart` | Para e sobe de novo | `[serviço...]` — padrão: o que já estiver rodando |
| `./aem status` | Mostra o que está rodando e em quais portas/URLs | nenhum |
| `./aem logs` | Mostra o log de um serviço — o log real do AEM pro author/publish (`crx-quickstart/logs/error.log`), saída bruta do processo pro resto | `<serviço> [-f]` |
| `./aem reset` | Apaga só o `crx-quickstart` (estado/repositório) de uma instância | `<author\|publish\|all>` |
| `./aem shell` | Um shell dentro da pasta da instância, ou `exec` no container do dispatcher rodando | `<author\|publish\|dispatcher>` |

### Serviços (usados com `start`/`stop`/`restart`)

| Serviço | Sobe | Porta padrão | Precisa de Docker |
|---|---|---|---|
| `author` | AEM Author | 4502 | Não |
| `publish` | AEM Publish | 4503 | Não |
| `ues` | Universal Editor Service (HTTPS) | 8000 | Não |
| `ssl` | `local-ssl-proxy` na frente do Author/Publish | 8443 / 8444 | Não |
| `dispatcher` | Apache + o módulo Dispatcher | 8080 | **Sim** |
| `full` | Atalho pra tudo acima | — | Sim (por causa do `dispatcher`) |

## Configuração (`.env`)

Copie `config/.env.example` pra `.env` (ou simplesmente rode `./aem
configure`, que faz isso por você) e mude só o que precisar. Cada porta é
lida uma única vez e se propaga pra tudo que depende dela — a instância do
AEM, sua configuração OSGi, o proxy SSL, o destino do Dispatcher, a config
de debug do VS Code, e as URLs que o `./aem status` mostra. Mude o
`AEM_PUBLISH_PORT` e o Dispatcher automaticamente passa a apontar pra
porta nova; nada fica hardcoded em dois lugares.

| Variável | Padrão | Controla |
|---|---|---|
| `AEM_AUTHOR_PORT` | `4502` | Porta HTTP do Author (e sua porta de debug, `3<porta>`) |
| `AEM_PUBLISH_PORT` | `4503` | Porta HTTP do Publish (e sua porta de debug, `3<porta>`) |
| `DISPATCHER_PORT` | `8080` | Porta do Dispatcher |
| `UES_PORT` | `8000` | Porta HTTPS do Universal Editor Service |
| `SSL_AUTHOR_PORT` | `8443` | Proxy HTTPS na frente do Author |
| `SSL_PUBLISH_PORT` | `8444` | Proxy HTTPS na frente do Publish |
| `CORS_ALLOWED_ORIGINS` | `https://localhost:3000` | Origens permitidas, aplicado às duas instâncias |
| `CORS_ALLOWED_ORIGINS_AUTHOR` / `_PUBLISH` | (usa o valor acima) | Override por instância, se precisarem ser diferentes |
| `AEM_JAVA_HOME` | (usa o `java` do `PATH`) | Aponte pra um JDK 21 se seu `java` padrão não for a versão 21 |

Uma mudança no `.env` só entra em vigor na próxima vez que o serviço
afetado subir — editar ele com o Author já rodando não faz nada sozinho;
rode `./aem restart author` (ou o serviço afetado) pra aplicar.

## Dispatcher

O Dispatcher exige Docker — confirmado pela própria documentação das
Dispatcher Tools da Adobe, que afirma que o Docker é necessário tanto pra
validação de sintaxe quanto pra realmente rodar. Não existe um caminho
nativo suportado, então este projeto não tenta nenhum workaround.

Em Apple Silicon: as versões atuais das Dispatcher Tools já trazem uma
imagem nativa arm64 junto com a amd64, e o `docker_run.sh` carrega a certa
sozinho — nenhum workaround de Rosetta/`DOCKER_DEFAULT_PLATFORM` é
necessário (forçar emulação amd64 numa versão atual na verdade quebra, já
que a imagem amd64 nunca é carregada no cache local de imagens).

Veja [dispatcher.md](dispatcher.md) pra entender como a config baseline é
gerada, e como apontar pra config real do Dispatcher do seu próprio
projeto.

## HTTPS

O Universal Editor exige HTTPS de ponta a ponta: ele serve a si mesmo via
TLS e se recusa a carregar uma página HTTP dentro do seu frame, então o
Author também precisa de TLS.

- O Universal Editor Service já tem suporte a HTTPS embutido e não precisa
  de proxy — `./aem start ues` aponta ele pra um certificado gerado
  localmente.
- Author (e Publish) recebem HTTPS via `local-ssl-proxy`, que a Adobe
  documenta exatamente pra esse propósito nos tutoriais do Universal
  Editor: `./aem start ssl` roda isso por você.
- Um certificado autoassinado é gerado uma vez dentro de `certs/`
  (ignorado pelo git).

**Um único passo manual:** seu navegador vai avisar sobre o certificado
autoassinado na primeira vez. Ou aceite o aviso (ex.: visite
`https://localhost:8000/ping` e prossiga uma vez), ou confie nele no
chaveiro do seu sistema operacional se quiser pular esse passo:

```bash
# macOS
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain certs/certificate.pem
```

## Reset

```bash
./aem reset author     # apaga só instances/author/crx-quickstart
./aem reset publish
./aem reset all
```

O reset nunca toca em `sdk/`, `ues/`, `config/`, ou `.env`. O próximo
`./aem start` reconstrói a instância a partir do jar do SDK e reaplica a
configuração OSGi automaticamente.

## Debug de Java no VS Code

O `./aem configure` e o `./aem start` regeneram o `.vscode/launch.json`
com duas configurações de "Attach" já prontas pra usar, apontando pras
portas de debug que Author/Publish já abrem (`3<porta>` — 34502/34503 por
padrão, ou onde quer que o seu `AEM_AUTHOR_PORT`/`AEM_PUBLISH_PORT`
configurado tenha colocado elas). Abra o painel Run and Debug, escolha
"Attach: Author" ou "Attach: Publish", e pronto — você está colocando
breakpoint numa requisição real do AEM, sem nenhuma configuração manual, e
sem risco de ficar apontando pra uma porta antiga depois de trocar a
porta. Fica fora do git, igual o `.env` e o `certs/` — é gerado, não
mantido à mão.

Isso só consegue resolver o código-fonte de verdade se o seu projeto AEM
estiver aberto no *mesmo* workspace do VS Code que este repositório (por
exemplo, um workspace multi-root com as duas pastas adicionadas). Se você
mantém seu projeto numa janela separada do VS Code, copie os mesmos dois
blocos do `.vscode/launch.json` pro `launch.json` *daquele* projeto — este
repositório nunca edita os arquivos do seu projeto por você, mas o trecho
é um simples copiar-e-colar.

## Configuração OSGi/JCR aplicada automaticamente

Aplicada numa instância nova em `crx-quickstart/install/` (ou, no caso da
replicação, via uma chamada HTTP direta assim que o Author estiver de pé)
toda vez que você dá `start` nela — reescrever o mesmo arquivo/valor toda
vez é o que torna isso idempotente:

- **CORS** (`com.adobe.granite.cors.impl.CORSPolicyImpl`) — tanto Author
  quanto Publish, a partir de `CORS_ALLOWED_ORIGINS` (ou do override por
  instância).
- **Remoção do `X-Frame-Options: SAMEORIGIN`** no Author
  (`org.apache.sling.engine.impl.SlingMainServlet`) — sem isso, o
  Universal Editor não consegue colocar o Author dentro de um iframe.
- **SameSite do Token Authentication Handler** definido como
  `Partitioned` no Author — necessário pro cookie de login-token funcionar
  dentro do iframe do Universal Editor.
- **Agente de replicação Author → Publish** habilitado e apontado pro seu
  `AEM_PUBLISH_PORT` configurado (veja "Começando rápido" acima) — esse
  aqui não é uma config OSGi, é uma atualização de propriedade no JCR,
  então acontece via HTTP assim que a API do Author está de fato no ar,
  em vez de via arquivo.

**Esses arquivos são reescritos a cada start**, não só na primeira vez. Se
você ajustar um desses mesmos PIDs à mão pelo `/system/console/configMgr`
pra testar algo, o próximo `start`/`restart` reverte isso silenciosamente
de volta pro que o `.env` diz.

## Referências

As decisões técnicas deste projeto são baseadas na documentação atual da
Adobe (Experience League), não em tutoriais de terceiros:

- [Local Development Environment Set Up — AEM Runtime](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/local-development-environment-set-up/aem-runtime)
- [Local Development Environment Set Up — Dispatcher Tools](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/local-development-environment-set-up/dispatcher-tools)
- [AEM as a Cloud Service SDK](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/aem-as-a-cloud-service-sdk)
- [Validating and Debugging using Dispatcher Tools](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/content-delivery/validation-debug)
- [Remote debugging the AEM SDK](https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/debugging/debugging-aem-sdk/remote-debugging)
- [Running Your Own Universal Editor Service](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/universal-editor/local-dev)
- [Universal Editor Overview for AEM Developers](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/universal-editor/developer-overview)
- [CORS configuration with AEM Headless](https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/headless/deployment/cross-origin-resource-sharing)

## Licença

[MIT](../../LICENSE)
