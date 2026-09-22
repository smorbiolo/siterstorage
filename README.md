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

# SITER Storage — Bootstrap específico

**Sistema:** Ubuntu Server 26.04 LTS 64 bits  
**Usuário executor:** `sig`  
**Pré-requisito:** bootstrap geral do SITER concluído.

Este procedimento prepara a infraestrutura básica da VM Storage: Syncthing, firewall, temas coloridos, armazenamento persistente, NFS, usuário de transferência e diretórios iniciais.

Não inclui o desenvolvimento do módulo nem as integrações funcionais com Drone e OGC.

## 1. Preparação

Antes de executar, confirme que:

- O bootstrap geral foi concluído.
- O volume `storage-volume` está anexado à VM e formatado em ext4.
- O volume está vazio, exceto pelo diretório `lost+found`.

O script não formata discos, não apaga dados e não altera a configuração do SSH.

## 2. Execução

Acesse a VM Storage pelo PuTTY como `sig` e execute o bloco completo.

`````bash
clear
echo "===============INÍCIO==============="

# VM: Storage
# Módulo: SITER Storage
# Repositório: Não aplicável
# Usuário: sig
# Objetivo: Bootstrap específico preservando dados existentes
# Natureza: ALTERAÇÃO

if bash -euo pipefail <<'SITER'

erro() {
    echo "ERRO: $*" >&2
    exit 1
}

# ========== 1. IDENTIDADE E VOLUME ==========

test "$(id -un)" = sig || erro "Execute como sig"
test "$(hostname)" = storage || erro "VM incorreta: $(hostname)"
sudo -n true || erro "Sudo indisponível"

VOLUME="/dev/disk/by-id/scsi-0DO_Volume_storage-volume"
DESTINO="/srv/siter/storage"
CFG="/home/sig/.local/state/syncthing"

test -b "$VOLUME" || erro "Volume storage-volume não encontrado"

DISCO="$(readlink -f "$VOLUME")"
UUID="$(sudo blkid -s UUID -o value "$DISCO")"
TIPO="$(sudo blkid -s TYPE -o value "$DISCO")"

test "$TIPO" = ext4 || erro "Volume não formatado em ext4"
test -n "$UUID" || erro "UUID não encontrado"

DISCO_SISTEMA="$(lsblk -no PKNAME "$(findmnt -n -o SOURCE /)" | head -n 1)"

test -n "$DISCO_SISTEMA" || erro "Disco do sistema não identificado"

test "$DISCO" != "/dev/$DISCO_SISTEMA" ||
    erro "O volume identificado é o disco do sistema"

test "$(lsblk -dn -o TYPE "$DISCO")" = disk ||
    erro "Dispositivo inesperado"

MONTAGEM="$(findmnt -rn -S "$DISCO" -o TARGET || true)"

case "$MONTAGEM" in
    ""|"/mnt/storage_volume"|"$DESTINO") ;;
    *) erro "Volume montado em local inesperado: $MONTAGEM" ;;
esac

# Impedir que uma montagem esconda arquivos do disco do sistema.

test ! -L "$DESTINO" || erro "Destino é um link simbólico"

if [ -e "$DESTINO" ] && ! mountpoint -q "$DESTINO"; then
    test -d "$DESTINO" || erro "Destino não é um diretório"

    test -z "$(sudo find "$DESTINO" -mindepth 1 -maxdepth 1 -print -quit)" ||
        erro "O destino contém arquivos fora do volume"
fi

if grep -Fq "$UUID" /etc/fstab; then
    grep -Eq "^UUID=${UUID}[[:space:]]+${DESTINO}[[:space:]]+ext4[[:space:]]" /etc/fstab ||
        erro "UUID já configurado em outro ponto de montagem"
else
    ! grep -Fq "$DESTINO" /etc/fstab ||
        erro "Destino já configurado para outro volume"
fi

test ! -e "$CFG/config.xml" ||
    erro "Syncthing já configurado nesta VM"

echo "IDENTIDADE=CONFIRMADA"
echo "VOLUME=$DISCO"
echo "UUID=$UUID"


# ========== 2. VOLUME PERSISTENTE ==========

if ! mountpoint -q "$DESTINO"; then

    if [ ! -e "$DESTINO" ]; then
        sudo install -d -o root -g root -m 000 "$DESTINO"
    else
        sudo chmod 000 "$DESTINO"
    fi

    if [ "$MONTAGEM" = "/mnt/storage_volume" ]; then
        sudo umount /mnt/storage_volume ||
            erro "Volume em uso: não foi possível desmontar"
    fi

    if ! sudo mount -t ext4 -o noatime "$DISCO" "$DESTINO"; then

        if [ "$MONTAGEM" = "/mnt/storage_volume" ]; then
            sudo mount -t ext4 "$DISCO" /mnt/storage_volume ||
                echo "AVISO: montagem anterior não restaurada"
        fi

        erro "Falha ao montar o volume"
    fi
fi

test "$(findmnt -rn -o UUID -M "$DESTINO")" = "$UUID" ||
    erro "Montagem incorreta"

# Configurar persistência somente se ainda não existir.

if ! grep -Eq "^UUID=${UUID}[[:space:]]+${DESTINO}[[:space:]]+ext4[[:space:]]" /etc/fstab; then

    sudo cp -p /etc/fstab /etc/fstab.siter-storage.bak

    printf 'UUID=%s %s ext4 defaults,noatime,nofail 0 2\n' \
        "$UUID" "$DESTINO" |
        sudo tee -a /etc/fstab >/dev/null

    sudo systemctl daemon-reload
fi

echo "VOLUME=CONFIGURADO"
echo "DADOS_EXISTENTES=PRESERVADOS"


# ========== 3. DIRETÓRIOS E USUÁRIO ==========

if ! id siter-upload >/dev/null 2>&1; then
    sudo useradd -m -U -s /bin/bash siter-upload
fi

sudo passwd -l siter-upload

RECEBIMENTO="$DESTINO/recebimento"
PUBLICADO="$DESTINO/publicado"

# Criar somente os diretórios que não existem.
# Nunca alterar recursivamente a propriedade dos dados existentes.

if sudo test -e "$RECEBIMENTO"; then

    sudo test -d "$RECEBIMENTO" ||
        erro "Recebimento não é um diretório"

    ESPERADO="$(id -u siter-upload):$(id -g siter-upload)"

    ATUAL="$(sudo stat -c '%u:%g' "$RECEBIMENTO")"

    test "$ATUAL" = "$ESPERADO" ||
        erro "UID/GID do recebimento incompatível. Dados preservados. Esperado=$ESPERADO Atual=$ATUAL"

    test "$(sudo stat -c '%a' "$RECEBIMENTO")" = 700 ||
        erro "Permissões preexistentes do recebimento incompatíveis"

else

    sudo install -d \
        -o siter-upload -g siter-upload -m 0700 \
        "$RECEBIMENTO"
fi

if sudo test -e "$PUBLICADO"; then

    sudo test -d "$PUBLICADO" ||
        erro "Publicado não é um diretório"

    test "$(sudo stat -c '%u:%g' "$PUBLICADO")" = "0:0" ||
        erro "Propriedade preexistente do publicado incompatível"

    test "$(sudo stat -c '%a' "$PUBLICADO")" = 750 ||
        erro "Permissões preexistentes do publicado incompatíveis"

else

    sudo install -d \
        -o root -g root -m 0750 \
        "$PUBLICADO"
fi

install -d -m 0700 /home/sig/swap

echo "DIRETORIOS=CONFIGURADOS"
echo "ARQUIVOS_PREEXISTENTES=INALTERADOS"


# ========== 4. INSTALAÇÃO DE PACOTES ==========

sudo apt-get update

sudo DEBIAN_FRONTEND=noninteractive apt-get install -y \
    syncthing \
    nfs-kernel-server

systemctl is-active --quiet nfs-kernel-server ||
    erro "Servidor NFS não iniciou"

echo "PACOTES=INSTALADOS"


# ========== 5. CONFIGURAÇÃO DO SYNCTHING ==========

syncthing generate \
    --home="$CFG" \
    --no-default-folder

python3 - "$CFG/config.xml" <<'PY'
import sys
import xml.etree.ElementTree as ET

path = sys.argv[1]
tree = ET.parse(path)
root = tree.getroot()

root.find("gui/address").text = "127.0.0.1:858"

options = root.find("options")

for address in options.findall("listenAddress"):
    options.remove(address)

ET.SubElement(
    options, "listenAddress"
).text = "tcp://0.0.0.0:22000"

for name in (
    "globalAnnounceEnabled",
    "localAnnounceEnabled",
    "relaysEnabled",
    "natEnabled",
    "startBrowser",
):
    element = options.find(name)

    if element is None:
        element = ET.SubElement(options, name)

    element.text = "false"

tree.write(
    path,
    encoding="utf-8",
    xml_declaration=True
)
PY

sudo install -d \
    /etc/systemd/system/syncthing@sig.service.d

printf '%s\n' \
    '[Service]' \
    'AmbientCapabilities=CAP_NET_BIND_SERVICE' \
    'CapabilityBoundingSet=CAP_NET_BIND_SERVICE' |
    sudo tee \
    /etc/systemd/system/syncthing@sig.service.d/10-siter-gui.conf \
    >/dev/null

sudo systemctl daemon-reload

echo "SYNCTHING=CONFIGURADO"


# ========== 6. TEMAS COLORIDOS ==========

VERSAO="$(syncthing --version | awk '{print $2}')"

BASE="$(mktemp)"
STAGE="$(mktemp -d "$CFG/.themes.XXXXXX")"

trap 'rm -f "$BASE"; rm -rf "$STAGE"' EXIT

curl -fsSL \
    "https://raw.githubusercontent.com/syncthing/syncthing/${VERSAO}/gui/dark/assets/css/theme.css" \
    -o "$BASE"

test -s "$BASE" || erro "CSS original não encontrado"

python3 - "$BASE" "$STAGE" "$CFG/gui" <<'PY'
import re
import sys
import shutil
from pathlib import Path

base = Path(sys.argv[1]).read_text(encoding="utf-8")
stage = Path(sys.argv[2])
gui = Path(sys.argv[3])

temas = [
    ("Azul",     "#003", "#004"),
    ("Vermelho", "#300", "#400"),
    ("Verde",    "#030", "#040"),
    ("Amarelo",  "#220", "#330"),
    ("Ciano",    "#022", "#033"),
    ("Roxo",     "#202", "#303"),
    ("Laranja",  "#310", "#420"),
]

body = re.compile(
    r'(body\s*\{[^}]*?background-color\s*:\s*)'
    r'#[0-9a-fA-F]{3,6}(\s*!important\s*;)',
    re.S
)

navbar = re.compile(
    r'(\.navbar\s*\{[^}]*?background-color\s*:\s*)'
    r'#[0-9a-fA-F]{3,6}(\s*!important\s*;)',
    re.S
)

for nome, _, _ in temas:
    if (gui / nome).exists():
        raise SystemExit(f"Tema já existente: {nome}")

for nome, cor_body, cor_navbar in temas:

    css, n1 = body.subn(
        lambda m: m[1] + cor_body + m[2],
        base,
        count=1
    )

    css, n2 = navbar.subn(
        lambda m: m[1] + cor_navbar + m[2],
        css,
        count=1
    )

    if n1 != 1 or n2 != 1:
        raise SystemExit(f"CSS incompatível: {nome}")

    destino = stage / nome / "assets/css/theme.css"

    destino.parent.mkdir(parents=True, exist_ok=True)
    destino.write_text(css, encoding="utf-8")

gui.mkdir(parents=True, exist_ok=True)

for nome, _, _ in temas:
    shutil.move(str(stage / nome), str(gui / nome))
    print(f"TEMA_INSTALADO={nome}")
PY

echo "TEMAS=INSTALADOS"


# ========== 7. FIREWALL ==========

# Preservar SSH e permitir sincronização direta com o PC.
# A GUI permanece restrita ao localhost, sem porta pública.

sudo ufw allow 22000/tcp

sudo ufw status | grep -q '^Status: active' ||
    erro "Firewall inativo"

echo "FIREWALL=CONFIGURADO"


# ========== 8. INICIAR E VERIFICAR ==========

sudo systemctl enable --now syncthing@sig.service

for i in $(seq 1 15); do

    PORTAS="$(sudo ss -lntH | awk '{print $4}')"

    if grep -Fxq '127.0.0.1:858' <<<"$PORTAS" &&
       grep -Fxq '0.0.0.0:22000' <<<"$PORTAS"; then
        break
    fi

    sleep 1
done

systemctl is-active --quiet syncthing@sig.service ||
    erro "Syncthing não iniciou"

sudo ss -lntH | awk '{print $4}' |
    grep -Fxq '127.0.0.1:858' ||
    erro "GUI indisponível"

sudo ss -lntH | awk '{print $4}' |
    grep -Fxq '0.0.0.0:22000' ||
    erro "Sincronização indisponível"

test "$(findmnt -rn -o UUID -M "$DESTINO")" = "$UUID" ||
    erro "Volume não montado"

for tema in Azul Vermelho Verde Amarelo Ciano Roxo Laranja; do

    test -s "$CFG/gui/$tema/assets/css/theme.css" ||
        erro "Tema ausente: $tema"

done

echo "========== ESTADO FINAL =========="

findmnt "$DESTINO"
df -hT "$DESTINO"

systemctl is-active syncthing@sig.service
systemctl is-active nfs-kernel-server

sudo ufw status

echo "BOOTSTRAP_ESPECIFICO=CONFIGURADO"
echo "DADOS_PREEXISTENTES=PRESERVADOS"

SITER
then
    echo "RESULTADO=SUCESSO"
else
    echo "RESULTADO=ERRO"
fi

echo "===============FIM=================="
`````

## 3. Acesso ao Syncthing

Configure o túnel SSH no PuTTY:

| Campo | Valor |
|---|---|
| Source port | `11858` |
| Destination | `127.0.0.1:858` |
| Tipo | Local |

Abra `http://127.0.0.1:11858` no navegador.

Cadastre a pasta `/home/sig/swap` e autorize o computador local como dispositivo de sincronização.

Os sete temas estarão disponíveis para seleção manual.

## 4. Relatório de Avaliação

Após executar o bootstrap específico, reinicie a VM e faça a verificação consolidada dos dois bootstraps.

O relatório deverá confirmar o estado do sistema, firewall, SSH, Syncthing, temas, volume persistente, diretórios, permissões e servidor NFS.

`````
clear
echo "===============INÍCIO==============="

# VM: Storage
# Módulo: Bootstrap geral + específico SITER Storage
# Repositório: Não aplicável
# Usuário: sig
# Objetivo: Relatório final de homologação dos bootstraps
# Natureza: SOMENTE LEITURA

bash <<'SITER'

FALHAS=0
STORAGE="/srv/siter/storage"
CFG="/home/sig/.local/state/syncthing"
VOLUME="/dev/disk/by-id/scsi-0DO_Volume_storage-volume"

verificar() {
    local descricao="$1"
    shift

    if "$@" >/dev/null 2>&1; then
        echo "[OK] $descricao"
    else
        echo "[ERRO] $descricao"
        FALHAS=$((FALHAS + 1))
    fi
}

# Identidade antes de qualquer outra operação
if [ "$(id -un)" != sig ] ||
   [ "$(hostname)" != storage ] ||
   [ ! -b "$VOLUME" ]; then
    echo "RESULTADO=ERRO | VM ou usuário incorreto"
    echo "ESPERADO: sig@storage com volume storage-volume"
    echo "DETECTADO: $(id -un)@$(hostname)"
    exit 1
fi

echo "========== 1. BOOTSTRAP GERAL =========="

. /etc/os-release

verificar "Ubuntu Server 26.04" test "$VERSION_ID" = "26.04"
verificar "Arquitetura 64 bits" test "$(uname -m)" = "x86_64"
verificar "Usuário sig com sudo" sudo -n true
verificar "SSH ativo" systemctl is-active --quiet ssh

verificar "Login SSH de root desabilitado" \
    bash -c "sudo sshd -T | grep -qx 'permitrootlogin no'"

for pacote in ca-certificates curl git nano ufw; do
    verificar "Pacote $pacote" \
        bash -c "dpkg-query -W -f='\${Status}' '$pacote' | grep -qx 'install ok installed'"
done

FW="$(sudo ufw status verbose)"

verificar "Firewall ativo" \
    grep -q '^Status: active' <<<"$FW"

verificar "Entradas bloqueadas por padrão" \
    grep -q 'Default: deny (incoming)' <<<"$FW"

verificar "SSH 22 liberado" \
    grep -Eq '^22/tcp[[:space:]]+ALLOW[[:space:]]+Anywhere' <<<"$FW"

echo "========== 2. ARMAZENAMENTO =========="

DEV="$(readlink -f "$VOLUME")"
UUID="$(sudo blkid -s UUID -o value "$DEV")"

echo "DISPOSITIVO=$DEV"
echo "UUID=$UUID"

verificar "Volume ext4" \
    test "$(sudo blkid -s TYPE -o value "$DEV")" = ext4

verificar "Volume montado no local correto" \
    test "$(findmnt -rn -o UUID -M "$STORAGE")" = "$UUID"

verificar "Montagem persistente no fstab" \
    awk -v uuid="$UUID" -v destino="$STORAGE" \
    '$1=="UUID="uuid && $2==destino && $3=="ext4" {ok=1}
     END {exit !ok}' /etc/fstab

verificar "Diretório recebimento" \
    test -d "$STORAGE/recebimento"

verificar "Diretório publicado" \
    test -d "$STORAGE/publicado"

verificar "Permissões do recebimento" \
    test "$(stat -c '%a %U:%G' "$STORAGE/recebimento")" = \
    "700 siter-upload:siter-upload"

verificar "Permissões do publicado" \
    test "$(stat -c '%a %U:%G' "$STORAGE/publicado")" = \
    "750 root:root"

df -hT "$STORAGE"

echo "========== 3. USUÁRIO DE TRANSFERÊNCIA =========="

verificar "Usuário siter-upload existente" id siter-upload

verificar "Senha do usuário bloqueada" \
    bash -c "sudo passwd -S siter-upload | grep -Eq '^siter-upload[[:space:]]+L'"

verificar "Usuário sem sudo" \
    bash -c "! id -nG siter-upload | grep -qw sudo"

echo "========== 4. SYNCTHING =========="

verificar "Serviço ativo" \
    systemctl is-active --quiet syncthing@sig.service

verificar "Inicialização automática" \
    systemctl is-enabled --quiet syncthing@sig.service

PORTAS="$(sudo ss -lntH | awk '{print $4}')"

verificar "GUI restrita ao localhost na porta 858" \
    grep -Fxq '127.0.0.1:858' <<<"$PORTAS"

verificar "Sincronização TCP 22000 disponível" \
    grep -Fxq '0.0.0.0:22000' <<<"$PORTAS"

verificar "Configuração de rede e discovery" \
    python3 - "$CFG/config.xml" <<'PY'
import sys
import xml.etree.ElementTree as ET

root = ET.parse(sys.argv[1]).getroot()
gui = root.find("gui")
opt = root.find("options")

assert gui.findtext("address") == "127.0.0.1:858"

addresses = [
    e.text for e in opt.findall("listenAddress")
]
assert addresses == ["tcp://0.0.0.0:22000"]

for name in (
    "globalAnnounceEnabled",
    "localAnnounceEnabled",
    "relaysEnabled",
    "natEnabled",
):
    assert opt.findtext(name) == "false", name
PY

verificar "Pasta de troca PC/VM" \
    test "$(stat -c '%a %U:%G' /home/sig/swap)" = "700 sig:sig"

echo "========== 5. TEMAS COLORIDOS =========="

for tema in Azul Vermelho Verde Amarelo Ciano Roxo Laranja; do
    verificar "Tema $tema" \
        test -s "$CFG/gui/$tema/assets/css/theme.css"
done

echo "========== 6. SERVIDOR NFS =========="

verificar "NFS instalado" \
    bash -c "dpkg-query -W -f='\${Status}' nfs-kernel-server | grep -qx 'install ok installed'"

verificar "Servidor NFS ativo" \
    systemctl is-active --quiet nfs-kernel-server

verificar "Nenhum diretório NFS exportado" \
    test -z "$(sudo exportfs -v)"

echo "========== 7. FIREWALL ESPECÍFICO =========="

verificar "Syncthing TCP 22000 liberado" \
    grep -Eq '^22000/tcp[[:space:]]+ALLOW[[:space:]]+Anywhere' <<<"$FW"

verificar "GUI e NFS sem liberação pública no UFW" \
    bash -c "! sudo ufw status | grep -Eq '^(858|8384|2049)(/tcp|/udp)?[[:space:]]+ALLOW'"

sudo ufw status

echo "========== RELATÓRIO FINAL =========="

echo "VM=$(hostname)"
echo "SISTEMA=$PRETTY_NAME"
echo "USUARIO=$(id -un)"
echo "FALHAS=$FALHAS"

if [ "$FALHAS" -eq 0 ]; then
    echo "BOOTSTRAP_GERAL=APROVADO"
    echo "BOOTSTRAP_STORAGE=APROVADO"
    echo "RESULTADO=SUCESSO"
else
    echo "BOOTSTRAP=VERIFICACAO_INCOMPLETA"
    echo "RESULTADO=ERRO"
fi

SITER

echo "===============FIM=================="
