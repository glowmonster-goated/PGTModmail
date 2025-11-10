# Deploying Modmail to Render

These steps cover deploying the Modmail bot as a Render Background Worker. Render's worker services are ideal for long-running Discord bots because they are restarted automatically when the process exits.

## 1. Fork or import the repository
1. Fork `modmail-dev/modmail` to your GitHub account (required so Render can read environment variables securely).
2. Push any custom changes (such as config defaults or plugins) to your fork before continuing.

## 2. Create a MongoDB database
Modmail requires an external MongoDB instance because Render's worker filesystem is ephemeral. You can use MongoDB Atlas or any compatible MongoDB provider.

Record the connection string; you will provide it as `CONNECTION_URI` later.

## 3. Provision a Render Background Worker
1. Log in to Render and click **New** → **Blueprint**.
2. Point Render at your fork and select the root directory (the repo already includes a `render.yaml` blueprint).
3. Render will show the **Modmail Bot** worker defined in the blueprint. Accept the defaults or adjust the plan/region for your use case.
4. Click **Create Resources**.

> **Alternative:** If you prefer to create the service manually, choose **New** → **Background Worker** instead of using the blueprint. Use `Docker` as the environment, the repo root as context, the `Dockerfile` provided in this repository, and `python bot.py` as the Start Command.

## 4. Configure environment variables
The bot reads configuration from environment variables. Set at least the following in Render's dashboard (Secrets Manager → Environment → Add Secret):

| Key | Description |
| --- | --- |
| `TOKEN` | Discord bot token from the [Developer Portal](https://discord.com/developers/applications). |
| `GUILD_ID` | Discord server ID the bot will operate in (right-click the server in Discord with Developer Mode enabled). |
| `OWNERS` | Comma-separated Discord user IDs that should have owner-only permissions. |
| `CONNECTION_URI` | MongoDB connection string created in step 2. |
| `LOG_URL` | URL of your Modmail log viewer. If you do not host one yet, you can temporarily set `https://logs.modmail.app` to keep the bot booting. |

Optional variables supported by Modmail can also be added:

- `DATABASE_TYPE` (defaults to `mongodb`).
- `GITHUB_TOKEN` (for automatic plugin updates).
- `REGISTRY_PLUGINS_ONLY` (set to `true` to limit plugins to the registry).

Render redeploys automatically when you edit secrets. Make sure there are no leading or trailing spaces in the values—discord tokens are especially sensitive.

## 5. Deploy and monitor
1. Trigger a deploy (Render does this automatically the first time).
2. Watch the **Logs** tab. Successful startup shows `Logged in as ...` followed by `Connected to database` messages. If you see `TOKEN must be set`, double-check the variable spelling and casing.
3. Render restarts the worker if it exits. Use **Manual Deploy** whenever you push new commits.

## 6. Keep Render awake (optional)
Background Workers already stay online as long as the process is running. You do **not** need an HTTP health check or uptime pings.

## 7. Update strategy
When upgrading Modmail:
1. Pull the latest changes into your fork.
2. Review the release notes for new required environment variables.
3. Push to the branch Render is tracking; it will redeploy automatically.

For plugin updates, commit the plugin files to your repository or configure `GITHUB_TOKEN` so the bot can self-update from plugin repositories.

---

If deployment still fails, download the deploy logs from Render and look for stack traces. Common causes include missing MongoDB permissions or typos in the token.

### MongoDB DNS errors on Render

If the logs show `ConfigurationError: The DNS query name does not exist`, Render cannot resolve the hostname inside your `CONNECTION_URI`. Copy the Atlas connection string again and ensure the host still looks like `cluster0.xxxxx.mongodb.net`. Do not replace the host with your database name (for example changing it to `pgtmodmail.xxxxx.mongodb.net`) or any other custom word—Atlas SRV records rely on the original host name. After updating the secret Render will redeploy automatically.
