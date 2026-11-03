---
title: "Homelab em NixOS, parte 2: bifrost, a stack de mídia sem containers"
date: 2026-09-09T11:00:00-03:00
tags:
  - Homelab
  - NixOS
  - Plex
  - Jellyfin
  - Self-hosting
---

O bifrost é o servidor de mídia: Plex, Jellyfin, Sonarr, Radarr, Prowlarr, Bazarr, qBittorrent e alguns satélites. No Fedora cada um deles era um container em quadlet, todos na mesma rede `arr.network`, todos rodando como o usuário `1000`, todos com a config num diretório dentro da minha home. No NixOS **nenhum deles é container**. Cada um é um serviço nativo do nixpkgs, com seu próprio usuário de sistema, e o que os liga é um grupo Unix chamado `media`.

<!--more-->

{{< callout type="info" >}}
Esta é a parte 2 da série sobre meu homelab em NixOS. A [parte 1](/blog/2026/09/09/homelab-nixos-parte-1-repo-e-fluxo/) explica a estrutura do repo e o fluxo de deploy.
{{< /callout >}}

## Por que largar os containers

Não foi ideologia. Foi que, olhando o que o container me dava, quase tudo o NixOS já dá de graça:

| No Fedora (quadlet) | No NixOS |
|---|---|
| `plex.container` | `services.plex.enable = true` |
| `sonarr.container`, `radarr.container`, ... | `services.sonarr`, `services.radarr`, ... |
| `homepage.container` | `services.homepage-dashboard` |
| `recyclarr.service` + `.timer` às 04:04 | pacote `recyclarr` + timer do systemd |
| rede `arr.network` compartilhada | não precisa: tudo é `localhost` |
| `/media/plex` montado em cada container | `/mnt/media` no host, grupo `media` |
| `~/*-config` | `/var/lib/<app>`, dono é o serviço |

A imagem do container me dava isolamento de versão. O nixpkgs também dá: a versão do Sonarr é a do commit do nixpkgs que está pinado no `flake.lock`, e se um dia eu precisar de algo mais novo, faço override do pacote. O que eu perdi foi a rede isolada, que nunca me serviu pra nada, e o que ganhei foi cada serviço rodando como seu próprio usuário em vez de tudo como `1000`.

## O módulo

```nix {filename="modules/media/default.nix"}
{ lib, ... }:
{
  imports = [
    ./plex.nix
    ./jellyfin.nix
    ./homepage.nix
    ./recyclarr.nix
    ./flaresolverr.nix
    ./qui.nix
  ];

  users.groups.media.gid = 992;

  users.users = {
    jellyfin.uid = 990;
    plex.uid = 193;
    sonarr.uid = 274;
    radarr.uid = 275;
    bazarr.uid = 994;
    qbittorrent.uid = 992;
  };

  systemd.tmpfiles.rules = [
    "d /mnt/media 2775 root media -"
    "d /mnt/media/movies 2775 root media -"
    "d /mnt/media/tv 2775 root media -"
    "d /mnt/media/downloads 2775 root media -"
  ];

  services.sonarr = { enable = true; openFirewall = true; group = "media"; };
  services.radarr = { enable = true; openFirewall = true; group = "media"; };
  services.bazarr = { enable = true; openFirewall = true; group = "media"; };
  services.prowlarr = { enable = true; openFirewall = true; };
  services.qbittorrent = { enable = true; openFirewall = true; group = "media"; };

  networking.firewall.allowedTCPPorts = [ 6881 ];
  networking.firewall.allowedUDPPorts = [ 6881 ];

  systemd.services.radarr.serviceConfig.UMask = lib.mkForce "0002";
  systemd.services.sonarr.serviceConfig.UMask = lib.mkForce "0002";
  systemd.services.bazarr.serviceConfig.UMask = lib.mkForce "0002";
  systemd.services.qbittorrent.serviceConfig.UMask = lib.mkForce "0002";
}
```

É curto, mas cada bloco tem uma história.

### O grupo `media` e o setgid

Todo serviço que encosta em arquivo de mídia roda com `group = "media"`. Os diretórios em `/mnt/media` são criados pelo `systemd.tmpfiles` com modo `2775`: o `2` na frente é o **setgid**, que faz todo arquivo novo dentro do diretório herdar o grupo `media` em vez do grupo primário de quem criou. Sem isso, o qBittorrent baixa um arquivo como `qbittorrent:qbittorrent` e o Radarr não consegue mover.

### O UMask

O setgid resolve o grupo, mas não a permissão. Por padrão os serviços do nixpkgs rodam com umask `0022`, então um arquivo novo nasce `644`: dono escreve, grupo só lê. Pro Bazarr gravar legenda do lado de um filme que o Radarr importou, o grupo precisa escrever. Daí o `UMask = "0002"` forçado em cada serviço que escreve em `/mnt/media`. O `lib.mkForce` é porque o módulo do nixpkgs já define um valor, e sem forçar dá conflito de definição.

O Jellyfin e o Plex não recebem esse override: eles só leem.

### Os uids pinados

Esse é o bloco que eu mais gosto, porque ele resolve um problema que ainda não aconteceu.

O kernel guarda dono de arquivo como **número**. `root:media` é só como o `/etc/group` renderiza o `992` de volta. Quando você não pina o gid, o NixOS aloca um de um pool (400 a 999) e anota a escolha em `/var/lib/nixos/gid-map`. Isso é estável enquanto a máquina viver. Numa reinstalação, o arquivo some, a alocação acontece de novo, e nada garante que `media` vai ser `992` outra vez.

Eu tenho ~178 GB em `/mnt/media` carimbados com gid `992`. Se eu reinstalar o bifrost (e vou, quando trocar pro NVMe de 2 TB) e o grupo `media` virar `993`, todo aquele conteúdo fica órfão e cada serviço perde acesso ao próprio `/var/lib`. O TODO do repo já carregava o sintoma como passo manual: "depois de restaurar, `chown -R <svc>:media`". Pinar os números remove o passo.

Os valores são os que o bifrost já usa hoje, então hoje isso é um no-op. Na reinstalação vira garantia.

{{< callout type="info" >}}
Conferi contra `nixos/modules/misc/ids.nix` do nixpkgs antes de pinar. Plex (193), Sonarr (274) e Radarr (275) já estão reservados lá, então nunca estiveram em risco. Jellyfin, qBittorrent, Bazarr e o gid `media` **não** estão no registro e vinham do pool dinâmico. Esses quatro são os que o bloco realmente salva. Escolhi manter os seis explícitos porque o arquivo fica mais honesto sobre a intenção.
{{< /callout >}}

O Prowlarr está ausente da lista de propósito. O módulo dele usa `DynamicUser = yes`, o systemd aloca o uid em tempo de execução e cuida de `/var/lib/private/prowlarr` sozinho. Não dá pra pinar, e não precisa: a indireção é exatamente o que faz ele sobreviver a uma reinstalação de graça.

## Plex e Jellyfin: o transcode silencioso

```nix {filename="modules/media/jellyfin.nix"}
{ ... }:
{
  services.jellyfin = {
    enable = true;
    openFirewall = true;
    group = "media";
  };

  users.users.jellyfin.extraGroups = [ "render" "video" ];
}
```

O módulo do Jellyfin cria o usuário, mas não o coloca nos grupos `render` e `video`. Sem isso ele não consegue abrir `/dev/dri` e **não dá erro**: ele cai pra transcode em software, pina a CPU, e você descobre por um stream travando. Duas linhas resolvem, mas você precisa saber que elas faltam.

Tem uma ressalva de hardware aqui. O 3600X **não tem iGPU**. O transcode por hardware só funciona enquanto a GPU dedicada estiver ligada ao host. O plano de fase 2 é passar essa mesma GPU pra uma VM de jogos via VFIO, e nesse momento Plex e Jellyfin voltam pro CPU. Uma placa, dois usos, mutuamente exclusivos. Ainda não decidi, e o módulo `vfio.nix` existe no repo mas não está importado.

## Recyclarr: um timer em vez de um container

O Recyclarr sincroniza perfis de qualidade e custom formats do TRaSH Guides pro Sonarr e Radarr. No Fedora era um container acordado por um `.timer`. Aqui é o pacote do nixpkgs e um timer do systemd, sem intermediário:

```nix {filename="modules/media/recyclarr.nix"}
{ pkgs, ... }:
{
  systemd.services.recyclarr = {
    description = "Recyclarr sync";
    after = [ "sonarr.service" "radarr.service" ];
    serviceConfig = {
      Type = "oneshot";
      StateDirectory = "recyclarr";
      Environment = "RECYCLARR_CONFIG_DIR=/var/lib/recyclarr";
      ExecStart = "${pkgs.recyclarr}/bin/recyclarr sync";
    };
  };

  systemd.timers.recyclarr = {
    description = "Run recyclarr daily";
    wantedBy = [ "timers.target" ];
    timerConfig = {
      OnCalendar = "*-*-* 04:04:00";
      Persistent = true;
    };
  };
}
```

`StateDirectory` faz o systemd criar `/var/lib/recyclarr` com o dono certo. O `recyclarr.yml`, que carrega as API keys dos *arr, vive lá e fica fora do git. `Persistent = true` faz o timer rodar na próxima oportunidade se a máquina estava desligada às 04:04.

## Homepage declarativo

O Homepage era o único serviço onde eu sentia falta de ter a config em YAML no repo. Aqui ela é Nix, e o módulo gera o YAML:

```nix {filename="modules/media/homepage.nix"}
{ ... }:
{
  services.homepage-dashboard = {
    enable = true;
    openFirewall = true;
    listenPort = 3000;
    allowedHosts = "home.pedroluiz.com,192.168.1.92:3000,localhost:3000";

    settings = {
      title = "bifrost";
      layout.Media = { style = "row"; columns = 3; };
    };

    widgets = [
      { resources = { cpu = true; memory = true; disk = "/mnt/media"; }; }
    ];

    services = [
      {
        "Media" = [
          { "Plex" = { href = "https://plex.pedroluiz.com"; icon = "plex.png"; }; }
          { "Jellyfin" = { href = "https://jellyfin.pedroluiz.com"; icon = "jellyfin.png"; }; }
          { "Radarr" = { href = "https://radarr.pedroluiz.com"; icon = "radarr.png"; }; }
          # ...
        ];
      }
    ];
  };
}
```

O `allowedHosts` é obrigatório desde que o Homepage passou a validar o header `Host`. Se o hostname que você acessa não está na lista, a página abre em branco com um erro genérico. Como ele fica atrás do Caddy no outro servidor, o nome público precisa estar aqui.

## O que não é declarativo

Vale ser honesto sobre a fronteira. O que está no repo é: quais serviços existem, como rodam, quem é dono de quê, quais portas abrem. O que **não** está é o estado interno de cada app: a biblioteca do Plex, os indexers do Prowlarr, os perfis do Sonarr, as credenciais do qBittorrent. Isso mora em `/var/lib/<app>` e é backup, não config.

Alguns segredos ainda estão nessa fronteira também. O `qui` (um gerenciador web pro qBittorrent) precisa de um secret de sessão, e hoje ele lê de um arquivo criado na mão em `/var/lib/qui-secret`. O plano é mover pra sops-nix, que já está montado no yggdrasil. Isso fica pra [parte 4](/blog/2026/09/09/homelab-nixos-parte-4-caddy-adguard-sops/).

Na [parte 3](/blog/2026/09/09/homelab-nixos-parte-3-yggdrasil-haos-vlan/), o outro servidor: yggdrasil, o Home Assistant numa VM, e a VLAN de IoT que me custou uma noite de debug.
