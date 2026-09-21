# SITER — Bootstrap Geral Ubuntu Server

Bootstrap comum para VMs do SITER, utilizando Ubuntu Server 26.04 LTS 64 bits.

**Objetivo:** preparar uma VM básica, segura e reutilizável como imagem-base para snapshots.

## Bloco 1 — Criar SIG e ativar firewall

Acesse a VM como `root`.

Este bloco cria o usuário `sig`, configura SSH e sudo e ativa o firewall, permitindo somente conexões de entrada pela porta SSH 22, sem restrição por IP.

O login de root permanece habilitado até o bloco 3.

`````bash
clear
echo "===============INÍCIO===2026-09-21,16:09:54"

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

echo "===============FIM===2026-09-21,16:09:54"
`````

## Bloco 2 — Testar acesso com SIG

**Abra uma nova sessão PuTTY como `sig`.** Mantenha a sessão de root aberta.

Execute o bloco para confirmar o acesso SSH e o funcionamento do sudo.

`````bash
clear
echo "===============INÍCIO===2026-09-21,16:09:54"

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

echo "===============FIM===2026-09-21,16:09:54"
`````

## Bloco 3 — Desabilitar acesso SSH de root

Após confirmar o sucesso do bloco 2, volte à sessão PuTTY de `root`.

Mantenha a sessão de `sig` aberta.

`````bash
clear
echo "===============INÍCIO===2026-09-21,16:09:54"

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

echo "===============FIM===2026-09-21,16:09:54"
`````

## Bloco 4 — Atualizar sistema e reiniciar

Execute como `sig`.

Atualiza o Ubuntu, instala os pacotes básicos comuns ao SITER e agenda a reinicialização da VM.

O firewall permanece configurado para permitir somente SSH na entrada.

**A conexão PuTTY será interrompida durante o reboot.**

`````bash
clear
echo "===============INÍCIO===2026-09-21,16:09:54"

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

    sudo shutdown -r +1

); then
    echo "RESULTADO=SUCESSO"
else
    echo "RESULTADO=ERRO"
fi

echo "===============FIM===2026-09-21,16:09:54"
`````

## Bloco 5 — Recuperação e verificação final

Após o reboot, abra uma nova sessão PuTTY como `sig`.

Execute o bloco para verificar o estado geral da VM.

`````bash
clear
echo "===============INÍCIO===2026-09-21,16:09:54"

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

echo "===============FIM===2026-09-21,16:09:54"
`````

## Snapshot e reutilização

Após a conclusão dos cinco blocos, a VM está preparada com o bootstrap comum do SITER.

O snapshot pode ser criado antes de instalar programas ou aplicar configurações específicas de qualquer módulo.

Ao criar novas VMs a partir do snapshot, garantir que cada instância tenha identidade própria, especialmente hostname, machine-id e chaves de host SSH. Verificar se o provedor realiza essa individualização automaticamente.

O bootstrap específico de cada VM será realizado separadamente, conforme a necessidade de seu módulo.

**Fim do bootstrap geral.**
