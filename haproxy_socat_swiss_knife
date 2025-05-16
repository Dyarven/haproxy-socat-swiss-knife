#!/bin/bash

HAPROXY_SOCK="/var/run/haproxy/admin.sock"

# Colours for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No colour

check_reqs() {
  if ! command -v socat >/dev/null 2>&1; then
    echo -e "${RED}Socat package is not installed.${NC}"
    exit 1
  fi

  if [ ! -S "$HAPROXY_SOCK" ]; then
    echo -e "${RED}Error: HAProxy socket not found at $HAPROXY_SOCK${NC}"
    exit 1
  fi
}

run_socat() {
  local cmd="$1"
  output=$(echo "$cmd" | socat stdio unix-connect:"$HAPROXY_SOCK" 2>&1)
  if [ $? -ne 0 ]; then
    echo -e "${RED}Socat run command failed: $cmd${NC}"
    echo "$output"
    return 1
  fi
  echo "$output"
}

select_backend() {
  local stat_output
  stat_output=$(run_socat "show stat")
  mapfile -t backend_list < <(echo "$stat_output" | awk -F, 'NR>1 && $2 == "BACKEND" {print $1}')

  [ ${#backend_list[@]} -eq 0 ] && return 1

  echo -e "${GREEN}Available backends:${NC}"
  for i in "${!backend_list[@]}"; do
    local status
    status=$(echo "$stat_output" | grep "^${backend_list[$i]},BACKEND" | cut -d, -f18)
    case $status in
      "UP") color="$GREEN" ;;
      "DOWN") color="$RED" ;;
      "MAINT") color="$YELLOW" ;;
      *) color="$NC" ;;
    esac
    printf "%2d. %-30s ${color}%s${NC}\n" $((i+1)) "${backend_list[$i]}" "[$status]"
  done

  read -rp "Enter backend number: " selection
  if [[ "$selection" =~ ^[0-9]+$ && $selection -le ${#backend_list[@]} ]]; then
    selected_backend="${backend_list[$((selection-1))]}"
    return 0
  fi
  return 1
}

list_frontends() {
  local stat_output
  stat_output=$(run_socat "show stat")

  echo -e "${BLUE}=== Frontends ===${NC}"
  # Show the list numbered to ease selection in next function. Need to define it inside AWK (since it's not aware of bash local variables)
  echo "$stat_output" | awk -F, 'BEGIN {count=1} $2=="FRONTEND" {printf "%-3d %-20s %-10s %-15s\n", count++, $1, $2, $18}'
}

enable_disable_frontend() {
  list_frontends
  local frontends
  # Store frontends in array
  mapfile -t frontends < <(run_socat "show stat" | awk -F, '$2=="FRONTEND" {print $1}')
  read -rp "Enter a number (1-${#frontends[@]}): " num
  [[ ! "$num" =~ ^[0-9]+$ || $num -lt 1 || $num -gt ${#frontends[@]} ]] && return 1

  read -rp "Enter 1 to enable or 2 to disable $frontend" action
  case $action in
    1) cmd="enable frontend ${frontends[$((num-1))]}" ;;
    2) cmd="disable frontend ${frontends[$((num-1))]}" ;;
    *) return 1 ;;
  esac
  run_socat "$cmd"
  new_state=$(run_socat "show stat" | awk -F, -v fe="$frontend" '$1==fe && $2=="FRONTEND" {print $18}')
  echo -e "New state: ${GREEN}$new_state${NC}"
}

show_info() {
  echo -e "${BLUE}=== HAProxy Runtime Info ===${NC}"
  run_socat "show info" | awk -F: '{printf "%-30s %s\n", $1, $2}'
}

show_errors() {
  echo -e "${BLUE}=== Latest Errors ===${NC}"
  local errors
  errors=$(run_socat "show errors")
  [ -z "$errors" ] && echo -e "${YELLOW}No recent errors${NC}" && return
  echo "$errors" | while read -r line; do
    [[ "$line" == *"[ALERT]"* ]] && echo -e "${RED}$line${NC}" && continue
    [[ "$line" == *"[WARNING]"* ]] && echo -e "${YELLOW}$line${NC}" && continue
    echo "$line"
  done
}

show_sessions() {
  echo -e "${BLUE}=== Active Sessions ===${NC}"
  local sessions
  sessions=$(run_socat "show sess")
  [ -z "$sessions" ] && echo -e "${YELLOW}No active sessions${NC}" && return
  echo "$sessions" | head -n 20 | awk '{
    if (length($0) > 120) print substr($0, 1, 120) "..."
    else print $0
  }' | column -t -s,
}

show_threads() {
  echo -e "${BLUE}=== Threads ===${NC}"
  run_socat "show threads" | column -s, -t | less -S -X +F
}

show_peers() {
  echo -e "${BLUE}=== Peers Status ===${NC}"
  local peers
  peers=$(run_socat "show peers")
  if [ -z "$peers" ]; then
    echo -e "${YELLOW}No peers configured${NC}"
    return
  fi
  echo "$peers"
}

show_stick_table() {
  [ -z "$1" ] && select_backend || selected_backend="$1"
  [ -z "$selected_backend" ] && return 1

  local table
  table=$(run_socat "show table $selected_backend")
  [[ "$table" == *"Unknown table."* ]] && echo -e "${RED}No stick table found${NC}" && return 1

  echo "$table" | awk 'BEGIN {OFS=" | "}
  NR==1 {print; next}
  /^0x/ {
    gsub(/=/,": ",$NF)
    printf "%-15s | %-15s | %-10s | %s\n", $1, $2, $3, $NF
  }'
}

clear_stick_table() {
  select_backend || return
  echo -e "${RED}WARNING: Clearing all entries in $selected_backend stick table${NC}"
  read -rp "Confirm (y/n)? " confirm
  [[ "$confirm" =~ [yY] ]] || return

  run_socat "clear table $selected_backend"
  echo -e "${GREEN}Stick table cleared${NC}"
}

change_backend_server_state() {
  select_backend || return
  echo -e "${GREEN}Selected backend: $selected_backend${NC}"

  # List servers for the backend
  local stat_output
  stat_output=$(run_socat "show stat")
  mapfile -t server_list < <(echo "$stat_output" | awk -F, -v bk="$selected_backend" 'NR>1 && $1==bk && $2 != "BACKEND" && $2 != "FRONTEND" {print $2","$18}')

  if [ ${#server_list[@]} -eq 0 ]; then
    echo -e "${RED}No servers found for backend $selected_backend${NC}"
    return 1
  fi

  echo -e "${GREEN}Available servers for $selected_backend:${NC}"
  for i in "${!server_list[@]}"; do
    IFS=',' read -r server status <<< "${server_list[$i]}"
    case $status in
      "UP") color="$GREEN" ;;
      "DOWN") color="$RED" ;;
      "MAINT") color="$YELLOW" ;;
      *) color="$NC" ;;
    esac
    printf "%2d. %-20s ${color}[%s]${NC}\n" $((i+1)) "$server" "$status"
  done

  read -rp "Enter server number: " selection
  if ! [[ "$selection" =~ ^[0-9]+$ && $selection -le ${#server_list[@]} ]]; then
    echo -e "${RED}Invalid selection.${NC}"
    return 1
  fi
  IFS=',' read -r selected_server current_status <<< "${server_list[$((selection-1))]}"

  echo -e "${GREEN}Selected server: $selected_backend/$selected_server (current status: [$current_status])${NC}"

  echo "Select desired state:"
  echo "1. ready"
  echo "2. maint"
  echo "3. drain"
  echo "4. up"
  echo "5. down"
  read -rp "Choose action: " action
  case $action in
    1) cmd="set server $selected_backend/$selected_server state ready" ;;
    2) cmd="set server $selected_backend/$selected_server state maint" ;;
    3) cmd="set server $selected_backend/$selected_server state drain" ;;
    4) cmd="set server $selected_backend/$selected_server health up" ;;
    5) cmd="set server $selected_backend/$selected_server health down" ;;
    *) echo -e "${RED}Invalid action.${NC}"; return 1 ;;
  esac

  run_socat "$cmd"
  local new_status
  new_status=$(run_socat "show stat" | grep "^$selected_backend,$selected_server," | cut -d, -f18)
  echo -e "${GREEN}New status: $new_status${NC}"
}

check_cookies() {
  local stats
  stats=$(run_socat "show stat")
  echo -e "${BLUE}=== Cookie-Based Backends ===${NC}"
  echo "$stats" | awk -F, '
    NR==1 {for(i=1;i<=NF;i++) cols[$i]=i}
    NR>1 && $cols["mode"]=="http" && $cols["cookie"]!="" {
      printf "%-25s %-20s\n", $1, $cols["cookie"]
    }'
}

watch_stats() {
  read -rp "Enter refresh interval (seconds): " interval
  interval=${interval:-2}
  echo -e "${GREEN}Starting monitoring (Ctrl+C to stop)...${NC}"

  # Not sure if current formatting is all that useful
  watch -n $interval -c "
    echo 'show stat' | socat unix-connect:'$HAPROXY_SOCK' stdio | \
    awk -F, 'BEGIN {ORS=\" \"}
      NR==1 {
        for(i=1;i<=NF;i++) h[\$i]=i
      }
      NR>1 && /$1/ {
        printf \"\033[33m%-20s\033[0m | Status: \", \$1
        if(\$h[\"status\"] ~ /OPEN|UP/) printf \"\033[32m%8s\033[0m | \", \$h[\"status\"]
        else printf \"\033[31m%8s\033[0m | \", \$h[\"status\"]
        printf \"Sessions: \033[34m%4s\033[0m | Bytes: \033[35m%6s\033[0m\n\", \
               \$h[\"scur\"], \$h[\"bytes_out\"]
      }'
  "
}

trigger_health_check() {
  select_backend || return
  local server_list
  server_list=$(run_socat "show stat" | awk -F, -v bk="$selected_backend" 'NR>1 && $1==bk && $2 != "BACKEND" && $2 != "FRONTEND" {print $2}')
  [ -z "$server_list" ] && echo -e "${RED}No servers found for backend $selected_backend${NC}" && return 1

  PS3="Select a server: "
  select server in $server_list; do
    [ -z "$server" ] && return 1
    # Show/record the current state for the selected server.
    current_state=$(run_socat "show stat" | awk -F, -v bk="$selected_backend" -v srv="$server" '$1==bk && $2==srv {print $18}')
    echo -e "${GREEN}Selected server: $selected_backend/$server (current status: [$current_state])${NC}"
    echo "Do you want to (e)nable or (d)isable health check? (e/d)"
    read -r action

    if [[ "$action" =~ ^[eE] ]]; then
      echo -e "${GREEN}Enabling health check for $selected_backend/$server...${NC}"
      run_socat "enable health $selected_backend/$server"
      echo -e "await for results... (30 retries)"
      # Once enabled, loops through healthcheck results and stops when the backend state changes (or it reaches max attempts)
      for i in {1..30}; do
        sleep 1
        new_state=$(run_socat "show stat" | awk -F, -v bk="$selected_backend" -v srv="$server" '$1==bk && $2==srv {print $18}')
        echo -e "Attempt $i: Health-check result is ${GREEN}$new_state${NC}"
        if [ "$new_state" != "$current_state" ]; then
          break
        fi
      done
      echo -e "Current health-check status for ${GREEN}$selected_backend/$server${NC}: ${GREEN}$new_status${NC}"

    elif [[ "$action" =~ ^[dD] ]]; then
      echo -e "${GREEN}Disabling health check for $selected_backend/$server...${NC}"
      run_socat "disable health $selected_backend/$server"
    else
      echo -e "${RED}Invalid action. Please choose 'e' or 'd'.${NC}"
      return 1
    fi
    break
  done
}


change_socket() {
  read -rp "New socket path: " new_sock
  [ -S "$new_sock" ] && HAPROXY_SOCK="$new_sock" || \
    echo -e "${RED}Invalid socket. Keeping default one: $HAPROXY_SOCK${NC}"
}

interactive_mode() {
  if grep -qiE 'ID_LIKE=.*(debian|ubuntu)' /etc/os-release; then
  # Debian/Ubuntu
    echo -e "${BLUE}Starting interactive session with netcat${NC}"
    echo -e "Enter <prompt> to stay in interactive mode"
    sudo nc -U "$HAPROXY_SOCK"
  else
  # RHEL/Rocky/Alma
    echo -e "${RED}Starting interactive session with socat readline${NC}"
    socat READLINE,history=$HOME/.haproxy_history UNIX-CONNECT:"$HAPROXY_SOCK"
  fi
}

main_menu() {
  clear
  echo -e "${BLUE}===================== HAProxy Socat Swiss Knife =======================${NC}"
  echo -e " Socket: ${GREEN}$HAPROXY_SOCK${NC}"
  echo -e " Press 101 to run in interactive mode"
  echo  " 1. Show Runtime Info           2. Show Errors"
  echo  " 3. Active Sessions             4. Show Statistics"
  echo  " 5. Peers Status                6. Select a Stick Table"
  echo  " 7. Clear Stick Table           8. List Frontends"
  echo  " 9. Enable/Disable Frontend    10. Change Backend Status"
  echo  "11. Check Cookies              12. Watch Statistics"
  echo  "13. Trigger Health Check       14. Change Socket "
  echo  " 0. Exit"
  echo -e "${BLUE}=======================================================================${NC}"
  echo -e "${BLUE}==============github.com/Dyarven/haproxy-socat-swiss-knife=============${NC}"
  echo -e "${BLUE}=======================================================================${NC}"
}

show_menu() {
  check_reqs
  while true; do
    main_menu
    read -rp "Enter choice: " choice
    while read -t 0 -r -n 10000; do : ; done  # Clear input buffer
    case $choice in
      1) show_info ;;
      2) show_errors ;;
      3) show_sessions ;;
      4) show_threads ;;
      5) show_peers ;;
      6) show_stick_table ;;
      7) clear_stick_table ;;
      8) list_frontends ;;
      9) enable_disable_frontend ;;
      10) change_backend_server_state ;;
      11) check_cookies ;;
      12) watch_stats ;;
      13) trigger_health_check ;;
      14) change_socket ;;
      101) interactive_mode ;;
      0) exit 0 ;;
      *) echo -e "${RED}Invalid option${NC}" ;;
    esac
    read -rp "Press enter to continue..."
  done
}

trap 'echo -e "\n${GREEN}Bye bye...${NC}"; exit 0' SIGINT SIGTERM
show_menu
