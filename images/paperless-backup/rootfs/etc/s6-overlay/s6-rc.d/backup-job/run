#!/bin/bash

while true; do
    current_hour=$(date +%_H)
    current_minute=$(date +%M)

    if [[ "$current_hour" == "01" && "$current_minute" > "30" ]]; then
        document_exporter --use-filename-format --use-folder-prefix --no-thumbnail --delete /usr/src/backup-target
        sleep 30m
    else
        sleep 15m
    fi
done
