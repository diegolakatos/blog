---
date : '2026-02-18T09:50:09Z'
draft : false
title : 'Migrando do K3s para o Talos Linux - Parte 1'
Description : "Como migrar um cluster k3s singlenode para um cluster multinode usando o Talos Linux"
DisableComments : "true"
Tags : "[Kubernetes, Containers, HomeLab, Linux]"
Categories : [IaC]
language : pt-br
---
Sou um grande fã de homelabs e self-hosted. Como profissional de TI, acredito ser importante ter um local onde eu possa testar e aprender coisas novas, especialmente aquelas que não estão diretamente relacionadas ao meu trabalho atual. Também gosto de manter meus dados pessoais em sigilo e não dependo de grandes empresas que constantemente puxam o tapete seus clientes. A [*merdificação*](https://pt.wikipedia.org/wiki/Merdifica%C3%A7%C3%A3o) veio para ficar e só tende a piorar, sem mencionar o uso de nossos dados para treinar modelos de IA.

No meu laboratório doméstico, a maioria dos serviços onde armazeno dados valiosos, como fotos, possui sua própria máquina virtual. Essa máquina virtual pode usar contêineres para executar a aplicação (usando um arquivo docker-compose) ou a aplicação pode ser instalada por outro método. As VMs são implantadas em um cluster Proxmox de dois nós.

Todos os serviços possuem seus próprios certificados gerados com o Let's Encrypt e são acessíveis através do Tailscale.

Para serviços sem estado ou que não armazenam dados muito importantes, atualmente utilizo um cluster k3s de um único nó. O K3s é uma distribuição Kubernetes leve e otimizada para *edge computing*. Ele funciona muito bem e eu o recomendo fortemente para quem deseja usar o Kubernetes sem a complexidade de configurar um cluster mais complexo, lembrando que o K3s também pode ser usado em implantações com vários nós.

Embora eu seja um grande fã da filosofia K.I.S.S. (Keep It Simple, Stupid - Mantenha Simples, Idiota) no meu trabalho, gosto de explorar cenários mais complexos para aprender coisas novas, e o Talos Linux é um desses exemplos. O Talos é uma distribuição criada para executar clusters Kubernetes. Ele é minimalista e seguro por natureza, e sua configuração é feita por meio de arquivos de configuração e da ferramenta de linha de comando `talosctl` (ssh nem funciona). Isso significa que posso armazenar toda a minha configuração em um repositório Git, permitindo uma abordagem GitOps e tempos de reconstrução mais rápidos, se necessário.

No meu cenário atual, meu servidor K3s expõe os serviços usando o balanceador de carga integrado e o *ingress controler* do Traefik. Os aplicativos são implantados usando o ArgoCD. Os aplicativos que precisam gravar em disco podem fazer isso usando o hostPath (o que não é o ideal, mas funciona bem o suficiente neste caso). O monitoramento é feito usando uma combinação de Uptime Kuma e netdata.

Os objetivos deste "projeto" são:
- Ter um cluster de alta disponibilidade usando Talos Linux
- Monitoramento adequado usando Prometheus + Loki, que será usado para monitorar todo o meu laboratório doméstico
- Usar Longhorn como CSI
- Substituir o Ingress por API Gateway
- Migrar todos os aplicativos em execução no k3s
- Manter tudo funcionando como antes

Neste primeiro post, demonstrarei como baixei o Talos Linux e criei as máquinas virtuais no meu cluster Proxmox. Vou me concentrar nas partes específicas do Talos, portanto, quaisquer ações realizadas no Proxmox não serão mostradas, a menos que sejam muito específicas.

Primeiramente, precisamos baixar a ISO do Talos Linux. Para isso, precisamos acessar a [Image Factory](https://factory.talos.dev/).
O processo aqui não é tão simples quanto em outras distribuições, onde basta escolher a arquitetura ou a variante. O Talos permite uma grande variedade de personalizações desde o início e, embora muitas das opções possam ser alteradas posteriormente, precisamos prestar atenção a alguns detalhes, caso contrário, não conseguiremos inicializar o sistema. Felizmente, a página de download é muito bem documentada e fácil de navegar.
A primeira coisa a fazer é escolher o tipo de hardware. Como estou usando o Proxmox, escolho "Cloud server".

{{< lightbox src="/images/talos-pt1/1.png" alt="1" width="800" class="full-bleed">}}

Agora escolho a versão, neste caso é a 1.12.2.

{{< lightbox src="/images/talos-pt1/2.png" alt="1" width="800" class="full-bleed">}}

Como escolhi "Cloud server", agora preciso escolher qual tipo de nuvem usar. A lista aqui é bastante abrangente, com muitos dos principais provedores de nuvem. No meu caso, escolherei "Nocloud".

{{< lightbox src="/images/talos-pt1/3.png" alt="1" width="800" class="full-bleed">}}

Escolha a arquitetura, no meu caso amd64, eu não queria usar o Secure Boot, mas se for o caso, confira [este ótimo artigo de Mauricio Teixeira](https://mteixeira.wordpress.com/2026/02/01/booting-talos-on-a-proxmox-vm/)

{{< lightbox src="/images/talos-pt1/4.png" alt="1" width="800" class="full-bleed">}}

Você pode pré-baixar extensões de sistema, que são usadas para expandir os recursos do cluster, como carregar firmwares adicionais ou um ambiente de execução personalizado. Como o sistema de arquivos Talos é *read-only* e imutável, você precisa instalar essas extensões apenas durante uma atualização ou instalação inicial. No meu caso, vou instalá-las em uma etapa posterior.

{{< lightbox src="/images/talos-pt1/5.png" alt="1" width="800" class="full-bleed">}}

Na página de personalização, não precisei alterar nada.

{{< lightbox src="/images/talos-pt1/7.png" alt="1" width="800" class="full-bleed">}}

Por fim, você encontrará o endereço para baixar a imagem, que também fornece endereços e instruções para usar a imagem personalizada para atualizar um cluster existente.

{{< lightbox src="/images/talos-pt1/8.png" alt="1" width="800" class="full-bleed">}}

Você pode baixar a imagem ISO pelo link fornecido, mas como estou usando o Proxmox, acho mais fácil fazer o upload diretamente para o Proxmox. [Este artigo mostra como fazer isso](https://www.thomas-krenn.com/en/wiki/Proxmox_upload_ISO_image). Com a ISO carregada, basta criar uma máquina virtual (ou várias, neste caso) e iniciá-las. Após a inicialização, você deverá ver algo como isto:

{{< lightbox src="/images/talos-pt1/9.png" alt="1" width="800" class="full-bleed">}}

Com isso, estamos prontos para configurar e inicializar o cluster, o que será feito na parte 2. Até breve!