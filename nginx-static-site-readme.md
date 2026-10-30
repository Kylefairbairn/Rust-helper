# Nginx: Find Its Configuration and Serve a Website

This guide explains how a systemd-managed Nginx process finds its configuration, how to identify the files your installation uses, and how to serve static website files from a directory.

## 1. How Nginx finds its configuration

The Nginx executable and its configuration are separate things. The executable is often installed under a path such as \`/usr/sbin/nginx\`; that does not imply the configuration is under \`/usr/sbin\`.

When Nginx starts without a \`-c\` argument, it uses the default configuration path compiled into that executable. On many Linux distributions this is \`/etc/nginx/nginx.conf\`, but installations can differ.

The systemd unit can live outside \`/etc/systemd/system\`. Common locations are:

- \`/usr/lib/systemd/system/nginx.service\`
- \`/lib/systemd/system/nginx.service\`
- \`/etc/systemd/system/nginx.service\` (often a locally supplied unit or override)

The vendor unit starts Nginx. The main Nginx configuration usually lives separately, and that configuration may include more files.

## 2. Inspect the running service and configuration

Show the unit file and any systemd drop-ins:

\`\`\`sh
systemctl cat nginx
\`\`\`

Show the unit location and effective start command:

\`\`\`sh
systemctl show nginx -p FragmentPath -p ExecStart
\`\`\`

Look at Nginx's compiled paths and options:

\`\`\`sh
nginx -V 2>&1
\`\`\`

Find \`--conf-path\` and \`--prefix\` in the output. If the unit's \`ExecStart\` does not pass \`-c\`, the compiled default config path is used.

Print the full active configuration, including files pulled in by \`include\` directives:

\`\`\`sh
sudo nginx -T
\`\`\`

This output may include sensitive settings. Review it locally before sharing it. To focus on relevant directives:

\`\`\`sh
sudo nginx -T 2>&1 | less
\`\`\`

Check the executable path:

\`\`\`sh
command -v nginx
readlink -f "$(command -v nginx)"
\`\`\`

## 3. Understand the configuration layout

The main file is commonly \`/etc/nginx/nginx.conf\`. It may contain directives such as:

\`\`\`nginx
include /etc/nginx/conf.d/*.conf;
\`\`\`

or:

\`\`\`nginx
include /etc/nginx/sites-enabled/*;
\`\`\`

An included file can define a \`server { ... }\` block. Use \`nginx -T\` to see the actual include paths for this installation; do not assume both layouts are enabled.

A server block tells Nginx which address/port to listen on, which hostnames it handles, and where the website files live.

## 4. Serve a static website directory

For example, to serve files from \`/var/www/my-site\` over HTTP on port 80, create a directory and a simple page:

\`\`\`sh
sudo mkdir -p /var/www/my-site
printf '%s\\n' '<!doctype html><html><head><title>My site</title></head><body><h1>Nginx is serving this page</h1></body></html>' | sudo tee /var/www/my-site/index.html
\`\`\`

Create a server configuration in a directory that the main config includes. On systems using \`conf.d\`, for example:

\`\`\`sh
sudo editor /etc/nginx/conf.d/my-site.conf
\`\`\`

Put this in the file:

\`\`\`nginx
server {
    listen 80 default_server;
    server_name _;

    root /var/www/my-site;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
\`\`\`

The \`root\` directive maps a URL path to a filesystem path. For example, a request for \`/images/logo.png\` maps to \`/var/www/my-site/images/logo.png\`. The \`index\` directive names the file to serve for a directory URL. \`try_files\` serves a matching file or directory, and returns 404 if nothing matches.

If a default server block already listens on port 80, do not blindly add another \`default_server\` on the same address and port. Edit or disable the existing default block, or configure the intended \`server_name\`. Use \`sudo nginx -T\` to find existing listeners.

On systems using \`sites-available\` and \`sites-enabled\`, create the file in \`/etc/nginx/sites-available/\`, then enable it with a symlink if that is how the main config is set up:

\`\`\`sh
sudo ln -s /etc/nginx/sites-available/my-site /etc/nginx/sites-enabled/my-site
\`\`\`

Do not create the symlink if it already exists.

## 5. Check, reload, and verify

Validate configuration syntax and referenced files:

\`\`\`sh
sudo nginx -t
\`\`\`

If the test succeeds, reload Nginx so it reads the updated configuration:

\`\`\`sh
sudo systemctl reload nginx
\`\`\`

If Nginx is not running, start it with:

\`\`\`sh
sudo systemctl start nginx
\`\`\`

Check service status:

\`\`\`sh
systemctl status nginx --no-pager
\`\`\`

Check listening TCP ports and the owning process:

\`\`\`sh
sudo ss -ltnp
\`\`\`

Nginx's process arguments shown by \`ps aux\` generally do not list configured ports. The \`listen\` directive and socket listing show that information.

Test locally:

\`\`\`sh
curl -i http://127.0.0.1/
\`\`\`

If accessing from another computer, use the server's IP address or configured DNS name, and ensure the firewall permits the chosen port.

## 6. File access and common errors

Nginx's worker process must be able to read the site files and traverse every parent directory. Check the worker user in the main configuration (often a \`user\` directive) and inspect permissions:

\`\`\`sh
namei -l /var/www/my-site/index.html
\`\`\`

Avoid making the whole tree world-writable. Give the Nginx worker read access and directory traversal access using the ownership and permissions appropriate for your system.

Useful logs are commonly:

- \`/var/log/nginx/error.log\`
- \`/var/log/nginx/access.log\`

View recent service logs with:

\`\`\`sh
sudo journalctl -u nginx -n 100 --no-pager
\`\`\`

Common symptoms:

- **Connection refused:** Nginx may not be running or may not listen on the requested address/port.
- **Default page appears:** Another server block may be the default for that address and port, or your new file may not be included.
- **403 Forbidden:** Check directory traversal and file read permissions; also check for access rules in the configuration.
- **404 Not Found:** Confirm the URL maps to a file under the configured \`root\`, and check the error log.
- **Configuration test fails:** Read the reported file and line number. Common causes include a missing semicolon, unmatched brace, duplicate listener settings, or a file referenced by an \`include\` that does not exist.

## 7. Quick mental model

1. systemd starts the Nginx executable using the service's \`ExecStart\`.
2. Nginx uses the \`-c\` config path if one is supplied; otherwise it uses its compiled default.
3. The main config may include other config files.
4. A matching \`server\` block handles the request.
5. Directives such as \`listen\`, \`server_name\`, \`root\`, and \`index\` determine how the request is served.

