# SITER — Bootstrap Geral Ubuntu Server

Bootstrap comum para VMs do SITER, utilizando Ubuntu Server 26.04 LTS 64 bits.

**Objetivo:** preparar uma VM básica, segura e reutilizável como imagem-base para snapshots.

## Bloco 1 — Criar SIG e ativar firewall

Acesse a VM como `root`.

Este bloco cria o usuário `sig`, configura SSH e sudo e ativa o firewall, permitindo somente conexões de entrada pela porta SSH 22, sem restrição por IP.

O login de root permanece habilitado até o bloco 3.

`````bash
clear
echo "===============INÍCIO==============="
echo "===============Bloco 1=============="

# VM: VM-base SITER
# Módulo: Bootstrap geral
# Repositório: Não aplicável
# Usuário: root
# Objetivo: Criar sig e liberar somente SSH na entrada
# Natureza: ALTERAÇÃO

if (
    set -e
    test "$(id -un)" = root
    test -s /root/.ssh/authorized_keys

    id sig >/dev/null 2>&1 || adduser --disabled-password --gecos "" sig
    usermod -aG sudo sig

    install -d -m 700 -o sig -g sig /home/sig/.ssh
    install -m 600 -o sig -g sig \
        /root/.ssh/authorized_keys \
        /home/sig/.ssh/authorized_keys

    echo 'sig ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/90-siter-sig
    chmod 440 /etc/sudoers.d/90-siter-sig
    visudo -cf /etc/sudoers.d/90-siter-sig

    if ! command -v ufw >/dev/null 2>&1; then
        apt-get update
        apt-get install -y ufw
    fi

    ufw --force reset
    ufw default deny incoming
    ufw default deny routed
    ufw default allow outgoing
    ufw allow 22/tcp
    ufw --force enable

    id sig
    ufw status verbose

); then
    echo "RESULTADO=SUCESSO"
else
    echo "RESULTADO=ERRO"
fi

echo "===============FIM=================="
echo "===============Bloco 1=============="
`````

## Bloco 2 — Testar acesso com SIG

**Abra uma nova sessão PuTTY como `sig`.** Mantenha a sessão de root aberta.

Execute o bloco para confirmar o acesso SSH e o funcionamento do sudo.

`````bash
clear
echo "===============INÍCIO==============="
echo "===============Bloco 2=============="

# VM: VM-base SITER
# Módulo: Bootstrap geral
# Repositório: Não aplicável
# Usuário: sig
# Objetivo: Validar acesso SSH e sudo
# Natureza: SOMENTE LEITURA

if [ "$(id -un)" = sig ] && sudo -n true; then
    echo "USUARIO=$(whoami)"
    echo "HOSTNAME=$(hostname)"
    echo "SUDO=FUNCIONAL"
    echo "RESULTADO=SUCESSO"
else
    echo "RESULTADO=ERRO"
fi

echo "===============FIM=================="
echo "===============Bloco 2=============="
`````

## Bloco 3 — Desabilitar acesso SSH de root

Após confirmar o sucesso do bloco 2, volte à sessão PuTTY de `root`.

Mantenha a sessão de `sig` aberta.

`````bash
clear
echo "===============INÍCIO==============="
echo "===============Bloco 3=============="

# VM: VM-base SITER
# Módulo: Bootstrap geral
# Repositório: Não aplicável
# Usuário: root
# Objetivo: Desabilitar login SSH de root
# Natureza: ALTERAÇÃO

if [ "$(id -un)" != root ] || ! id sig >/dev/null 2>&1; then
    echo "RESULTADO=ERRO | Usuário incorreto ou sig inexistente"
else
    ARQUIVO=/etc/ssh/sshd_config.d/00-siter-no-root.conf

    printf 'PermitRootLogin no\n' > "$ARQUIVO"

    if sshd -t &&
       sshd -T | grep -qx 'permitrootlogin no' &&
       systemctl reload ssh; then
        echo "SSH_ROOT=DESABILITADO"
        echo "RESULTADO=SUCESSO"
    else
        rm -f "$ARQUIVO"
        echo "RESULTADO=ERRO"
    fi
fi

echo "===============FIM=================="
echo "===============Bloco 3=============="
`````

## Bloco 4 — Atualizar sistema e reiniciar

Execute como `sig`.

Atualiza o Ubuntu, instala os pacotes básicos comuns ao SITER e agenda a reinicialização da VM.

O firewall permanece configurado para permitir somente SSH na entrada.

**A conexão PuTTY será interrompida durante o reboot.**

`````bash
clear
echo "===============INÍCIO==============="
echo "===============Bloco 4=============="

# VM: VM-base SITER
# Módulo: Bootstrap geral
# Repositório: Não aplicável
# Usuário: sig
# Objetivo: Atualizar Ubuntu, instalar pacotes básicos e reiniciar
# Natureza: ALTERAÇÃO

if [ "$(id -un)" != sig ]; then
    echo "RESULTADO=ERRO | Execute como sig"
elif (
    set -e

    sudo apt-get update
    sudo DEBIAN_FRONTEND=noninteractive apt-get upgrade -y

    sudo DEBIAN_FRONTEND=noninteractive apt-get install -y \
        ca-certificates curl git nano

    sudo ufw status verbose

    sudo reboot

); then
    echo "RESULTADO=SUCESSO"
else
    echo "RESULTADO=ERRO"
fi

echo "===============FIM=================="
echo "===============Bloco 4=============="
`````

## Bloco 5 — Recuperação e verificação final

Após o reboot, abra uma nova sessão PuTTY como `sig`.

Execute o bloco para verificar o estado geral da VM.

`````bash
clear
echo "===============INÍCIO==============="
echo "===============Bloco 5=============="

# VM: VM-base SITER
# Módulo: Bootstrap geral
# Repositório: Não aplicável
# Usuário: sig
# Objetivo: Verificação geral após reboot
# Natureza: SOMENTE LEITURA

if (
    set -e

    test "$(id -un)" = sig

    echo "========== SISTEMA =========="
    grep PRETTY_NAME /etc/os-release
    uname -r
    uptime -p

    echo "========== USUÁRIO E SUDO =========="
    id
    sudo -n true
    echo "SUDO=FUNCIONAL"

    echo "========== SSH =========="
    systemctl is-active ssh
    sudo /usr/sbin/sshd -T | grep -x 'permitrootlogin no'

    echo "========== FIREWALL =========="
    sudo ufw status verbose
    sudo ufw status | grep -q 'Status: active'
    sudo ufw status | grep -qE '^22/tcp +ALLOW IN +Anywhere'

    echo "========== PACOTES =========="
    dpkg -s ca-certificates curl git nano >/dev/null
    echo "PACOTES=INSTALADOS"

); then
    echo "RESULTADO=SUCESSO"
else
    echo "RESULTADO=ERRO"
fi

echo "===============FIM=================="
echo "===============Bloco 5=============="
`````

## Snapshot e reutilização

Após a conclusão dos cinco blocos, a VM está preparada com o bootstrap comum do SITER.

O snapshot pode ser criado antes de instalar programas ou aplicar configurações específicas de qualquer módulo.

Ao criar novas VMs a partir do snapshot, garantir que cada instância tenha identidade própria, especialmente hostname, machine-id e chaves de host SSH. Verificar se o provedor realiza essa individualização automaticamente.

O bootstrap específico de cada VM será realizado separadamente, conforme a necessidade de seu módulo.

**Fim do bootstrap geral.**

---

# Bootstrap específico — SITER Storage

**Sistema operacional:** Ubuntu Server 26.04 LTS 64 bits  
**Usuário operacional:** `sig`  
**Pré-requisito:** bootstrap geral do SITER concluído.

Este procedimento prepara a infraestrutura básica da VM Storage. A configuração dos serviços de publicação e das integrações com Drone e OGC pertence ao desenvolvimento do módulo.

## 1. Rede e firewall

A Storage mantém o firewall ativo, com as seguintes regras:

| Serviço | Acesso |
|---|---|
| SSH — TCP 22 | Qualquer IP |
| Syncthing — TCP 22000 | IP público e rede privada |
| GUI Syncthing — TCP 858 | Somente localhost, via túnel SSH |
| Demais portas de entrada | Bloqueadas |

O compartilhamento de arquivos entre módulos utilizará exclusivamente a rede privada do SITER.

As regras específicas de transferência e compartilhamento serão adicionadas conforme a implementação dos respectivos serviços.

## 2. Syncthing

Acesse como `sig`.

Instale e habilite o serviço:

`````bash
sudo apt-get update
sudo apt-get install -y syncthing
sudo systemctl enable --now syncthing@sig.service
`````

Configuração adotada:

| Parâmetro | Valor |
|---|---|
| GUI | `127.0.0.1:858` |
| Sincronização | TCP 22000 |
| Discovery público e local | Desabilitados |
| Relay | Desabilitado |
| NAT | Desabilitado |
| Usuário do serviço | `sig` |
| Diretório de configuração | `/home/sig/.local/state/syncthing` |

A GUI deve permanecer vinculada ao localhost. A porta 858 não deve ser liberada no firewall.

Como a porta 858 é inferior a 1024, configure a permissão necessária para o serviço utilizá-la sem executar como root:

`````bash
sudo mkdir -p /etc/systemd/system/syncthing@sig.service.d

printf '%s\n' \
    '[Service]' \
    'AmbientCapabilities=CAP_NET_BIND_SERVICE' \
    'CapabilityBoundingSet=CAP_NET_BIND_SERVICE' \
    | sudo tee /etc/systemd/system/syncthing@sig.service.d/10-siter-gui.conf

sudo systemctl daemon-reload
sudo systemctl restart syncthing@sig.service
`````

**Túnel SSH no PuTTY:**

- Source port: `11858`
- Destination: `127.0.0.1:858`
- Tipo: Local

Acesse a interface pelo navegador:

`http://127.0.0.1:11858`

Para sincronização direta, utilize o IP público da VM na porta TCP 22000. Autorize os dispositivos participantes no Syncthing.

### Temas coloridos

Instalar os sete temas personalizados baseados no tema `dark` do Syncthing:

| Tema | Body | Navbar |
|---|---|---|
| Azul | `#003` | `#004` |
| Vermelho | `#300` | `#400` |
| Verde | `#030` | `#040` |
| Amarelo | `#220` | `#330` |
| Ciano | `#022` | `#033` |
| Roxo | `#202` | `#303` |
| Laranja | `#310` | `#420` |

Utilizar como referência o script `create_syncthing_custom_theme.v2.sh` e seu arquivo `.vars`, ajustando:

`````bash
BASE_THEME_NAME="dark"
SET_CUSTOM_THEME_AS_ACTIVE="no"
ACTIVE_THEME_NAME="Azul"
SYNCTHING_GUI_URL="http://127.0.0.1:858"
SYNCTHING_CONFIG_DIR="/home/sig/.local/state/syncthing"
RESTART_SYNCTHING="yes"
`````

O script e seu `.vars` devem ser preservados junto à documentação de instalação. Esses parâmetros não substituem os arquivos originais do instalador.

Os temas foram instalados na Storage, sem ativação automática. A seleção será feita manualmente pela GUI.

## 3. Volume persistente

A Storage utiliza um volume adicional independente do disco do sistema operacional.

Configuração aplicada à VM Storage:

| Propriedade | Valor |
|---|---|
| Volume | `storage-volume` |
| Capacidade inicial | 50 GB |
| Sistema de arquivos | ext4 |
| Ponto de montagem | `/srv/siter/storage` |
| UUID | `54adace4-48db-4dd9-a6ad-ed32e60f0cd8` |

O volume já foi entregue formatado e montado pelo provedor. Não foi necessário formatá-lo novamente.

Antes de configurar outra VM, identificar seu próprio volume e UUID. Não reutilizar o UUID acima em outra instância.

A montagem persistente foi configurada no `/etc/fstab` da Storage com a seguinte entrada:

`````fstab
UUID=54adace4-48db-4dd9-a6ad-ed32e60f0cd8 /srv/siter/storage ext4 defaults,noatime,nofail 0 2
`````

A remontagem automática foi validada após reinicialização da VM.

**Importante:** não criar ou utilizar o acervo publicado quando o volume persistente estiver desmontado. A proteção contra escrita no diretório subjacente deverá ser garantida na configuração operacional do módulo.

## 4. Estrutura inicial de armazenamento

Acesse como `sig`.

Criar os diretórios básicos no volume persistente, após confirmar que o volume correto está montado:

`````bash
sudo install -d -o root -g root -m 0700 \
    /srv/siter/storage/recebimento

sudo install -d -o root -g root -m 0750 \
    /srv/siter/storage/publicado
`````

Estrutura resultante:

`````text
/srv/siter/storage/
├── recebimento/
└── publicado/
`````

`recebimento/` armazena transferências ainda não validadas.

`publicado/` mantém produtos completos e aprovados, que poderão ser consultados pelo módulo OGC.

Os arquivos incompletos não devem ser disponibilizados como produtos publicados.

## 5. Servidor NFS

Acesse como `sig`.

Instalar o servidor NFS:

`````bash
sudo apt-get install -y nfs-kernel-server
`````

O serviço foi instalado e está ativo.

A exportação da área `publicado/` será configurada durante a integração com a VM OGC, utilizando a rede privada e permissões somente de leitura.

Nenhum diretório deve ser exportado publicamente.

## 6. Usuário de transferência

Acesse como `sig`.

Criar o usuário específico para receber arquivos enviados pelo módulo Drone:

`````bash
sudo useradd -m -U -s /bin/bash siter-upload
sudo passwd -l siter-upload

sudo chown siter-upload:siter-upload \
    /srv/siter/storage/recebimento

sudo chmod 700 /srv/siter/storage/recebimento
`````

O usuário `siter-upload` não possui sudo nem autenticação por senha.

A autenticação SSH por chave e as restrições da transferência serão configuradas durante a integração com a Drone.

Esse usuário terá escrita somente na área de recebimento. A promoção dos produtos para `publicado/` será responsabilidade do processo autorizado da Storage.

## 7. Pasta de troca do Syncthing

Acesse como `sig`.

Criar uma pasta independente do acervo publicado para transferência de arquivos entre o computador local e a VM:

`````bash
install -d -m 700 /home/sig/swap
`````

Cadastrar `/home/sig/swap` na interface do Syncthing e compartilhar com o dispositivo local autorizado.

O Syncthing não substitui o mecanismo de publicação Drone → Storage nem deve publicar automaticamente arquivos recebidos nessa pasta.

## 8. Encerramento do bootstrap específico

**Estado: CONCLUÍDO.**

Foram preparados:

- Syncthing, interface administrativa por túnel SSH e temas coloridos.
- Firewall com regras básicas de SSH e sincronização.
- Volume persistente de armazenamento.
- Diretórios iniciais de recebimento e publicação.
- Servidor NFS.
- Usuário de transferência `siter-upload`.
- Pasta de troca `/home/sig/swap`.
