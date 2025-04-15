# Bash



<pre>
#!/bin/bash

# ==============================================
# CONFIGURATION
# ==============================================
LOG_DIR="/var/log/nginx"
ACCESS_LOG="${LOG_DIR}/access.log"
ERROR_LOG="${LOG_DIR}/error.log"
TMP_DIR="/tmp/log_analyzer"
LOCKFILE="${TMP_DIR}/lock"
LAST_RUN_FILE="${TMP_DIR}/lastrun"
MAIL_TO="it@kvadrat-c.ru"
SUBJECT="Hourly Web Server Report ($(hostname))"
MAX_ENTRIES=10  # Количество записей в топах

# ==============================================
# INITIALIZATION
# ==============================================
mkdir -p "${TMP_DIR}"
trap "cleanup" EXIT INT TERM HUP

# ==============================================
# FUNCTIONS
# ==============================================
cleanup() {
    rm -f "${LOCKFILE}"
    exit
}

die() {
    echo "$1" | mail -s "Script Error: ${SUBJECT}" "${MAIL_TO}"
    cleanup
}

get_timestamp() {
    date -d "$1" +"%Y-%m-%d %H:%M:%S" 2>/dev/null || date +"%Y-%m-%d %H:%M:%S"
}

check_logs() {
    [ -r "${ACCESS_LOG}" ] || die "Cannot read access log: ${ACCESS_LOG}"
    [ -r "${ERROR_LOG}" ] || die "Cannot read error log: ${ERROR_LOG}"
}

acquire_lock() {
    if [ -e "${LOCKFILE}" ] && kill -0 $(cat "${LOCKFILE}") 2>/dev/null; then
        die "Script is already running (PID: $(cat "${LOCKFILE}"))"
    fi
    echo $$ > "${LOCKFILE}" || die "Failed to create lock file"
}

get_last_run() {
    if [ -f "${LAST_RUN_FILE}" ]; then
        cat "${LAST_RUN_FILE}"
    else
        date -d "1 hour ago" +"%Y-%m-%d %H:%M:%S"
    fi
}

process_logs() {
    local start_date="$1"
    local end_date="$2"
    
    # Обработка access.log и архивов
    find "${LOG_DIR}" -name "access.log*" -type f -newermt "${start_date}" ! -newermt "${end_date}" | sort | while read logfile; do
        if [[ "${logfile}" == *.gz ]]; then
            zcat "${logfile}"
        else
            cat "${logfile}"
        fi
    done
}

analyze_data() {
    local log_data="$1"
    
    # 1. Топ IP
    local top_ips=$(echo "${log_data}" | awk '{print $1}' | sort | uniq -c | sort -nr | head -n ${MAX_ENTRIES} | sed 's/^[ \t]*//')
    
    # 2. Топ URL (очищаем от параметров)
    local top_urls=$(echo "${log_data}" | awk '{print $7}' | sed 's/\?.*$//' | sort | uniq -c | sort -nr | head -n ${MAX_ENTRIES} | sed 's/^[ \t]*//')
    
    # 3. HTTP коды
    local http_codes=$(echo "${log_data}" | awk '$9 ~ /^[0-9]{3}$/ {print $9}' | sort | uniq -c | sort -nr | sed 's/^[ \t]*//')
    
    # 4. Ошибки из error.log
    local errors=$(find "${LOG_DIR}" -name "error.log*" -type f -newermt "${start_date}" ! -newermt "${end_date}" | sort | while read logfile; do
        if [[ "${logfile}" == *.gz ]]; then
            zcat "${logfile}"
        else
            cat "${logfile}"
        fi
    done | grep -E -i 'error|warn|fail|critical' | tail -n ${MAX_ENTRIES})
    
    echo "${top_ips}" "${top_urls}" "${http_codes}" "${errors}"
}

format_report() {
    local start_date="$1"
    local end_date="$2"
    local analysis="$3"
    
    cat <<EOF
Отчет за период с ${start_date} по ${end_date}

=== Топ ${MAX_ENTRIES} IP-адресов ===
$(echo "${analysis}" | awk 'NR==1,NR=='${MAX_ENTRIES}'')

=== Топ ${MAX_ENTRIES} URL ===
$(echo "${analysis}" | awk 'NR=='$((MAX_ENTRIES+1))',NR=='$((MAX_ENTRIES*2))'')

=== Коды HTTP ответа ===
$(echo "${analysis}" | awk 'NR=='$((MAX_ENTRIES*2+1))',NR=='$((MAX_ENTRIES*3))'')

=== Ошибки сервера ===
$(echo "${analysis}" | awk 'NR>'$((MAX_ENTRIES*3))'')
EOF
}

send_report() {
    local report="$1"
    echo "${report}" | mail -s "${SUBJECT}" "${MAIL_TO}"
}

# ==============================================
# MAIN SCRIPT
# ==============================================
acquire_lock
check_logs

START_TIME=$(get_last_run)
END_TIME=$(date +"%Y-%m-%d %H:%M:%S")

# Получаем данные
LOG_DATA=$(process_logs "${START_TIME}" "${END_TIME}")

# Анализируем
ANALYSIS_RESULTS=$(analyze_data "${LOG_DATA}")

# Формируем отчет
REPORT=$(format_report "${START_TIME}" "${END_TIME}" "${ANALYSIS_RESULTS}")

# Отправляем
send_report "${REPORT}"

# Сохраняем время последнего запуска
echo "${END_TIME}" > "${LAST_RUN_FILE}"

cleanup
</pre>

Сохраняем скрипт в файл /usr/local/bin/log_analyzer.sh

Делаем его исполняемым: chmod +x /usr/local/bin/log_analyzer.sh

Добавляем в к crontab 

<pre> crontab -e </pre>

0 * * * * /bin/bash /usr/local/bin/log_analyzer.sh

все

