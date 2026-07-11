1. Creating a systemd service file only defines the service; it will not start on server reboot unless you run `systemctl enable service-name`.
2. Django static files finally made sense to me when I understood that there are two common ways to organize them.

One way is to keep one main `static/` folder at the project level, usually beside `manage.py`. In this case, Django does not automatically know about that folder, so we tell it using `STATICFILES_DIRS`. This is similar to how we add a project-level `templates/` folder inside the `TEMPLATES["DIRS"]` list.

The second way is to keep static files inside each app, for example `myapp/static/myapp/...`, just like we often keep templates inside `myapp/templates/myapp/...`. In this app-level approach, Django can find those files automatically through its app directories system, so we usually do not need to add those app static folders manually in `STATICFILES_DIRS`.

Then there is `STATIC_ROOT`, which is a different thing. `STATIC_ROOT` is not where I normally write or edit static files during development. It is the final collection folder where Django puts all static files when I run `python manage.py collectstatic`.

So in development, static files may live in project-level `static/` folders and app-level `static/` folders. But when the project is ready for production and `DEBUG=False`, I run `collectstatic`, and Django copies all discovered static files into one final folder defined by `STATIC_ROOT`.

That final folder can technically have any name, but a common convention is something like:

STATIC_ROOT = os.path.join(BASE_DIR, "staticfiles")

The reason we usually do not set `STATIC_ROOT` to the same project-level `static/` folder is to avoid confusion or conflicts. The `static/` folder is usually where we keep source/static files while developing. The `staticfiles/` folder is usually the generated output folder created by `collectstatic`.

So the clean mental model is:

`STATICFILES_DIRS` = extra project-level static folders Django should search during development  
`app/static/` = app-level static folders Django can find automatically  
`STATIC_ROOT` = final production folder where `collectstatic` collects everything  
`STATIC_URL` = the URL prefix used in HTML to access static files, usually `/static/`

This means I should not mix up my working static files with the collected production static files. I write static files in app-level or project-level static folders, and let Django collect them into `STATIC_ROOT` only when needed for production.

3. In my production Django projects, I intentionally keep Django logging simple and send logs only through a `StreamHandler`. I do not attach a Django `FileHandler`, because I do not want every Django or Gunicorn worker handling file writes directly.

The logging flow I use is:

`Django logger -> StreamHandler -> stdout/stderr -> systemd -> log file -> EzLog`

The systemd service redirects `StandardOutput` and `StandardError` to the log file that `legeRise/ezlog` streams. This keeps responsibilities separated: Django creates log records, systemd writes them to the file, logrotate manages file size and rotation, and EzLog reads and displays the file.

Because Django is not writing the file itself, I also do not use Python's `RotatingFileHandler` or `TimedRotatingFileHandler`. Rotation must instead be handled externally, usually with `logrotate`. Adding a Django file handler as well would duplicate file I/O and could cause multiple Gunicorn workers to write to the same file unnecessarily.

So the mental model is:

`StreamHandler` = Django emits logs to the process output  
`systemd` = captures stdout/stderr and writes the log file  
`logrotate` = rotates and compresses the log file  
`EzLog` = streams and displays the log file

This setup is intentional, not naive. A Django file handler would only make sense if Python itself needed to own the log file and no external process manager was already doing that job.