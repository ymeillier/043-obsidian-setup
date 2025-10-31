```bash

function md_cleanup() {
    # Check for required argument
    if [ -z "$1" ]; then
        echo "Usage: cleanup_obsidian_vault <path_to_repo>"
        return 1
    fi
    
    local vault_dir="$(realpath "$1")"
    local config_file="$HOME/Library/Application Support/obsidian/obsidian.json"
    
    # 1. Look up the Vault ID based on the directory path
    local vault_id_to_remove=$(jq -r --arg path "$vault_dir" \
        '.vaults | to_entries[] | select(.value.path == $path) | .key' \
        "$config_file" 2>/dev/null)

    # 2. File System Cleanup (Removes the marker)
    if [ -d "$vault_dir/.obsidian" ]; then
        rm -rf "$vault_dir/.obsidian"
        echo "✅ File system clean: Removed temporary .obsidian folder from $vault_dir"
    else
        echo "ℹ️ Note: .obsidian folder not found, skipping file system cleanup."
    fi
    
    # 3. JSON Configuration Cleanup (Removes the reference)
    if [[ -n "$vault_id_to_remove" ]]; then
        # Use jq to delete the key associated with the vault ID
        jq "del(.vaults[\"$vault_id_to_remove\"])" \
            "$config_file" > "$config_file.tmp" && mv "$config_file.tmp" "$config_file"
            
        echo "✅ Config clean: Removed vault ID ($vault_id_to_remove) from obsidian.json."
        
        # 4. Quit Obsidian (To force immediate config reload and cleanup of the open window)
        osascript -e 'quit app "Obsidian"' 2>/dev/null
        echo "ℹ️ Obsidian closed to finalize cleanup."
    else
        echo "ℹ️ Vault entry not found in obsidian.json. No config cleanup needed."
    fi
}



```