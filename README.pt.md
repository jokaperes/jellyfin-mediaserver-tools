# jellyhand

*Automação de Jellyfin sem esforço.*

Três scripts leves que automatizam um fluxo de Jellyfin self-hosted:
**baixar um torrent → buscar a melhor legenda no seu idioma → mandar o
Jellyfin reescanear → receber uma notificação no celular.** Feito para um
servidor Linux headless (originalmente um Raspberry Pi 5), mas funciona em
qualquer máquina com Python 3, `curl` e qBittorrent + Jellyfin.

O idioma padrão é **inglês**, mas dá pra escolher pt-BR (ou 15+ outros) com
uma linha de config — veja [Escolhendo o idioma da legenda](#escolhendo-o-idioma-da-legenda).

> 🇬🇧 English version: [`README.md`](README.md)

## Componentes

| Script     | Linguagem | Função |
|------------|-----------|--------|
| `grab`     | bash      | Adiciona um magnet/torrent no qBittorrent na categoria certa (`anime`/`movies`/`shows`). |
| `postdl`   | bash      | Hook de "executar ao concluir" do qBittorrent. Chama o `fetchsub`, manda o Jellyfin reescanear e envia um push via ntfy. |
| `fetchsub` | python3   | Baixa a melhor legenda (no idioma que você escolher) de um arquivo/pasta, salva como `<video>.<iso>.srt`. Funciona sozinho. |

Cada um é independente — dá pra usar o `fetchsub` sozinho, sem a parte de torrent.

## Instalação

### 1. Arquivo de configuração
```bash
mkdir -p ~/.config/mediaserver
cp config.env.example ~/.config/mediaserver/config.env
$EDITOR ~/.config/mediaserver/config.env   # URLs + API key do Jellyfin + tópico ntfy
```

### 2. Chaves dos provedores de legenda
O `fetchsub` lê as chaves de caminhos fixos (nunca ficam no repo nem no ambiente):
```bash
mkdir -p ~/.config/opensubtitles ~/.config/subdl
echo "SUA_CHAVE_OPENSUBTITLES" > ~/.config/opensubtitles/api_key   # obrigatória
echo "SUA_CHAVE_SUBDL"         > ~/.config/subdl/api_key           # opcional (fallback)
chmod 600 ~/.config/opensubtitles/api_key ~/.config/subdl/api_key
```
- OpenSubtitles: https://www.opensubtitles.com/en/consumers (grátis ≈ 100 downloads/dia).
- SubDL: https://subdl.com/panel/api — **opcional**. Se o arquivo não existir, o SubDL é pulado.

### 3. Instalar os scripts
```bash
sudo install -m 755 fetchsub postdl grab /usr/local/bin/
```

### 4. Integrar com o qBittorrent
No qBittorrent: **Opções → Downloads → "Executar programa externo ao concluir torrent"**:
```
/usr/local/bin/postdl "%F" "%L" "%N" "%D"
```

## Uso

```bash
grab movies "magnet:?xt=urn:btih:..."          # um ou vários links
grab shows  "https://exemplo.org/file.torrent"
grab anime  "magnet:?xt=..." "magnet:?xt=..."

fetchsub "/srv/media/movies/Filme (1999)/Filme (1999).mkv"
fetchsub "/srv/media/shows/Serie/Season 01"    # pasta inteira, recursivo
fetchsub --imdb 133093 "/path/Matrix.mkv"      # forçar um id
```

## Escolhendo o idioma da legenda

O padrão é **inglês (`en`)**, mas qualquer idioma é cidadão de primeira classe.
Defina `SUB_LANG` com o que quiser — pontual:

```bash
SUB_LANG=pt-br fetchsub "/path/Filme.mkv"   # só nessa execução
```
…ou fixe no `~/.config/mediaserver/config.env` (o `postdl` repassa pro `fetchsub`):
```bash
SUB_LANG="pt-br"       # português do Brasil; sidecar vira .por.srt
SUB_LANG_FALLBACK="0"  # 1 = aceita a variante ampla quando a exata falta
```

Idiomas prontos: `en` `es` `es-mx` `fr` `de` `it` `nl` `pl` `ru` `ja` `ko`
`zh-cn` `ar` `tr` `pt-br` `pt` (qualquer código do OpenSubtitles também funciona).
Pra adicionar/ajustar, edite a tabela `LANG_TABLE` no topo do `fetchsub`.

**Estrito por padrão:** só a variante exata é baixada — `pt-br` (Brasil)
**nunca** baixa `pt` (Portugal), e vice-versa. Use `SUB_LANG_FALLBACK=1` se
preferir a variante ampla a ficar sem legenda.

## Como o `fetchsub` escolhe a legenda

Salva como **`<video>.<iso>.srt`** (ex.: `.por.srt` p/ português, `.eng.srt`
p/ inglês) pro Jellyfin selecionar o idioma automaticamente — **não** use
`.pt-br.srt` (o Jellyfin não reconhece).

Ordem dos provedores, parando no primeiro acerto confiável:

1. **OpenSubtitles** — moviehash (sync exata) → **séries: `parent_imdb_id` +
   temporada/episódio** (o `query` de texto livre retorna *nada* pra muitas
   séries; o id do show é resolvido via `/features?…&type=tv` e cacheado) →
   **filmes: `imdb_id`** ou query filtrada por título+ano, relaxada aos poucos.
   Ranking: moviehash → idioma (**pt-BR > pt-PT**) → mesmo tipo de fonte
   (WEB-DL/BluRay) → confiável → downloads.
2. **SubDL** — só se `~/.config/subdl/api_key` existir.
   ⚠️ **Códigos de idioma do SubDL são fora do padrão:** pt-BR é `BR_PT`
   (não `PT-BR`/`PB`/`BR`), Portugal é `PT`.

## Observações

- **Ordem de episódios pode divergir:** alguns rips numeram episódios
  diferente do banco de legendas, então um `SxxExx` certo no nome pode ser o
  episódio errado. Se a legenda parecer estranha, confira pelo conteúdo.
- **Logs:** `~/.config/opensubtitles/fetchsub.log` (detalhado) e
  `~/.config/mediaserver/mediaserver.log` (resumo). Configuráveis via
  `LOG_FILE` / `PROJECT_LOG`.
- **Segurança:** nenhum segredo fica no repo. As chaves moram em `~/.config/...`
  e `config.env` / `api_key` / `*.log` estão no `.gitignore`. Não cole chaves,
  IPs ou seu tópico ntfy em arquivos versionados.

Veja [`AGENTS.md`](AGENTS.md) para um guia de replicação voltado a agentes de IA.

## Licença

MIT — veja [`LICENSE`](LICENSE).
