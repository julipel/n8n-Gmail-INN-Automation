# Cursor Cloud — n8n-Gmail-INN-Automation

## Cursor Cloud specific instructions

- **Docker required.** Committed `docker-compose.yml` uses Windows path `C:/n8n/files`; on Linux use a one-off run:

```bash
mkdir -p n8n-files
sudo docker run -d --name n8n_compose -p 5678:5678 \
  -e N8N_ENCRYPTION_KEY='change_me_32_chars_minimum_len' \
  -v n8n_data:/home/node/.n8n \
  -v "$(pwd)/n8n-files:/home/node/.n8n-files" \
  n8nio/n8n:latest
```

- **UI:** http://localhost:5678 — import `Gmail INN Automation.json`, configure Gmail/Sheets OAuth and placeholders from README.
- **No Python/Node deps** in this repo.
- Use `sudo docker` if not in the `docker` group.

## Multi-repo workspace

Python projects under `/agent/repos/` use `.venv` per repo; see their `AGENTS.md` files.
