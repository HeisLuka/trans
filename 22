#!/bin/bash

# --- ЦВЕТА ---
RED='\033[0;31m'
GREEN='\033[0;32m'
CYAN='\033[0;36m'
YELLOW='\033[1;33m'
MAGENTA='\033[0;35m'
WHITE='\033[1;37m'
NC='\033[0m'

# --- ПРОВЕРКА ROOT ---
check_root() {
    if [ "$EUID" -ne 0 ]; then
        echo -e "${RED}[ERROR] Запустите скрипт с правами root!${NC}"
        exit 1
    fi
}

# --- ПОДГОТОВКА СИСТЕМЫ ---
prepare_system() {
    # Создание глобальной команды gokaskad
    if [ "$0" != "/usr/local/bin/gokaskad" ]; then
        cp -f "$0" "/usr/local/bin/gokaskad"
        chmod +x "/usr/local/bin/gokaskad"
    fi

    # Включение IP Forwarding
    if ! grep -q "^net.ipv4.ip_forward=1$" /etc/sysctl.conf; then
        if grep -q "^#net.ipv4.ip_forward=1$" /etc/sysctl.conf; then
            sed -i 's/^#net.ipv4.ip_forward=1$/net.ipv4.ip_forward=1/' /etc/sysctl.conf
        else
            echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
        fi
    fi

    # Включение BBR
    if ! grep -q "^net.core.default_qdisc=fq$" /etc/sysctl.conf; then
        echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
    fi
    if ! grep -q "^net.ipv4.tcp_congestion_control=bbr$" /etc/sysctl.conf; then
        echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
    fi

    sysctl -p > /dev/null

    # Установка зависимостей только если их нет
    export DEBIAN_FRONTEND=noninteractive
    if ! dpkg -s iptables-persistent >/dev/null 2>&1; then
        apt-get update -y > /dev/null
        apt-get install -y iptables-persistent netfilter-persistent > /dev/null
    fi
}

# --- ИНСТРУКЦИЯ ---
show_instructions() {
    clear
    echo -e "${MAGENTA}╔══════════════════════════════════════════════════════════════╗${NC}"
    echo -e "${MAGENTA}║                 ИНСТРУКЦИЯ ПО НАСТРОЙКЕ КАСКАДА             ║${NC}"
    echo -e "${MAGENTA}╚══════════════════════════════════════════════════════════════╝${NC}"
    echo ""
    echo -e "${CYAN}Схема работы:${NC}"
    echo -e "Клиент -> Этот сервер -> Целевой сервер"
    echo ""
    echo -e "${CYAN}Что нужно:${NC}"
    echo -e " - IP второго сервера"
    echo -e " - Порт на втором сервере"
    echo -e " - Понимание, какой протокол используешь: TCP или UDP"
    echo ""
    echo -e "${CYAN}Как использовать:${NC}"
    echo -e "1. Выберите тип правила:"
    echo -e "   - ${GREEN}UDP${NC} для WireGuard / AmneziaWG"
    echo -e "   - ${GREEN}TCP${NC} для VLESS / XRay / MTProto / SSH / RDP"
    echo -e "2. Введите IP второго сервера."
    echo -e "3. Введите порт."
    echo -e "4. Подключайте клиент уже к ${GREEN}этому${NC} серверу."
    echo ""
    echo -e "${CYAN}Примеры:${NC}"
    echo -e " - VLESS: вход 443 -> второй сервер:443"
    echo -e " - WireGuard: вход 51820/udp -> второй сервер:51820/udp"
    echo -e " - Кастом: вход 8443 -> второй сервер:443"
    echo ""
    echo -e "${YELLOW}Важно:${NC}"
    echo -e "Этот скрипт делает именно портовое перенаправление через iptables,"
    echo -e "а не полноценный site-to-site туннель."
    echo ""
    read -p "Нажмите Enter, чтобы вернуться в меню..."
}

# --- СТАНДАРТНАЯ НАСТРОЙКА (ПОРТ ВХОДА = ПОРТ ВЫХОДА) ---
configure_rule() {
    local PROTO=$1
    local NAME=$2

    echo -e "\n${CYAN}--- Настройка $NAME ($PROTO) ---${NC}"

    while true; do
        echo -e "Введите IP адрес назначения:"
        read -p "> " TARGET_IP
        if [[ -n "$TARGET_IP" ]]; then
            break
        fi
        echo -e "${RED}Ошибка: IP не может быть пустым!${NC}"
    done

    while true; do
        echo -e "Введите Порт (одинаковый для входа и выхода):"
        read -p "> " PORT
        if [[ "$PORT" =~ ^[0-9]+$ ]] && [ "$PORT" -ge 1 ] && [ "$PORT" -le 65535 ]; then
            break
        fi
        echo -e "${RED}Ошибка: порт должен быть числом от 1 до 65535!${NC}"
    done

    apply_iptables_rules "$PROTO" "$PORT" "$PORT" "$TARGET_IP" "$NAME"
}

# --- КАСТОМНАЯ НАСТРОЙКА (РАЗНЫЕ ПОРТЫ) ---
configure_custom_rule() {
    echo -e "\n${CYAN}--- Кастомное правило ---${NC}"
    echo -e "${WHITE}Можно указать разные входящий и исходящий порты.${NC}\n"

    while true; do
        echo -e "Выберите протокол (${YELLOW}tcp${NC} или ${YELLOW}udp${NC}):"
        read -p "> " PROTO
        if [[ "$PROTO" == "tcp" || "$PROTO" == "udp" ]]; then
            break
        fi
        echo -e "${RED}Ошибка: введите tcp или udp!${NC}"
    done

    while true; do
        echo -e "Введите IP адрес назначения:"
        read -p "> " TARGET_IP
        if [[ -n "$TARGET_IP" ]]; then
            break
        fi
        echo -e "${RED}Ошибка: IP не может быть пустым!${NC}"
    done

    while true; do
        echo -e "Введите ВХОДЯЩИЙ порт (на этом сервере):"
        read -p "> " IN_PORT
        if [[ "$IN_PORT" =~ ^[0-9]+$ ]] && [ "$IN_PORT" -ge 1 ] && [ "$IN_PORT" -le 65535 ]; then
            break
        fi
        echo -e "${RED}Ошибка: порт должен быть числом от 1 до 65535!${NC}"
    done

    while true; do
        echo -e "Введите ИСХОДЯЩИЙ порт (на целевом сервере):"
        read -p "> " OUT_PORT
        if [[ "$OUT_PORT" =~ ^[0-9]+$ ]] && [ "$OUT_PORT" -ge 1 ] && [ "$OUT_PORT" -le 65535 ]; then
            break
        fi
        echo -e "${RED}Ошибка: порт должен быть числом от 1 до 65535!${NC}"
    done

    apply_iptables_rules "$PROTO" "$IN_PORT" "$OUT_PORT" "$TARGET_IP" "Custom Rule"
}

# --- ПРИМЕНЕНИЕ ПРАВИЛ IPTABLES ---
apply_iptables_rules() {
    local PROTO=$1
    local IN_PORT=$2
    local OUT_PORT=$3
    local TARGET_IP=$4
    local NAME=$5

    local IFACE
    IFACE=$(ip route get 8.8.8.8 | awk '{print $5}')

    if [[ -z "$IFACE" ]]; then
        echo -e "${RED}[ERROR] Не удалось определить сетевой интерфейс!${NC}"
        read -p "Нажмите Enter..."
        return
    fi

    echo -e "${YELLOW}[*] Применение правил...${NC}"

    # Удаление старых таких же правил
    iptables -t nat -D PREROUTING -p "$PROTO" --dport "$IN_PORT" -j DNAT --to-destination "$TARGET_IP:$OUT_PORT" 2>/dev/null
    iptables -D INPUT -p "$PROTO" --dport "$IN_PORT" -j ACCEPT 2>/dev/null
    iptables -D FORWARD -p "$PROTO" -d "$TARGET_IP" --dport "$OUT_PORT" -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT 2>/dev/null
    iptables -D FORWARD -p "$PROTO" -s "$TARGET_IP" --sport "$OUT_PORT" -m state --state ESTABLISHED,RELATED -j ACCEPT 2>/dev/null

    # Новые правила
    iptables -A INPUT -p "$PROTO" --dport "$IN_PORT" -j ACCEPT
    iptables -t nat -A PREROUTING -p "$PROTO" --dport "$IN_PORT" -j DNAT --to-destination "$TARGET_IP:$OUT_PORT"

    if ! iptables -t nat -C POSTROUTING -o "$IFACE" -j MASQUERADE 2>/dev/null; then
        iptables -t nat -A POSTROUTING -o "$IFACE" -j MASQUERADE
    fi

    iptables -A FORWARD -p "$PROTO" -d "$TARGET_IP" --dport "$OUT_PORT" -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
    iptables -A FORWARD -p "$PROTO" -s "$TARGET_IP" --sport "$OUT_PORT" -m state --state ESTABLISHED,RELATED -j ACCEPT

    # Если активен UFW — добавим разрешение
    if command -v ufw >/dev/null 2>&1 && ufw status | grep -q "Status: active"; then
        ufw allow "$IN_PORT/$PROTO" >/dev/null
        sed -i 's/DEFAULT_FORWARD_POLICY="DROP"/DEFAULT_FORWARD_POLICY="ACCEPT"/' /etc/default/ufw
        ufw reload >/dev/null
    fi

    netfilter-persistent save > /dev/null

    echo -e "${GREEN}[SUCCESS] $NAME настроен!${NC}"
    echo -e "${WHITE}$PROTO:${NC} вход ${YELLOW}$IN_PORT${NC} -> выход ${YELLOW}$TARGET_IP:$OUT_PORT${NC}"
    read -p "Нажмите Enter для возврата в меню..."
}

# --- СПИСОК ПРАВИЛ ---
list_active_rules() {
    echo -e "\n${CYAN}--- Активные переадресации ---${NC}"
    echo -e "${MAGENTA}ПОРТ(ВХОД)\tПРОТОКОЛ\tЦЕЛЬ(IP:ВЫХОД)${NC}"

    iptables -t nat -S PREROUTING | grep "DNAT" | while read -r line; do
        local l_port l_proto l_dest
        l_port=$(echo "$line" | grep -oP '(?<=--dport )\d+')
        l_proto=$(echo "$line" | grep -oP '(?<=-p )\w+')
        l_dest=$(echo "$line" | grep -oP '(?<=--to-destination )[\d\.:]+')
        if [[ -n "$l_port" ]]; then
            echo -e "$l_port\t\t$l_proto\t\t$l_dest"
        fi
    done

    echo ""
    read -p "Нажмите Enter..."
}

# --- УДАЛЕНИЕ ОДНОГО ПРАВИЛА ---
delete_single_rule() {
    echo -e "\n${CYAN}--- Удаление правила ---${NC}"

    declare -a RULES_LIST
    local i=1

    while read -r line; do
        local l_port l_proto l_dest
        l_port=$(echo "$line" | grep -oP '(?<=--dport )\d+')
        l_proto=$(echo "$line" | grep -oP '(?<=-p )\w+')
        l_dest=$(echo "$line" | grep -oP '(?<=--to-destination )[\d\.:]+')
        if [[ -n "$l_port" ]]; then
            RULES_LIST[$i]="$l_port:$l_proto:$l_dest"
            echo -e "${YELLOW}[$i]${NC} Вход: $l_port ($l_proto) -> Выход: $l_dest"
            ((i++))
        fi
    done < <(iptables -t nat -S PREROUTING | grep "DNAT")

    if [ ${#RULES_LIST[@]} -eq 0 ]; then
        echo -e "${RED}Нет активных правил.${NC}"
        read -p "Нажмите Enter..."
        return
    fi

    echo ""
    read -p "Номер правила для удаления (0 = отмена): " rule_num

    if [[ "$rule_num" == "0" || -z "${RULES_LIST[$rule_num]}" ]]; then
        return
    fi

    local d_port d_proto d_dest
    IFS=':' read -r d_port d_proto d_dest <<< "${RULES_LIST[$rule_num]}"

    local target_ip="${d_dest%:*}"
    local target_port="${d_dest#*:}"

    iptables -t nat -D PREROUTING -p "$d_proto" --dport "$d_port" -j DNAT --to-destination "$d_dest" 2>/dev/null
    iptables -D INPUT -p "$d_proto" --dport "$d_port" -j ACCEPT 2>/dev/null
    iptables -D FORWARD -p "$d_proto" -d "$target_ip" --dport "$target_port" -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT 2>/dev/null
    iptables -D FORWARD -p "$d_proto" -s "$target_ip" --sport "$target_port" -m state --state ESTABLISHED,RELATED -j ACCEPT 2>/dev/null

    netfilter-persistent save > /dev/null
    echo -e "${GREEN}[OK] Правило удалено.${NC}"
    read -p "Нажмите Enter..."
}

# --- ПОЛНАЯ ОЧИСТКА ---
flush_rules() {
    echo -e "\n${RED}!!! ВНИМАНИЕ !!!${NC}"
    echo "Будут сброшены все правила iptables."
    read -p "Вы уверены? (y/n): " confirm

    if [[ "$confirm" == "y" ]]; then
        iptables -P INPUT ACCEPT
        iptables -P FORWARD ACCEPT
        iptables -P OUTPUT ACCEPT
        iptables -t nat -F
        iptables -t mangle -F
        iptables -F
        iptables -X
        netfilter-persistent save > /dev/null
        echo -e "${GREEN}[OK] Очищено.${NC}"
    fi

    read -p "Нажмите Enter..."
}

# --- МЕНЮ ---
show_menu() {
    while true; do
        clear
        echo -e "${MAGENTA}======================================================${NC}"
        echo -e "${MAGENTA}                  GOKASKAD — МЕНЮ                     ${NC}"
        echo -e "${MAGENTA}======================================================${NC}"
        echo ""
        echo -e "1) Настроить ${CYAN}AmneziaWG / WireGuard${NC} (UDP)"
        echo -e "2) Настроить ${CYAN}VLESS / XRay${NC} (TCP)"
        echo -e "3) Настроить ${CYAN}MTProto / TProxy${NC} (TCP)"
        echo -e "4) Создать ${YELLOW}кастомное правило${NC} (разные порты)"
        echo -e "5) Посмотреть активные правила"
        echo -e "6) ${RED}Удалить одно правило${NC}"
        echo -e "7) ${RED}Сбросить все настройки${NC}"
        echo -e "8) Показать инструкцию"
        echo -e "0) Выход"
        echo -e "------------------------------------------------------"
        read -p "Ваш выбор: " choice

        case $choice in
            1) configure_rule "udp" "AmneziaWG / WireGuard" ;;
            2) configure_rule "tcp" "VLESS / XRay" ;;
            3) configure_rule "tcp" "MTProto / TProxy" ;;
            4) configure_custom_rule ;;
            5) list_active_rules ;;
            6) delete_single_rule ;;
            7) flush_rules ;;
            8) show_instructions ;;
            0) exit 0 ;;
            *) ;;
        esac
    done
}

# --- ЗАПУСК ---
check_root
prepare_system
show_menu
