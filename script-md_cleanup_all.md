
```bash
function md_cleanup_all() {
    local config_file="$HOME/Library/Application Support/obsidian/obsidian.json"
    local TEMP_ID_PREFIX="99999999999999"

    if ! command -v jq &> /dev/null; then
        echo "Error: 'jq' is required for JSON manipulation. Aborting."
        return 1
    fi

    echo "--- Initiating Cleanup of Ephemeral Vaults ($TEMP_ID_PREFIX*) ---"
    
    # Use jq to select only the vaults whose keys match the temporary prefix
    local temp_vault_data=$(jq -r --arg prefix "$TEMP_ID_PREFIX" \
        '.vaults | to_entries[] | select(.key | startswith($prefix)) | "\(.value.path),\(.key)"' \
        "$config_file" 2>/dev/null)

    if [ -z "$temp_vault_data" ]; then
        echo "✅ No ephemeral vaults found requiring cleanup."
        return 0
    fi

    echo "$temp_vault_data" | while IFS=, read -r vault_dir vault_id_to_remove; do
        
        echo "Processing Vault: $(basename "$vault_dir") (ID: $vault_id_to_remove)"
        
        # 1. File System Cleanup (Removes the marker)
        if [ -d "$vault_dir/.obsidian" ]; then
            rm -rf "$vault_dir/.obsidian"
            echo "  - ✅ Removed temporary .obsidian folder."
        else
            echo "  - ℹ️ .obsidian folder not found (already cleaned)."
        fi
        
        # 2. JSON Configuration Cleanup (Deletes the entry by ID)
        jq "del(.vaults[\"$vault_id_to_remove\"])" \
            "$config_file" > "$config_file.tmp" && mv "$config_file.tmp" "$config_file"
            
        echo "  - ✅ Removed ID from obsidian.json."
    done

    # 3. Quit Obsidian (To force immediate config reload)
    osascript -e 'quit app "Obsidian"' 2>/dev/null
    echo ""
    echo "✨ Complete. All identified ephemeral vaults have been cleaned up and Obsidian closed."
}

```