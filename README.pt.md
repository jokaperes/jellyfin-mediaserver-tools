# Jellyfin Mediaserver Tools

Uma coleção de scripts leves para automatizar o download de legendas, atualização de biblioteca e notificações push para servidores de mídia Jellyfin.

## Componentes

1. **fetchsub**: Baixa legendas em português (pt-BR) do OpenSubtitles e SubDL.
2. **postdl**: Executado pelo qBittorrent após a conclusão do download. Roda o fetchsub, atualiza a biblioteca do Jellyfin e envia uma notificação push via ntfy.sh.
3. **grab**: Utilitário de linha de comando para adicionar torrents no qBittorrent com categorias automáticas (anime, movies, shows).

## Instalação

### 1. Arquivo de Configuração
Crie o diretório de configuração e copie o modelo:
```bash
mkdir -p ~/.config/mediaserver
cp config.env.example ~/.config/mediaserver/config.env
```
Preencha as chaves de API e URLs dentro de `~/.config/mediaserver/config.env`.

### 2. Chaves de API do OpenSubtitles
Salve suas chaves de API nos caminhos correspondentes:
```bash
echo "sua_chave_opensubtitles" > ~/.config/opensubtitles/api_key
echo "sua_chave_subdl" > ~/.config/subdl/api_key
```

### 3. Instalação dos Scripts
Mova os scripts para o diretório de binários do sistema e torne-os executáveis:
```bash
sudo cp fetchsub postdl grab /usr/local/bin/
sudo chmod +x /usr/local/bin/fetchsub /usr/local/bin/postdl /usr/local/bin/grab
```

### 4. Integração com o qBittorrent
Adicione o seguinte comando na configuração "Executar programa externo ao concluir torrent" do qBittorrent:
```bash
/usr/local/bin/postdl "%F" "%L" "%N" "%D"
```
