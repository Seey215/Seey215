<p align="center">
  <a href="https://squidfunk.github.io/mkdocs-material/">
    <img src="../assets/logo.svg" width="320" alt="Material for MkDocs">
  </a>
</p>

# Material for MkDocs

Made with [Material for MkDocs](https://www.mkdocs.org).

### Commands

> * `mkdocs new [dir-name]` - Create a new project. eg: `mkdocs new .`
> * `mkdocs serve` - Start the live-reloading docs server.
> * `mkdocs build` - Build the documentation site.
> * `mkdocs -h` - Print help message and exit.
> * `mkdocs gh-deploy` - Deploy to GitHub Pages.

### Action Order

```shell
# 检查Nginx配置是否正确
nginx -t
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful
# 主页路径：/usr/share/nginx/html
# 英语路径：/var/www/docs.seey215.cn/site
# Nginx地址：/etc/nginx

# 重启Nginx
sudo systemctl restart nginx
```