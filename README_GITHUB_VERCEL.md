# WoTTS — World Top 2% Scientists

GitHub/Vercel preparation package based on V23 Preview.

## Repository policy
The application source belongs in GitHub. The large SQLite datasets are intentionally excluded by `.gitignore`.
GitHub blocks ordinary Git objects larger than 100 MiB, and GitHub recommends external storage for generated/large database files.

## Current local preview
For the complete local preview, keep the seven `top_scientists_YYYY.db` files next to `server.py` and run:

    python3 server.py

Then open the local address printed by the server.

## Production architecture
- GitHub: source/version control.
- Vercel: public frontend and server-side endpoints.
- External persistent data layer: scientific datasets.
- Contact endpoint: server-side mail provider, recipient stored as an environment variable.
- Analytics: private; no public Visits control.
- Daily report: scheduled server-side job after analytics/mail credentials are configured.

## Important
This package does not contain secret keys and does not pretend that Contact/email or private analytics are active before provider credentials are configured.
