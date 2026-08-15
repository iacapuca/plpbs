---
title: "Mesmo domínio, dentro ou fora de casa: Tailscale, DNS e Caddy no meu homelab"
date: 2026-07-28
tags:
  - Homelab
  - Tailscale
  - Caddy
  - DNS
  - Networking
---

Eu queria acessar os serviços do meu homelab (Home Assistant, Grafana, etc.) com o **mesmo hostname** estando em casa ou longe — sem VPN full-tunnel, sem abrir porta no roteador, e com certificado válido nos dois casos. Nada disso passa por deixar algo realmente público na internet: o acesso continua restrito a quem está na minha [tailnet](https://tailscale.com/) (minha rede privada do Tailscale). O que muda é só o caminho que o tráfego percorre até chegar lá.

<!--more-->

## A ideia

O truque combina três coisas do Tailscale que sozinhas fazem pouco, mas juntas resolvem o problema todo: **Split DNS**, um **resolver próprio** (o AdGuard, que já rodo como DNS da rede local) e um **subnet router**.

```text
  dispositivo da tailnet (em casa ou longe)
             │
             │  "ha.pedroluiz.com?"
             ▼
  Split DNS (config no Tailscale):
  "pra pedroluiz.com, pergunte a 100.x.y.z"
             │
             ▼
  AdGuard (no mesmo host do Caddy)
  responde sempre: 192.168.1.50
             │
             ▼
  dispositivo tenta rotear até 192.168.1.50
             │
      ┌──────┴───────┐
      │               │
  em casa         longe de casa
  (LAN direta)    (via subnet router
                   anunciado pela
                   própria máquina)
      │               │
      └───────┬───────┘
              ▼
            Caddy
   (mesmo Host, mesmo cert DNS-01)
```

A resposta do DNS é **sempre a mesma** — um IP da minha LAN (`192.168.1.50`), nunca muda dependendo de onde eu estou. O que muda é só o *caminho* até esse IP: direto, se eu já tô na rede; via túnel do Tailscale, se eu tô longe. Isso só funciona porque a mesma máquina que roda o Caddy também **anuncia a sub-rede `192.168.1.0/24` pra tailnet** (subnet router) — sem isso, um IP privado da LAN seria um beco sem saída pra quem está longe de casa.

## Passo 1 — Caddy com certificado real via DNS-01

Continua sendo o Caddy que centraliza o roteamento por hostname, e continuo usando desafio **DNS-01** com a API da Cloudflare pra tirar certificado do Let's Encrypt — isso importa especialmente aqui, porque o desafio HTTP-01 tradicional exige que o Caddy seja alcançável publicamente na porta 80, e não é: só a tailnet chega nele.

```caddyfile {filename="/etc/caddy/Caddyfile"}
{
    email voce@pedroluiz.com
}

# Snippet reutilizável de DNS-01 via Cloudflare
(cf) {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
        resolvers 1.1.1.1
    }
}

ha.pedroluiz.com {
    import cf
    encode gzip
    reverse_proxy 127.0.0.1:8123
}

grafana.pedroluiz.com {
    import cf
    encode gzip
    reverse_proxy 127.0.0.1:3000
}
```

O DNS-01 só precisa provar posse do domínio via um registro TXT temporário (`_acme-challenge.ha.pedroluiz.com`) — não importa se o host é alcançável de fora ou não. Por isso dá pra ter certificado real do Let's Encrypt mesmo num serviço que só existe dentro da tailnet.

{{< callout type="info" >}}
Repare que não precisei de nenhum bloco especial pra "modo privado" — o Caddy nem sabe que esse hostname só é alcançável via tailnet. Ele só escuta em todas as interfaces (`0.0.0.0:443`), o que já inclui a interface do Tailscale automaticamente.
{{< /callout >}}

## Passo 2 — Split DNS: mandando a tailnet inteira perguntar pro meu AdGuard

Já tenho o AdGuard Home rodando como DNS da rede local, com um **DNS rewrite** fazendo `*.pedroluiz.com` resolver pro IP da LAN onde o Caddy escuta:

```text
Domínio          Resposta
*.pedroluiz.com   192.168.1.50
```

Isso já resolve pra quem está em casa (dispositivos usam o AdGuard como DNS via DHCP). O problema é fora de casa: o celular no 4G não tem como perguntar pro AdGuard, que só existe na minha LAN.

É pra isso que serve o **Split DNS** do Tailscale — não confundir com um DNS rewrite comum, é uma configuração específica do admin console (aba **DNS** → **Nameservers** → **Add nameserver**, com a opção **"Restrict to domain"**). Configuro:

```text
Domínio restrito: pedroluiz.com
Nameserver:       100.101.102.103   (IP da tailnet da própria máquina do AdGuard)
```

A partir daqui, **qualquer** dispositivo da minha tailnet — em casa ou longe — para de usar o DNS público pra `*.pedroluiz.com` e passa a perguntar direto pro meu AdGuard, via túnel do Tailscale. A resposta é sempre a mesma que o AdGuard já dava pra quem está em casa: `192.168.1.50`.

{{< callout type="info" >}}
Isso é diferente de simplesmente publicar um registro A público apontando pro IP da tailnet. Com Split DNS, quem **não** está na minha tailnet nem consegue *perguntar* — a consulta não sabe pra onde ir. Com um registro público comum, qualquer um consegue perguntar e recebe uma resposta (só não consegue chegar lá). É uma camada a mais de esconder, não só de bloquear o acesso.
{{< /callout >}}

## Passo 3 — Subnet router: fazendo o IP da LAN ser alcançável de longe

Ainda falta uma peça: `192.168.1.50` é um IP privado da minha LAN. Um dispositivo longe de casa recebe essa resposta do Split DNS, mas por padrão não tem como *rotear* até um IP `192.168.1.x` — isso não existe fora da rede física de casa.

A solução é fazer a própria máquina que roda o Caddy (e o AdGuard) anunciar a sub-rede inteira pra tailnet, como **subnet router**:

```bash
sudo tailscale up --advertise-routes=192.168.1.0/24
```

Depois preciso aprovar essa rota no admin console do Tailscale (**Machines** → a máquina → **Subnets** → aprovar `192.168.1.0/24`). A partir daí, qualquer dispositivo da tailnet ganha uma rota pra LAN inteira, usando essa máquina como gateway — e o IP que o Split DNS devolveu (`192.168.1.50`) vira alcançável de qualquer lugar.

{{< callout type="warning" >}}
Isso expõe a sub-rede `192.168.1.0/24` inteira pra tailnet, não só o host do Caddy — vale revisar as [ACLs do Tailscale](https://tailscale.com/kb/1018/acls/) pra restringir quem consegue alcançar o quê dentro dessa rota, em vez de deixar todo dispositivo da tailnet enxergar a LAN de casa inteira.
{{< /callout >}}

Juntando os dois: Split DNS garante que a tailnet inteira pergunte pro meu DNS (não pro público), e o subnet router garante que a resposta (um IP de LAN) seja alcançável de qualquer lugar. Nenhum dos dois sozinho resolveria — Split DNS sem subnet router te dá o IP certo mas sem rota até ele; subnet router sem Split DNS te dá a rota mas a consulta DNS ainda sairia pro público (que nem sabe que `192.168.1.50` existe).

## E o MagicDNS?

Vale separar uma coisa: isso aqui **não é** o MagicDNS do Tailscale. O MagicDNS dá um hostname automático tipo `minha-maquina.tail-net-name.ts.net` pra cada dispositivo da tailnet, sem eu escolher o nome — ótimo pra acesso rápido e ad-hoc (`ssh proxmox`, `tailscale ip -4 grafana`). O que fiz acima é diferente: uso meu **próprio domínio**, com os hostnames que eu quero, resolvidos pelo meu próprio AdGuard via Split DNS — dá mais controle (o mesmo nome funciona pra humano digitar no navegador e pra automação), ao custo de eu ter que manter o AdGuard e as rotas no ar.

Os dois convivem numa boa: MagicDNS pra mexer na infra por trás, domínio próprio pra tudo que tem uma interface web na frente.

## Testando

```bash
# Confirma o IP que o Tailscale atribuiu à máquina do AdGuard/Caddy
tailscale ip -4

# Resolve via Split DNS — deve vir sempre 192.168.1.50,
# esteja eu em casa ou longe (roda num dispositivo da tailnet)
dig +short ha.pedroluiz.com

# Confirma que a rota da sub-rede está ativa
tailscale status

# Acessa o serviço — certificado real via DNS-01, caminho local ou via subnet router
curl https://ha.pedroluiz.com
```

## Próximos passos

Esse padrão cobre tudo que eu quero manter privado, restrito à minha tailnet — que hoje é praticamente tudo. Se um dia eu quiser deixar alguma coisa *de verdade* pública (sem exigir Tailscale de quem acessa), o mecanismo é outro — Cloudflare Tunnel, port forward tradicional, ou Cloudflare Access na frente — e fica pra um próximo artigo, quando eu tiver um caso real pra isso. Também quero detalhar as ACLs do Tailscale, que uso pra restringir quais dispositivos da tailnet falam com quais máquinas, em vez de deixar tudo aberto entre todos os nós.
