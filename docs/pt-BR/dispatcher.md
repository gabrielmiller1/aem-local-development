🇺🇸 [Read this in English](../../dispatcher/README.md)

# Dispatcher

`dispatcher/src/` é **gerado, não versionado** (veja `.gitignore`). No
primeiro `./aem start dispatcher` (ou `./aem start full`), a CLI:

1. Extrai as Dispatcher Tools da Adobe a partir do seu zip do SDK em
   `sdk/`.
2. Copia o `src/` baseline que já vem dentro das próprias Dispatcher Tools
   pra `dispatcher/src/` — um par completo e funcional de farm/vhost
   padrão (arquivos `default_*` imutáveis, `enabled_farms`/
   `enabled_vhosts` já linkados via symlink pra `available_farms`/
   `available_vhosts`), exatamente como a Adobe empacota.
3. Roda o `docker_run.sh` contra essa pasta, apontado pro
   `AEM_PUBLISH_PORT` e `DISPATCHER_PORT` configurados no seu `.env`.

Isso te dá um Dispatcher funcionando de fábrica, **sem nenhum
conhecimento da aplicação** — é infraestrutura, não as regras reais de
cache/filtro do seu projeto.

## Usando a config real do seu projeto no lugar

Se o seu projeto AEM já tem seu próprio `dispatcher/src/` (vindo do
archetype do Cloud Manager), aponte pra ele em vez da config baseline
gerada:

```env
# no .env
DISPATCHER_SRC_DIR=/caminho/absoluto/pro/seu-projeto/dispatcher/src
```

Este repositório nunca lê nem modifica os arquivos do seu projeto — ele só
passa o caminho pro `docker_run.sh` da Adobe.
