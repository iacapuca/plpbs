---
title: "Homelab em NixOS, parte 3: yggdrasil, Home Assistant numa VM e a VLAN IoT com systemd-networkd"
date: 2026-09-09T12:00:00-03:00
draft: true
tags:
  - Homelab
  - NixOS
  - Home Assistant
  - Networking
  - libvirt
---

O yggdrasil é o servidor de automação. Ele roda Caddy, AdGuard e Tailscale como serviços nativos, e o Home Assistant roda numa **VM de Home Assistant OS** em cima de libvirt/KVM. No Debian antigo tudo isso era uma stack de Docker, incluindo o próprio Home Assistant em container. Esta parte é sobre a VM e, principalmente, sobre a rede que ela precisa, porque foi onde eu mais apanhei.

<!--more-->

{{< callout type="info" >}}
Parte 3 da série sobre meu homelab em NixOS. Antes: [a estrutura do repo](/blog/2026/09/09/homelab-nixos-parte-1-repo-e-fluxo/) e [a stack de mídia do bifrost](/blog/2026/09/09/homelab-nixos-parte-2-bifrost-media-stack/).
{{< /callout >}}

## Por que uma VM, e não o container

Home Assistant em container funciona, mas você perde o Supervisor, e com ele os add-ons. No meu caso os add-ons carregam coisa que não é opcional: o Matter server, o `rtl_433` que decodifica rádio de 915 MHz de alguns aparelhos, um servidor MCP pra falar com o HA. Recriar cada um deles como container separado é trabalho que o HAOS já faz.

O HAOS é um appliance: uma imagem que espera ser o sistema inteiro. A maneira mais limpa de rodar isso num servidor que também faz outras coisas é uma VM. E o NixOS torna a parte do host trivial:

```nix {filename="hosts/yggdrasil/virtualisation.nix"}
{ pkgs, ... }:
{
  virtualisation.libvirtd = {
    enable = true;
    onBoot = "start";
    onShutdown = "shutdown";
    qemu.swtpm.enable = true;
  };

  users.users.plpbs.extraGroups = [ "libvirtd" ];
  environment.systemPackages = with pkgs; [ virt-manager ];
}
```

`onBoot = "start"` sobe a VM junto com o host. `swtpm` dá um TPM virtual, que o HAOS quer pra UEFI com Secure Boot. O OVMF (o firmware UEFI) já vem habilitado por padrão no 26.05; a opção antiga `qemu.ovmf.enable` foi removida e meu primeiro build quebrou nela.

## A rede que a VM precisa

Aqui começa o problema de verdade. O Home Assistant descobre dispositivos na rede (Midea, ESPHome, Tuya) por **broadcast UDP**. Broadcast não atravessa sub-rede. Meus aparelhos de IoT estão numa VLAN separada (`192.168.20.0/24`, tag 20), e a VM, se ficasse só na LAN, nunca veria nenhum deles. Testei mandar um broadcast direcionado pra `192.168.20.255` a partir da LAN e o gateway descarta.

Então a VM precisa ter **duas pernas**: uma na LAN e uma na VLAN de IoT. E pra isso o host precisa oferecer duas bridges:

```text
                 enp2s0f1 (trunk: untagged LAN + VLAN 20)
                     │
                     ▼
                    br0 ──────────────── host: 192.168.1.128 (estático)
                     │  \
                     │   └─ vnet0 ──► VM enp1s0: 192.168.1.125
                     │
                 vlan-iot (id 20, sem IP)
                     │
                     ▼
                    br20 ──────────────── host: nenhum endereço (L2 puro)
                       \
                        └─ vnet1 ──► VM enp7s0: 192.168.20.114
```

Em Nix, com systemd-networkd:

```nix {filename="hosts/yggdrasil/bridge.nix"}
{ lib, ... }:
let
  iotVlanId = 20;
in
{
  networking.networkmanager.enable = lib.mkForce false;
  networking.useNetworkd = true;
  services.resolved.enable = true;

  boot.kernel.sysctl = {
    "net.bridge.bridge-nf-call-iptables" = 0;
    "net.bridge.bridge-nf-call-ip6tables" = 0;
  };

  systemd.network = {
    enable = true;

    netdevs."br0".netdevConfig = { Kind = "bridge"; Name = "br0"; };

    netdevs."vlan-iot" = {
      netdevConfig = { Kind = "vlan"; Name = "vlan-iot"; };
      vlanConfig.Id = iotVlanId;
    };

    netdevs."br20".netdevConfig = { Kind = "bridge"; Name = "br20"; };

    networks."10-enp2s0f1-bridged" = {
      matchConfig.Name = "enp2s0f1";
      networkConfig.Bridge = "br0";
    };

    networks."20-br0" = {
      matchConfig.Name = "br0";
      networkConfig = {
        DHCP = "no";
        Address = "192.168.1.128/24";
        Gateway = "192.168.1.1";
        DNS = "192.168.1.128";
      };
      linkConfig.MACAddress = "aa:bb:cc:dd:ee:ff"; # o MAC da NIC física
      vlan = [ "vlan-iot" ];
    };

    networks."30-vlan-iot-bridged" = {
      matchConfig.Name = "vlan-iot";
      networkConfig.Bridge = "br20";
    };

    networks."40-br20" = {
      matchConfig.Name = "br20";
      networkConfig = {
        DHCP = "no";
        LinkLocalAddressing = "no";
        IPv6AcceptRA = false;
      };
      linkConfig.RequiredForOnline = "no";
    };
  };
}
```

Lendo de cima pra baixo:

- **`common.nix` liga o NetworkManager**, então aqui eu desligo com `mkForce` e troco por `systemd-networkd`. Bridges e VLANs em networkd são declarativas e previsíveis; em NetworkManager eu nunca confiei.
- **`br0` é a bridge da LAN.** A NIC física entra nela, o IP do host mora nela, e a VM pluga nela via `vnet0`. A bridge copia o MAC da NIC pra que a reserva de DHCP no UniFi continue reconhecendo a máquina.
- **`vlan-iot`** é a VLAN 20 "levantada" de cima da `br0`. Como `br0` não faz filtro de VLAN, os frames com tag chegam intactos na porta local e esse netdev tira a tag.
- **`br20`** é a bridge onde a `vlan-iot` entra e onde a VM pluga o segundo NIC. O host **não tem endereço** nela. É só um fio entre o uplink com tag e o `vnet1` da VM. Isso evita que o yggdrasil ganhe uma segunda rota default e comece a rotear assimétrico.
- **`RequiredForOnline = "no"`** na `br20`, senão o `systemd-networkd-wait-online` fica esperando uma interface que nunca vai ter IP e segura o boot.

Do lado da VM, a interface de IoT tem métrica de rota bem maior que a da LAN, então o tráfego normal continua saindo pela LAN e a perna de IoT só serve pra descoberta e pra falar com os aparelhos.

## As duas coisas que quebraram

### O IP do host mudou sozinho

A primeira versão da `br0` usava DHCP, com reserva no UniFi. Funcionou até o primeiro `nixos-rebuild switch` que mexeu na rede. Reiniciar o `systemd-networkd` descarta o lease que estava na memória, então a `br0` faz um DISCOVER novo **enquanto ainda segura o `.128`**. O gateway vê o endereço em uso e entrega outro (`.174`, no meu caso).

Em qualquer outra máquina isso seria irritante. Aqui foi fatal, porque o AdGuard faz bind de DNS no endereço literal `192.168.1.128`. IP mudou, DNS caiu, casa inteira sem resolver nome. Daí a `br0` virou estática. A reserva no UniFi continua lá, mas por documentação.

### A VM "conectava" e nada passava

Esse foi o pior. Depois de subir a `br20`, a VM pegava IP na VLAN 20, o ARP resolvia, o `ip neigh` mostrava os vizinhos. E nenhum pacote ICMP ou TCP chegava. Nada entre o `vnet1` e a `enp2s0f1`. Do ponto de vista do HA, os aparelhos existiam e estavam mudos.

A causa era o **Docker**, que ainda estava instalado pra eu terminar de limpar imagens antigas. Docker carrega o módulo `br_netfilter`, e com ele o tráfego que atravessa uma bridge Linux passa a ser processado pela cadeia `FORWARD` do iptables do host. A do Docker e a do Tailscale. O ARP é camada 2 e passa; o IPv4 unicast cai nas regras e morre.

O sysctl `net.bridge.bridge-nf-call-iptables = 0` desliga esse comportamento. E depois eu tirei o Docker do host de vez, porque não sobrava mais nada pra ele fazer. O sysctl ficou no arquivo como seguro: se um dia voltar Podman ou algo que carregue `br_netfilter`, eu não quero redescobrir isso.

{{< callout type="warning" >}}
Mudança de rede num servidor sem monitor é onde generations valem ouro. Apliquei o `bridge.nix` com `nixos-rebuild test` (ativa sem virar default de boot) e um reboot agendado. Se eu perdesse o SSH, a máquina reiniciava na geração anterior, com NetworkManager, e eu tentava de novo.
{{< /callout >}}

## USB passthrough: o estado que não está no repo

Dois dongles USB são passados pra VM: um RTL-SDR (o rádio de 915 MHz do `rtl_433`) e um adaptador Bluetooth (pra comissionar dispositivo Matter pela interface do HA, sem depender do celular). Isso foi feito com `virsh`:

```bash
virsh attach-device homeassistant --file host-state/usb-rtl-sdr.xml --live --config
virsh attach-device homeassistant --file host-state/usb-ble.xml --live --config
```

O `--config` persiste no XML do domínio, então sobrevive a reboot. Mas o XML do domínio mora em `/var/lib/libvirt/qemu/`, **não no repo**. Se eu recriar a VM, os dois somem sem erro nenhum: o add-on do SDR diz "no device found" e o Matter perde BLE silenciosamente.

Não tenho solução declarativa pra isso ainda (NixVirt é o candidato). O que fiz foi o mínimo honesto: os XMLs vivem em `host-state/` no repo, com um README dizendo "se recriou a VM, rode isso". Um diretório inteiro pra admitir que algo não é gerenciado é melhor do que fingir que é.

Um detalhe dos XMLs: eles identificam o dispositivo por `vendor:product`, não por `bus:device`. O número de bus muda quando você tira e põe o dongle; o vendor ID não.

### O kernel do host também quer o dongle

O RTL-SDR é, fisicamente, um sintonizador de TV digital. O kernel reconhece e carrega `dvb_usb_rtl28xxu`. O libvirt, com `managed='yes'`, desanexa do host quando a VM sobe, e parece tudo certo. Até o dia em que o dispositivo re-enumerou com a VM já rodando, o host ganhou a corrida, e o `rtl_433` dentro da VM passou a crashar a cada 5 segundos. O log do add-on não dava nenhuma pista. O `hostdev` estava lá no XML, a VM estava de pé, o device 0 simplesmente não existia pra ela.

```nix {filename="hosts/yggdrasil/virtualisation.nix"}
  boot.blacklistedKernelModules = [
    "dvb_usb_rtl28xxu"
    "rtl2832"
    "rtl2832_sdr"
  ];
```

Blacklist elimina a corrida em vez de torcer pro libvirt ganhar.

## Firewall: só o que escuta

O Debian tinha aberto 8123, 1883, 10200, 10300, 5540, 19999 e a faixa do RustDesk. Nenhuma dessas portas tinha mais nada atrás: tudo isso migrou pra dentro da VM, que responde no próprio IP. Depois de conferir com `ss` o que realmente escutava no host, sobrou:

```nix {filename="hosts/yggdrasil/default.nix"}
  networking.firewall = {
    allowedTCPPorts = [ 53 80 443 ];
    allowedUDPPorts = [ 53 443 ];
  };
```

53 é o AdGuard, 80 e 443 o Caddy, e UDP 443 é HTTP/3. O Caddy já servia QUIC, mas a porta nunca tinha sido aberta, então os clientes caíam pra TCP em silêncio. A porta 22 vem do `common.nix` e a interface web do AdGuard abre no módulo dele.

Na [parte 4](/blog/2026/09/09/homelab-nixos-parte-4-caddy-adguard-sops/), o que roda por cima dessa rede: Caddy com certificado via Cloudflare, AdGuard como DNS da casa, e como os segredos entram sem passar pelo git.
