---
title: "Homelab em NixOS, parte 1: como aprendi a parar de me preocupar e amar o Nix"
date: 2026-09-09T10:00:00-03:00
draft: true
tags:
  - Homelab
  - NixOS
  - Nix
  - Infraestrutura
---

Eu tinha dois servidores em casa e nenhum dos dois eu teria coragem de reinstalar. Um rodava Fedora com a stack de mídia em quadlets do Podman; o outro rodava Debian com um `docker compose` que só crescia. Cada um tinha config espalhada em `~/*-config`, units de systemd escritos à mão, uma ou outra regra de firewall que eu não lembrava por que existia. Funcionava, mas era o tipo de "funciona" que depende de ninguém encostar.

Migrei os dois pra NixOS. Esta série conta como.

<!--more-->

{{< callout type="info" >}}
**A série**

1. **Como aprendi a parar de me preocupar e amar o Nix** (você está aqui)
2. [bifrost: a stack de mídia sem containers](/blog/2026/09/09/homelab-nixos-parte-2-bifrost-media-stack/)
3. [yggdrasil: Home Assistant numa VM e a VLAN IoT com systemd-networkd](/blog/2026/09/09/homelab-nixos-parte-3-yggdrasil-haos-vlan/)
4. [Caddy, AdGuard e segredos com sops-nix](/blog/2026/09/09/homelab-nixos-parte-4-caddy-adguard-sops/)
{{< /callout >}}

## O problema que eu queria resolver

Não era performance nem falta de recurso. Era **estado**. Cada máquina tinha acumulado decisões que só existiam nela: um `chown` feito na mão, uma porta aberta pra um serviço que já nem rodava mais, um `sysctl` ajustado numa madrugada de debug. Se o disco morresse, eu não teria como reconstruir aquilo, porque "aquilo" não estava escrito em lugar nenhum.

Containers ajudam com parte disso, porque a imagem é reproduzível. Mas tudo em volta do container continua sendo estado: a rede compartilhada, os volumes, o timer que roda o Recyclarr às 4h, o usuário `1000` que é dono de tudo. E no Home Assistant era pior, porque metade das coisas que eu queria (add-ons, Matter, Bluetooth) brigava com o modelo de container.

NixOS resolve isso de um jeito bem direto: a máquina inteira é uma função de um repositório. Se está no repo, existe; se não está, não existe. E vem com um bônus que eu não valorizava até precisar: cada `nixos-rebuild switch` gera uma **generation** nova, e a anterior continua no menu de boot. Quebrou a rede num servidor sem monitor? Reboot, escolhe a geração anterior, pronto.

## Os dois servidores

O repo se chama [heimar](https://github.com/iacapuca/heimar). É nórdico antigo pra "os reinos", os mundos que a Yggdrasil sustenta e a Bifröst atravessa. Ficou assim porque os dois hosts já tinham nome antes do repo:

| Host | Hardware | Papel |
|---|---|---|
| **bifrost** | Ryzen 5 3600X, 16 GB, 4 TB HDD pra mídia | Stack de mídia: Plex, Jellyfin, os *arr, qBittorrent |
| **yggdrasil** | Ryzen 5 PRO 5650GE, 16 GB | Hub de automação: Caddy, AdGuard, Tailscale e a VM do Home Assistant OS |

Os dois rodam `nixos-26.05`, e os dois são configurados a partir do mesmo flake.

## A estrutura do repo

```text
flake.nix                 # nixosConfigurations.bifrost e .yggdrasil
hosts/
  bifrost/
    default.nix           # hostname, bootloader, imports
    disko.nix             # layout de disco declarativo
    hardware.nix          # módulos de kernel, firmware
  yggdrasil/
    default.nix  disko.nix  hardware.nix
    bridge.nix            # br0 (LAN) + br20 (VLAN IoT) pra VM
    caddy.nix  adguard.nix  tailscale.nix  virtualisation.nix
modules/
  common.nix              # usuário, ssh, GC do nix, firewall, timezone
  sops.nix                # sops-nix (segredos)
  media/                  # a stack de mídia como serviços nativos
  virtualisation/vfio.nix # GPU passthrough (fase 2, ainda não importado)
secrets/secrets.yaml      # criptografado com age
justfile                  # just switch | build | check | deploy | install
```

A regra é simples: `hosts/<nome>/` é o que só faz sentido naquela máquina; `modules/` é o que pode ser importado por qualquer host. Na prática hoje só `common.nix` e `sops.nix` são compartilhados de fato, e a stack de mídia é importada só pelo bifrost. Mas a separação já paga por si quando eu quero ler "o que essa máquina tem de especial" sem passar pelo que é padrão.

## O flake

```nix {filename="flake.nix"}
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";
    disko = {
      url = "github:nix-community/disko";
      inputs.nixpkgs.follows = "nixpkgs";
    };
    sops-nix = {
      url = "github:Mic92/sops-nix";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { nixpkgs, disko, sops-nix, ... }:
    let
      systems = [ "x86_64-linux" "aarch64-darwin" ];
      forAllSystems = f: nixpkgs.lib.genAttrs systems
        (system: f nixpkgs.legacyPackages.${system});
    in {
      nixosConfigurations = {
        bifrost = nixpkgs.lib.nixosSystem {
          system = "x86_64-linux";
          modules = [
            disko.nixosModules.disko
            sops-nix.nixosModules.sops
            ./hosts/bifrost
          ];
        };
        yggdrasil = nixpkgs.lib.nixosSystem {
          system = "x86_64-linux";
          modules = [
            disko.nixosModules.disko
            sops-nix.nixosModules.sops
            ./hosts/yggdrasil
          ];
        };
      };

      formatter = forAllSystems (pkgs: pkgs.nixfmt);

      devShells = forAllSystems (pkgs: {
        default = pkgs.mkShell {
          packages = with pkgs; [
            just nixos-rebuild sops ssh-to-age age nixfmt deadnix statix
          ];
        };
      });
    };
}
```

Três inputs: `nixpkgs` (pinado no canal estável), `disko` (particionamento declarativo) e `sops-nix` (segredos). O `inputs.nixpkgs.follows` garante que os dois usem o mesmo nixpkgs que o resto, senão você acaba com duas árvores de pacotes na máquina.

O detalhe que importa é o `systems` incluir `aarch64-darwin`. Eu administro tudo de um Mac, e `formatter` e `devShells` são saídas por sistema. Sem isso, `nix develop` no Mac simplesmente não acha nada.

## O que todo host tem em comum

```nix {filename="modules/common.nix"}
{ pkgs, ... }:
{
  nix.settings.experimental-features = [ "nix-command" "flakes" ];
  nix.settings.auto-optimise-store = true;
  nix.gc = {
    automatic = true;
    dates = "weekly";
    options = "--delete-older-than 14d";
  };
  boot.tmp.cleanOnBoot = true;

  time.timeZone = "America/Sao_Paulo";

  networking.firewall = {
    enable = true;
    allowedTCPPorts = [ 22 ];
  };

  users.users.plpbs = {
    isNormalUser = true;
    extraGroups = [ "wheel" "libvirtd" "render" "video" "media" ];
    openssh.authorizedKeys.keys = [
      "ssh-ed25519 AAAA... plpbs"
    ];
  };
  security.sudo.wheelNeedsPassword = false;

  services.openssh = {
    enable = true;
    settings.PasswordAuthentication = false;
    settings.PermitRootLogin = "no";
  };

  environment.systemPackages = with pkgs; [ git vim htop curl ];
}
```

Duas escolhas que valem comentário. A primeira é o **GC semanal com 14 dias de retenção**: generations são ótimas, mas cada uma segura pacotes no `/nix/store`, e sem GC um servidor pequeno enche em alguns meses. Duas semanas dá margem pra perceber que algo quebrou e voltar.

A segunda é que **só a porta 22 está aqui**. Cada serviço abre a própria porta no seu módulo, via `openFirewall = true` ou declarando explicitamente. Isso foi uma lição do Debian antigo, que tinha meia dúzia de portas abertas pra serviços que já tinham sumido. Quando a porta mora ao lado do serviço, ela some junto com ele.

{{< callout type="warning" >}}
Com `PasswordAuthentication = false`, a chave SSH é o **único** jeito de entrar. Nunca faça deploy de um host sem uma chave real ali. Eu quase fiz isso uma vez com um placeholder, e o único motivo de não ter dado problema foi ter lido o próprio comentário antes.
{{< /callout >}}

## Disco declarativo com disko

Antes do NixOS, particionar era um passo manual que eu fazia uma vez e esquecia. Com [disko](https://github.com/nix-community/disko) o layout vira parte do repo:

```nix {filename="hosts/yggdrasil/disko.nix"}
{
  disko.devices.disk.main = {
    type = "disk";
    device = "/dev/disk/by-id/nvme-SAMSUNG_...";
    content = {
      type = "gpt";
      partitions = {
        ESP = {
          size = "1G";
          type = "EF00";
          content = {
            type = "filesystem";
            format = "vfat";
            mountpoint = "/boot";
            mountOptions = [ "umask=0077" ];
          };
        };
        swap = {
          size = "16G";
          content = { type = "swap"; resumeDevice = true; };
        };
        root = {
          size = "100%";
          content = {
            type = "btrfs";
            extraArgs = [ "-f" ];
            subvolumes = {
              "@"          = { mountpoint = "/";           mountOptions = [ "compress=zstd" "noatime" ]; };
              "@home"      = { mountpoint = "/home";       mountOptions = [ "compress=zstd" "noatime" ]; };
              "@nix"       = { mountpoint = "/nix";        mountOptions = [ "compress=zstd" "noatime" ]; };
              "@snapshots" = { mountpoint = "/.snapshots"; };
              "@srv"       = { mountpoint = "/srv";        mountOptions = [ "noatime" "nodatacow" ]; };
            };
          };
        };
      };
    };
  };
}
```

Root em **btrfs com subvolumes** (snapshots e checksums são seguro contra crash barato). No bifrost, o HDD de 4 TB que guarda mídia e downloads é **xfs**, não btrfs: escrita aleatória de torrent fragmenta filesystem copy-on-write de um jeito feio, e `nodatacow` só resolve parcialmente.

O `device` é sempre por `by-id`, nunca `/dev/nvme0n1`. Quando tem dois NVMe na máquina, a ordem de enumeração muda entre boots, e disko **apaga** o disco que você apontar.

Com isso, uma instalação do zero vira um comando a partir do Mac, usando [nixos-anywhere](https://github.com/nix-community/nixos-anywhere):

```bash
just install yggdrasil root@192.168.1.128
```

Ele boota o alvo num instalador via kexec, roda o disko, instala o sistema e reinicia. Depois do primeiro boot eu regenero o `hardware.nix` real e commito.

## O fluxo do dia a dia

Tudo passa pelo `justfile`:

```make {filename="justfile"}
# Build & switch na máquina local (escolhe a config pelo hostname)
switch:
    sudo nixos-rebuild switch --flake .

# Só builda, sem aplicar
build host:
    nixos-rebuild build --flake .#{{host}}

check:
    nix flake check

update:
    nix flake update

# Instalação do zero (⚠️ APAGA os discos conforme o disko.nix do host)
install host target:
    nix run github:nix-community/nixos-anywhere -- --flake .#{{host}} {{target}}

# Deploy remoto num host já instalado
deploy host target:
    TMPDIR=/tmp nixos-rebuild switch --flake .#{{host}} \
        --build-host {{target}} --target-host {{target}} --sudo

secrets:
    sops secrets/secrets.yaml
```

O `deploy` merece explicação, porque ele tem três coisas que não são óbvias.

**`--build-host` aponta pro próprio servidor.** Meu Mac é `aarch64-darwin`; os servidores são `x86_64-linux`. O Mac não consegue buildar a config deles localmente, então o build acontece no alvo e o `switch` também. O Mac só manda o flake e orquestra.

**`host` e `target` são argumentos separados, nessa ordem.** A primeira versão do justfile tinha `.#bifrost` fixo e só o alvo parametrizado. Se eu tivesse rodado `just deploy plpbs@yggdrasil`, teria construído a config do bifrost e aplicado no yggdrasil: hostname errado, serviços errados. No `install`, o disko errado. Agora os dois são explícitos e o comentário avisa.

**`TMPDIR=/tmp` é obrigatório no Mac.** `nixos-rebuild` cria um socket de SSH ControlMaster dentro de `$TMPDIR`, e `nix develop` aponta isso pra algo como `/tmp/nix-shell.XXXXXX/`. Esses 20 caracteres extras estouram o limite de 104 bytes que o macOS impõe pra caminhos de socket Unix. O erro é `unix_listener: path "..." too long for Unix domain socket`, e ele não aparece no Linux, que permite 108 bytes. Levei um tempo pra entender que o problema era o *caminho*, não o SSH.

## Coisas que me morderam

**Flakes só enxergam arquivos que o git conhece.** Criou um `.nix` novo e o build diz que o arquivo não existe? `git add` nele. Não precisa commitar, mas precisa estar no índice.

**Uma vírgula numa lista quebra tudo.** Listas em Nix são separadas por espaço, não por vírgula. Adicionei uma segunda chave SSH em `common.nix` com vírgula entre os elementos e o `nix eval` falhou com `syntax error, unexpected ','`. Em **ambos** os hosts, porque `common.nix` é compartilhado. `just check` antes de `just deploy` existe pra pegar isso.

**Generations só valem se você já testou voltar.** Uma das primeiras coisas que fiz depois de instalar o bifrost foi um `nixos-rebuild --rollback` de propósito, só pra ver que funcionava. Recomendo. Foi nesse momento que parei de me preocupar: a confiança de que "se quebrar eu volto" muda o quanto você se atreve a mexer.

## Próximos passos

Nas próximas partes entro em cada host: a [stack de mídia do bifrost sem containers](/blog/2026/09/09/homelab-nixos-parte-2-bifrost-media-stack/), o [yggdrasil com a VM do Home Assistant e a VLAN de IoT](/blog/2026/09/09/homelab-nixos-parte-3-yggdrasil-haos-vlan/), e por fim [Caddy, AdGuard e sops-nix](/blog/2026/09/09/homelab-nixos-parte-4-caddy-adguard-sops/).

Ainda tem coisa aberta no repo: reprovisionar o bifrost num NVMe de 2 TB (o `disko.nix` dele está escrito, mas comentado, porque a máquina ainda roda no disco original), passar a fazer deploy dos dois hosts com um único [colmena apply](https://github.com/zhaofengli/colmena), e talvez [impermanence](https://github.com/nix-community/impermanence), que é o passo lógico depois de você confiar que tudo que importa está no repo.
