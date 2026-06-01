# Cursor Automations for English Coach

Import these at [cursor.com/automations](https://cursor.com/automations).  
Repo: `forcexun1-oss/english-learning` · branch: `main`

## Daily article

- **File:** `daily-article.json`
- **Schedule:** `30 9 * * *` UTC (17:30 Asia/Shanghai)
- **Output:** `articles/YYYY-MM-DD.md`

## Weekly summary

- **File:** `weekly-summary.json`
- **Schedule:** `0 10 * * 5` UTC (18:00 Friday Asia/Shanghai)
- **Output:** `articles/weekly-YYYY-MM-DD.md` + `progress.md`

## Daily quiz

- **File:** `daily-quiz.json`
- **Schedule:** `0 15 * * *` UTC (23:00 Asia/Shanghai)
- **Output:** `quiz/YYYY-MM-DD.json`
- **Questions:** 6 multiple-choice + 4 free-text, drawn from that day's log
- **Idempotent:** skips if output file already exists or log is empty

## Notes

- Pick any Cursor model in the automation settings.
- Do **not** enable “Open pull request” — prompt commits directly to `main`.
- Ensure GitHub repo access is connected in Cursor Cloud Agents settings.
