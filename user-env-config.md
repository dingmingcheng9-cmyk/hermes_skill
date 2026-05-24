# User Environment Config

## NAS
- IP: 192.168.31.187, user: nasuser, pwd: in Windows Credential Manager (cmdkey)
- Shares: hermes-ext2 (Y:), hermes-ext (Z:)
- Startup: %APPDATA%\...\Startup\map_nas.bat (delete-then-remap to fix SMB reconnect)

## Backup target
- Z:\windows-hermes-日志\

## WSL2
- VHDX: D:\wsl\ubuntu\ext4.vhdx, user: dingm, sudo pwd: dingm
- pip → pipx (PEP 668)

## Obsidian
- Vault: D:\知识库\knowledge-warehouse
- Remote: dingmingcheng9-cmyk/knowledge-warehouse
- obsidian-git v2.38.2: commit 10min, push/pull 30min
