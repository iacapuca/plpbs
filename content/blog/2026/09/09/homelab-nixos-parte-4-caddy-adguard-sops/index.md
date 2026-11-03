---
title: "Homelab em NixOS, parte 4: Caddy, AdGuard e segredos com sops-nix"
date: 2026-09-09T13:00:00-03:00
tags:
  - Homelab
  - NixOS
  - Caddy
  - DNS
  - sops-nix
---

Última parte. O yggdrasil é a porta de entrada de tudo: o AdGuard responde DNS pra casa inteira, o Caddy termina TLS e distribui `*.pedroluiz.com` pros serviços nos dois servidores, e o token da Cloudflare que o Caddy usa pra emitir certificado precisa chegar até ele sem aparecer no git. Os três em Nix.

<!--more-->

{{< callout type="info" >}}
Parte 4 da série sobre meu homelab em NixOS. Antes: [a estrutura do repo](/blog/2026/09/09/homelab-nixos-parte-1-repo-e-fluxo/), [a stack de mídia do bifrost](/blog/2026/09/09/homelab-nixos-parte-2-bifrost-media-stack/) e [a VM do Home Assistant e a VLAN IoT no yggdrasil](/blog/2026/09/09/homelab-nixos-parte-3-yggdrasil-haos-vlan/).
{{< /callout >}}

## Caddy com o plugin da Cloudflare

Já escrevi sobre [por que uso DNS-01 via Cloudflare](/blog/2026/07/28/mesmo-dominio-local-e-remoto-com-tailscale/): nada aqui é alcançável publicamente na porta 80, então HTTP-01 não serve. O Caddy do nixpkgs não vem com o provedor da Cloudflare, mas o pacote tem uma função pra compilar com plugins:

```nix {filename="hosts/yggdrasil/caddy.nix"}
{ pkgs, lib, config, ... }:
let
  bifrost = "192.168.1.92";
  haVm = "192.168.1.125";

  cfTlsOnly = ''
    tls {
      dns cloudflare {env.CF_API_TOKEN}
      resolvers 1.1.1.1
    }
  '';

  cfTls = ''
    ${cfTlsOnly}
    encode gzip
  '';

  proxy = target: ''
    ${cfTls}
    reverse_proxy ${target}
  '';
in
{
  services.caddy = {
    enable = true;
    email = "voce@pedroluiz.com";

    package = pkgs.caddy.withPlugins {
      plugins = [ "github.com/caddy-dns/cloudflare@v0.2.4" ];
      hash = "sha256-dQvk6ezY6TQ1J7PjhCXnThF/SqVgPwBO8/RXzHCY+js=";
    };

    virtualHosts = {
      "ha.pedroluiz.com".extraConfig = proxy "${haVm}:80";
      "adguard.pedroluiz.com".extraConfig = proxy "127.0.0.1:3003";

      "mcp.pedroluiz.com".extraConfig = ''
        ${cfTlsOnly}
        reverse_proxy ${haVm}:9583
      '';

      "plex.pedroluiz.com".extraConfig = proxy "${bifrost}:32400";
      "jellyfin.pedroluiz.com".extraConfig = proxy "${bifrost}:8096";
      "radarr.pedroluiz.com".extraConfig = proxy "${bifrost}:7878";
      "sonarr.pedroluiz.com".extraConfig = proxy "${bifrost}:8989";
      "home.pedroluiz.com".extraConfig = proxy "${bifrost}:3000";
      # ...
    };
  };

  networking.firewall.allowedTCPPorts = [ 80 443 ];
}
```

Como a config é Nix e não Caddyfile, dá pra fazer o que um Caddyfile faria com snippets, mas com uma função de verdade: `proxy "host:porta"` gera o vhost completo. Cada serviço novo é uma linha.

O `mcp` é o único vhost escrito à mão, e o motivo é o `encode gzip`. Ele expõe um servidor MCP que roda como add-on dentro da VM do Home Assistant, e MCP usa HTTP streamável. Comprimir uma resposta em streaming é fonte conhecida de buffering, e gzip não ganha nada nesses payloads. Por isso o bloco de TLS ficou separado (`cfTlsOnly`) do bloco com compressão (`cfTls`): o resto dos vhosts continua comprimindo, esse não.

### O hash que quebra a cada update

O `hash` no `withPlugins` é o hash do vendor do Go depois de adicionar o plugin. Ele **muda quando o nixpkgs muda a versão do Caddy**, mesmo que você não toque no plugin. Já aconteceu três vezes desde agosto: eu rodo `nix flake update`, faço deploy do yggdrasil, e o build morre com:

```text
specified: sha256-PWadA5qr/gR2qDcT8l8u1Xku7LM2HIfWTLOkzezCYy0=
got:       sha256-dQvk6ezY6TQ1J7PjhCXnThF/SqVgPwBO8/RXzHCY+js=
```

A correção é copiar o `got` pro arquivo e buildar de novo. É chato, mas é o comportamento correto: o Nix se recusa a usar um artefato que não bate com o que você declarou. O bifrost nunca é afetado porque não roda Caddy.

## AdGuard: config imutável

```nix {filename="hosts/yggdrasil/adguard.nix"}
{ ... }:
{
  services.adguardhome = {
    enable = true;
    openFirewall = true;
    mutableSettings = false;
    host = "0.0.0.0";
    port = 3003;

    settings = {
      dns = {
        bind_hosts = [ "192.168.1.128" ];
        upstream_dns = [ "1.1.1.1" "8.8.8.8" ];
        bootstrap_dns = [ "1.1.1.1" "8.8.8.8" ];
      };
      filtering = {
        protection_enabled = true;
        filtering_enabled = true;
        rewrites = [
          { domain = "pedroluiz.com"; answer = "192.168.1.128"; enabled = true; }
          { domain = "*.pedroluiz.com"; answer = "192.168.1.128"; enabled = true; }
        ];
      };
    };
  };
}
```

`mutableSettings = false` é a decisão importante. Por padrão o módulo escreve o `AdGuardHome.yaml` uma vez e depois deixa a interface web mandar. Com `false`, o arquivo é regenerado a cada `switch` a partir do que está no Nix, e qualquer coisa que eu mude pela interface é descartada no próximo deploy. Isso é exatamente o que eu quero: os rewrites que fazem `*.pedroluiz.com` resolver pro Caddy são infraestrutura, e infraestrutura mora no repo.

O `bind_hosts` com o IP literal é o que obrigou a `br0` a ser estática, como contei na [parte 3](/blog/2026/09/09/homelab-nixos-parte-3-yggdrasil-haos-vlan/).

{{< callout type="warning" >}}
`openFirewall = true` no módulo do AdGuard abre **só a interface web** (3003), não a porta 53. Eu descobri isso avaliando `config.networking.firewall.allowedTCPPorts` antes de um deploy que tiraria a 53 da lista do host, achando que estava duplicada. Ela não estava. Sem ela, a casa inteira ficava sem DNS. A 53 fica declarada explicitamente em `hosts/yggdrasil/default.nix`, com um comentário dizendo por quê.
{{< /callout >}}

## Segredos com sops-nix

O Caddy precisa do token da API da Cloudflare em `CF_API_TOKEN`. Na primeira versão isso era um arquivo `.env` criado na mão em `/var/lib/caddy-secrets/`, com um comentário no Nix dizendo "crie esse arquivo antes do deploy". Funciona, mas é exatamente o tipo de estado fora do repo que eu queria eliminar.

[sops-nix](https://github.com/Mic92/sops-nix) resolve assim: o arquivo de segredos vai pro git **criptografado** com [age](https://age-encryption.org/), e cada host consegue descriptografar usando uma chave age derivada da própria chave SSH de host. Não tem chave nova pra gerenciar.

### As chaves

```yaml {filename=".sops.yaml"}
keys:
  - &admin_pedro age1lm3vnnd03fkgajg7kzcnqmgqkkqpahv7ecdf5q9m2tvmvgn54ypq70y04s
  - &host_yggdrasil age1lqffcxwsg9cwrf7enqse33c3r7dkxpywup8mdexf4hzgs8gwlahsrdg8eu

creation_rules:
  - path_regex: secrets/[^/]+\.yaml$
    key_groups:
      - age:
          - *admin_pedro
          - *host_yggdrasil
```

Duas chaves públicas: a minha, gerada no Mac com `age-keygen`, e a do yggdrasil, derivada da chave SSH de host:

```bash
ssh-to-age < /etc/ssh/ssh_host_ed25519_key.pub
```

Qualquer segredo em `secrets/*.yaml` fica criptografado pras duas. Eu edito no Mac com `sops secrets/secrets.yaml`, que abre o editor com o conteúdo em claro e re-criptografa ao salvar. O yggdrasil descriptografa na ativação.

### O módulo

```nix {filename="modules/sops.nix"}
{ ... }:
{
  sops = {
    defaultSopsFile = ../secrets/secrets.yaml;
    age.sshKeyPaths = [ "/etc/ssh/ssh_host_ed25519_key" ];
  };
}
```

E o uso, de volta no `caddy.nix`:

```nix {filename="hosts/yggdrasil/caddy.nix"}
  sops.secrets.cloudflare_api_token = { };

  sops.templates."caddy.env".content = ''
    CF_API_TOKEN=${config.sops.placeholder.cloudflare_api_token}
  '';

  systemd.services.caddy.serviceConfig.EnvironmentFile =
    config.sops.templates."caddy.env".path;
```

`sops.secrets.cloudflare_api_token` declara que esse segredo existe e deve ser descriptografado pra `/run/secrets/`. Mas o Caddy quer um arquivo de ambiente no formato `CHAVE=valor`, não o valor cru. É pra isso que serve `sops.templates`: você escreve o arquivo com um **placeholder**, e o sops-nix substitui pelo valor real na ativação. O Nix nunca vê o segredo em claro, então ele nunca vai parar no `/nix/store`, que é legível por todo mundo.

Esse último ponto é o que diferencia sops-nix de "colocar o segredo no Nix e torcer": qualquer string que entre numa expressão Nix acaba no store. O placeholder é o que impede isso.

{{< callout type="info" >}}
Um detalhe que me pegou: sops-nix resolve segredos **na ativação**, não no build. Se você referencia um segredo que ainda não existe no `secrets.yaml`, o build passa e o `switch` falha. Num servidor sem monitor isso é desagradável. Foi por isso que o Tailscale ficou só com `enable = true` por um tempo, sem `authKeyFile`: a máquina já estava registrada na tailnet e reusava a identidade em `/var/lib/tailscale`.
{{< /callout >}}

## O que ainda está fora do repo

Sendo honesto sobre a fronteira, como na parte 2:

- **O secret do `qui`** no bifrost ainda é um arquivo criado na mão. O bifrost ainda não tem a chave dele no `.sops.yaml`; é o próximo.
- **Os USB passthrough da VM** vivem no XML do libvirt, documentados em `host-state/` mas não gerenciados.
- **A config do Home Assistant** em si mora dentro da VM e não é declarativa por natureza. O que eu versiono é a [convenção de nomes das entidades](https://github.com/iacapuca/heimar/blob/main/docs/ha-naming.md), porque isso é uma decisão, não estado.

## Balanço

Depois de dois meses com os dois servidores em NixOS, o que mudou de verdade não foi a confiabilidade. Os dois já eram estáveis antes. O que mudou foi que **eu sei o que está rodando**. Cada porta aberta tem um comentário do lado dizendo por quê. Cada `sysctl` tem a história do bug que ele evita. Quando o `nix flake update` quebra um hash, eu sei em trinta segundos o que aconteceu e como consertar.

E a reinstalação, que era a coisa que eu mais evitava, virou um `just install` que eu vou rodar quando o NVMe de 2 TB chegar. Os uids estão pinados, o disko está escrito, os segredos estão no git. O que sobra é restaurar `/var/lib` de um backup.

Ainda tem fila: colmena pra fazer deploy dos dois com um comando, impermanence pra provar que nada importante vive fora do repo, e a decisão sobre a GPU do bifrost. Mas isso é assunto pra quando tiver um caso real, como sempre.
